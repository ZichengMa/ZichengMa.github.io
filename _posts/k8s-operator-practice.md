---
title: k8s operator practice
typora-root-url: k8s operator practice
date: 2023-03-26 20:28:15
description: Introduction about k8s operator and a sample code of a very simple controller.
tags:
---

I am learning how to write Kubernetes operator these days. This post is used to record some basic definition and a very good sample controller implemented by [RedHat](http://workshop.coreostrain.me/lab/go-operator/podset/).

### Introduction to Kubernetes Operators

Kubernetes is a powerful platform for managing containerized applications at scale. While Kubernetes provides a robust set of primitives for deploying and managing applications, it can be challenging to build and manage complex applications using only these primitives. This is where Kubernetes Operators come in.

An Operator is a method of packaging, deploying, and managing a Kubernetes application. Operators use custom resources, controllers, and reconciliation loops to automate the deployment and management of complex applications on Kubernetes. In this article, we will explore the concept of Operators in Kubernetes and dive into the details of building an Operator using Go.

### Reconciler Overview

One of the key components of a Kubernetes Operator is the reconciler. The reconciler is responsible for comparing the desired state of a Kubernetes resource with its current state and taking actions to reconcile any differences between the two.

In Go-based Kubernetes Operators, the reconciler is typically implemented as a controller. The controller watches for changes to a specific type of Kubernetes resource, and when a change is detected, it initiates a reconciliation loop. During the reconciliation loop, the controller retrieves the current state of the resource from the Kubernetes API server, compares it with the desired state, and takes actions to reconcile any differences.

### Code analysis
The CRDs we defined is PodSet. 
	Spec: Replicas(How many pods we want) 
	Status: ReadyReplicas(The number of ready replicas)   PodNames(A slice for all ready replicas name)
Controller will try to update Status.ReadyReplicas = Spec.Replicas and up/down scaling pods based on the difference.
```go
/*
Copyright 2023.

Licensed under the Apache License, Version 2.0 (the "License");
you may not use this file except in compliance with the License.
You may obtain a copy of the License at

    http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software
distributed under the License is distributed on an "AS IS" BASIS,
WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
See the License for the specific language governing permissions and
limitations under the License.
*/

package controllers

import (
	"context"
	"reflect"

	corev1 "k8s.io/api/core/v1"
	"k8s.io/apimachinery/pkg/api/errors"
	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/labels"
	"k8s.io/apimachinery/pkg/runtime"
	ctrl "sigs.k8s.io/controller-runtime"
	"sigs.k8s.io/controller-runtime/pkg/client"
	"sigs.k8s.io/controller-runtime/pkg/controller/controllerutil"
	"sigs.k8s.io/controller-runtime/pkg/log"

	batchv1 "tutorial.kubebuilder.io/operator-practice/api/v1"
)

// PodSetReconciler reconciles a PodSet object
type PodSetReconciler struct {
	client.Client
	Scheme *runtime.Scheme
}

//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=podsets,verbs=get;list;watch;create;update;patch;delete
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=podsets/status,verbs=get;update;patch
//+kubebuilder:rbac:groups=batch.tutorial.kubebuilder.io,resources=podsets/finalizers,verbs=update

// Reconcile is part of the main kubernetes reconciliation loop which aims to
// move the current state of the cluster closer to the desired state.
// TODO(user): Modify the Reconcile function to compare the state specified by
// the PodSet object against the actual cluster state, and then
// perform operations to make the cluster state reflect the state specified by
// the user.
//
// For more details, check Reconcile and its Result here:
// - https://pkg.go.dev/sigs.k8s.io/controller-runtime@v0.14.1/pkg/reconcile
func (r *PodSetReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
	log := log.FromContext(ctx)

	// TODO(user): your logic here

	// listen to the creation of PodSet objects

	// fetch the PodSet object
	podset := &batchv1.PodSet{}
	err := r.Get(context.TODO(), req.NamespacedName, podset) // get the PodSet object from the API server by its name and namespace and store it in the podset variable
	if err != nil {
		if errors.IsNotFound(err) {
			// Request object not found, could have been deleted after reconcile request.
			// Owned objects are automatically garbage collected. For additional cleanup logic use finalizers.
			// Return and don't requeue
			return ctrl.Result{}, nil
		}
		// Error reading the object - requeue the request.
		// If the error is not a NotFound error, the second if statement is triggered, and the controller
		// returns a ctrl.Result{} along with the original error to requeue the request for reconciliation.
		// This is done to allow the controller to retry the operation at a later time, in case the error was temporary or transient.
		return ctrl.Result{}, err
	}

	podList := &corev1.PodList{} // create a new list of pods not podsets!
	// The lbs variable is a map of labels that will be used to filter the list of pods. In this case, the labels are set to app=podset.Name and version=v0.1.
	// This means that the query will only return pods that have the app label set to the name of the podset object and the version label set to v0.1.
	lbs := map[string]string{
		"app":     podset.Name,
		"version": "v0.1",
	}
	labelSelector := labels.SelectorFromSet(lbs) // create a label selector from the lbs map
	// the listOps variable is a client.ListOptions object that contains the options for querying the Kubernetes API server.
	listOps := &client.ListOptions{Namespace: podset.Namespace, LabelSelector: labelSelector} // create a list options object with the namespace and label selector
	if err = r.List(context.TODO(), podList, listOps); err != nil {
		return ctrl.Result{}, err
	}

	// Count the pods that are pending or running as available
	var available []corev1.Pod
	for _, pod := range podList.Items {
		// For each pod, it first checks whether the DeletionTimestamp field is set.
		// If it is, this means that the pod is scheduled for deletion, and the code skips it
		if pod.ObjectMeta.DeletionTimestamp != nil {
			continue
		}
		// If the pod is not scheduled for deletion, then check whether the pod's Status.Phase field is set to either corev1.PodRunning or corev1.PodPending.
		// If the pod is in one of these states, it is considered available, and its reference is added to the available slice.
		if pod.Status.Phase == corev1.PodRunning || pod.Status.Phase == corev1.PodPending {
			available = append(available, pod)
		}
	}
	numAvailable := int32(len(available)) // get the number of available pods

	// This code block creates a new slice of strings called availableNames,
	// which contains the names of all the available pods returned by the previous code block.
	availableNames := []string{}
	for _, pod := range available {
		availableNames = append(availableNames, pod.ObjectMeta.Name)
	}

	// Update the status if necessary
	status := batchv1.PodSetStatus{
		PodNames:      availableNames,
		ReadyReplicas: numAvailable,
	}
	// The code first checks whether the podset.Status field is equal to the new status value using the reflect.DeepEqual() function.
	// If the two values are not equal, it means that the status value has been updated and needs to be written back to the Kubernetes API server.
	if !reflect.DeepEqual(podset.Status, status) {
		podset.Status = status
		err = r.Status().Update(context.TODO(), podset)
		if err != nil {
			log.Error(err, "Failed to update PodSet status")
			return ctrl.Result{}, err
		}
	}

	if numAvailable == podset.Spec.Replicas {
		// If the number of available pods is equal to the number of replicas specified in the podset.Spec.Replicas field,
		// then the controller returns a ctrl.Result{} object without an error to indicate that the reconciliation is complete.
		return ctrl.Result{}, nil
	}

	// Scale up or down
	if numAvailable > podset.Spec.Replicas {
		log.Info("Scaling down pods", "Currently available", numAvailable, "Required replicas", podset.Spec.Replicas)
		diff := numAvailable - podset.Spec.Replicas
		dpods := available[:diff]
		for _, dpod := range dpods {
			err = r.Delete(context.TODO(), &dpod) // Writer interface --> Create Delete Update ...
			if err != nil {
				log.Error(err, "Failed to delete pod", "pod.name", dpod.Name)
				return ctrl.Result{}, err
			}
		}
		return ctrl.Result{Requeue: true}, nil
	}

	if numAvailable < podset.Spec.Replicas {
		log.Info("Scaling up pods", "Currently available", numAvailable, "Required replicas", podset.Spec.Replicas)
		// Define a new Pod object
		pod := newPodForCR(podset)
		// Set PodSet instance as the owner and controller
		// This ensures that the new pod is "owned" by the PodSet object and is managed by the controller.
		// When a child object is created, it is important to set a reference to its owner object using the SetControllerReference function.
		// This ensures that the owner object is set as the "controller" of the child object,
		// which allows Kubernetes to automatically manage the child object's lifecycle and ensures that it is deleted when the owner object is deleted.
		if err := controllerutil.SetControllerReference(podset, pod, r.Scheme); err != nil {
			return ctrl.Result{}, err
		}
		err = r.Create(context.TODO(), pod)
		if err != nil {
			log.Error(err, "Failed to create pod", "pod.name", pod.Name)
			return ctrl.Result{}, err
		}
		return ctrl.Result{Requeue: true}, nil
	}
	return ctrl.Result{}, nil
}

// TODO: Add a function called newPodForCR() that creates a new pod object for the podset object passed as an argument.
// args: podset *batchv1.PodSet
// newPodForCR returns a busybox pod with the same name/namespace as the cr
func newPodForCR(cr *batchv1.PodSet) *corev1.Pod {
	// creates a map of labels called labels, which contains two key-value pairs:
	// app set to the name of the PodSet object, and version set to "v0.1".
	// These labels are used to identify the pods created by the PodSet object.
	labels := map[string]string{
		"app":     cr.Name,
		"version": "v0.1",
	}
	return &corev1.Pod{
		ObjectMeta: metav1.ObjectMeta{
			GenerateName: cr.Name + "-pod-", // GenerateName is used to generate a unique name for the pod.
			Namespace:    cr.Namespace,
			Labels:       labels,
		},
		Spec: corev1.PodSpec{
			Containers: []corev1.Container{
				{
					Name:    "busybox",
					Image:   "busybox",
					Command: []string{"sleep", "3600"},
				},
			},
		},
	}
}

// SetupWithManager sets up the controller with the Manager.
func (r *PodSetReconciler) SetupWithManager(mgr ctrl.Manager) error {
	return ctrl.NewControllerManagedBy(mgr).
		For(&batchv1.PodSet{}).
		Complete(r)
}

```

