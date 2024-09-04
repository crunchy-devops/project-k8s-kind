package main

import (
	"context"
	"flag"
	"fmt"
	"log"
	"os"
	"os/signal"
	"syscall"
	"time"

	"k8s.io/apimachinery/pkg/api/errors"
	"k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/runtime"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/util/workqueue"
	"k8s.io/client-go/util/wait"
	"k8s.io/client-go/util/homedir"
	"path/filepath"

	"k8s.io/client-go/tools/clientcmd"
	"k8s.io/client-go/util/retry"
)

var (
	fooGVR = schema.GroupVersionResource{
		Group:    "samplecontroller.k8s.io",
		Version:  "v1",
		Resource: "foos",
	}
)

func main() {
	var kubeconfig string
	if home := homedir.HomeDir(); home != "" {
		kubeconfig = filepath.Join(home, ".kube", "config")
	} else {
		kubeconfig = ""
	}

	kubeconfig = *flag.String("kubeconfig", kubeconfig, "(optional) absolute path to the kubeconfig file")
	flag.Parse()

	config, err := clientcmd.BuildConfigFromFlags("", kubeconfig)
	if err != nil {
		log.Fatalf("Error building kubeconfig: %s", err.Error())
	}

	dynClient, err := dynamic.NewForConfig(config)
	if err != nil {
		log.Fatalf("Error creating dynamic client: %s", err.Error())
	}

	queue := workqueue.NewRateLimitingQueue(workqueue.DefaultControllerRateLimiter())

	informer := cache.NewSharedInformer(
		cache.NewFilteredListWatchFromClient(
			dynClient.Resource(fooGVR).Namespace("default"),
			"foos",
			v1.NamespaceDefault,
			func(options *v1.ListOptions) {
				options.LabelSelector = ""
			},
		),
		&runtime.Unknown{},
		0,
	)

	informer.AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
		UpdateFunc: func(oldObj, newObj interface{}) {
			key, err := cache.MetaNamespaceKeyFunc(newObj)
			if err == nil {
				queue.Add(key)
			}
		},
		DeleteFunc: func(obj interface{}) {
			key, err := cache.DeletionHandlingMetaNamespaceKeyFunc(obj)
			if err == nil {
				queue.Add(key)
			}
		},
	})

	stopCh := make(chan struct{})
	defer close(stopCh)

	go informer.Run(stopCh)

	// Wait for cache to sync
	if !cache.WaitForCacheSync(stopCh, informer.HasSynced) {
		log.Fatalf("Error syncing cache")
	}

	go wait.Until(func() {
		for {
			obj, shutdown := queue.Get()
			if shutdown {
				return
			}

			func(obj interface{}) {
				defer queue.Done(obj)

				key := obj.(string)
				namespace, name, err := cache.SplitMetaNamespaceKey(key)
				if err != nil {
					log.Printf("Error splitting key: %s", err.Error())
					return
				}

				// Get the Foo object
				err = retry.OnError(retry.DefaultRetry, errors.IsNotFound, func() error {
					foo, err := dynClient.Resource(fooGVR).Namespace(namespace).Get(context.TODO(), name, v1.GetOptions{})
					if err != nil {
						return err
					}

					// Extract and log the `bar` field from the Foo spec
					bar, found, err := unstructured.NestedString(foo.Object, "spec", "bar")
					if err != nil || !found {
						log.Printf("Error retrieving bar field or field not found: %s", err.Error())
						return nil
					}

					log.Printf("Processing Foo: %s/%s - Bar: %s", namespace, name, bar)

					return nil
				})

				if err != nil {
					log.Printf("Error processing key: %s", err.Error())
				}
			}(obj)
		}
	}, time.Second, stopCh)

	// Handle graceful shutdown
	sigCh := make(chan os.Signal, 1)
	signal.Notify(sigCh, syscall.SIGINT, syscall.SIGTERM)
	<-sigCh
	close(stopCh)
	queue.ShutDown()
}

func unstructuredNestedString(obj map[string]interface{}, fields ...string) (string, bool, error) {
	if m, found, err := unstructuredNestedFieldNoCopy(obj, fields...); found && err == nil {
		if str, ok := m.(string); ok {
			return str, true, nil
		}
		return "", false, fmt.Errorf("field %s is not a string", fields)
	} else {
		return "", found, err
	}
}

func unstructuredNestedFieldNoCopy(obj map[string]interface{}, fields ...string) (interface{}, bool, error) {
	m := obj
	for i, field := range fields {
		if v, ok := m[field]; ok {
			if i == len(fields)-1 {
				return v, true, nil
			}
			switch t := v.(type) {
			case map[string]interface{}:
				m = t
			default:
				return nil, false, fmt.Errorf("unexpected type %T for field %s", v, field)
			}
		} else {
			return nil, false, nil
		}
	}
	return nil, false, nil
}
