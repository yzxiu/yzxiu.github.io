# client-go解析(6) - Factory


## 概述

`sharedIndexInformer` 一般不单独使用，通常都是使用 `factory` 来管理 sharedIndexInformer。

本文主要了解几种常见的 `factory` 以及它们如何构造和持有 sharedIndexInformer



## SharedInformerFactory

```go
type sharedInformerFactory struct {
   client           kubernetes.Interface
   namespace        string
   tweakListOptions internalinterfaces.TweakListOptionsFunc
   lock             sync.Mutex
   defaultResync    time.Duration
   customResync     map[reflect.Type]time.Duration

   informers map[reflect.Type]cache.SharedIndexInformer
   // startedInformers is used for tracking which informers have been started.
   // This allows Start() to be called multiple times safely.
   startedInformers map[reflect.Type]bool
   // wg tracks how many goroutines were started.
   wg sync.WaitGroup
   // shuttingDown is true when Shutdown has been called. It may still be running
   // because it needs to wait for goroutines.
   shuttingDown bool
}
```

缓存informer的结构为：

`informers map[reflect.Type]cache.SharedIndexInformer`

<br>

最常见的 factory 是 `SharedInformerFactory`，我们熟悉的 `kube-controller-manager`，主要就是由 SharedInformerFactory 构成。

main()  -> NewControllerManagerCommand() -> Run()

```go
// Run runs the KubeControllerManagerOptions.
func Run(c *config.CompletedConfig, stopCh <-chan struct{}) error {
	...
	...
	run := func(ctx context.Context, startSATokenController InitFunc, initializersFunc ControllerInitializersFunc) {
		controllerContext, err := CreateControllerContext(c, rootClientBuilder, clientBuilder, ctx.Done())
		...
		...
		controllerInitializers := initializersFunc(controllerContext.LoopMode)
		if err := StartControllers(ctx, controllerContext, startSATokenController, controllerInitializers, unsecuredMux, healthzHandler); err != nil {
			klog.Fatalf("error starting controllers: %v", err)
		}

		controllerContext.InformerFactory.Start(stopCh)
		controllerContext.ObjectOrMetadataInformerFactory.Start(stopCh)
		close(controllerContext.InformersStarted)

		<-ctx.Done()
	}

	// No leader election, run directly
	if !c.ComponentConfig.Generic.LeaderElection.LeaderElect {
		ctx, _ := wait.ContextForChannel(stopCh)
		run(ctx, saTokenControllerInitFunc, NewControllerInitializers)
		return nil
	}
	...
	...
	<-stopCh
	return nil
}
```

```go
// CreateControllerContext creates a context struct containing references to resources needed by the
// controllers such as the cloud provider and clientBuilder. rootClientBuilder is only used for
// the shared-informers client and token controller.
func CreateControllerContext(s *config.CompletedConfig, rootClientBuilder, clientBuilder clientbuilder.ControllerClientBuilder, stop <-chan struct{}) (ControllerContext, error) {
   versionedClient := rootClientBuilder.ClientOrDie("shared-informers")
   sharedInformers := informers.NewSharedInformerFactory(versionedClient, ResyncPeriod(s)())

   metadataClient := metadata.NewForConfigOrDie(rootClientBuilder.ConfigOrDie("metadata-informers"))
   metadataInformers := metadatainformer.NewSharedInformerFactory(metadataClient, ResyncPeriod(s)())
   ...
   ...
   ctx := ControllerContext{
      ClientBuilder:                   clientBuilder,
      InformerFactory:                 sharedInformers,
      ObjectOrMetadataInformerFactory: informerfactory.NewInformerFactory(sharedInformers, metadataInformers),
      ComponentConfig:                 s.ComponentConfig,
      RESTMapper:                      restMapper,
      AvailableResources:              availableResources,
      Cloud:                           cloud,
      LoopMode:                        loopMode,
      InformersStarted:                make(chan struct{}),
      ResyncPeriod:                    ResyncPeriod(s),
      ControllerManagerMetrics:        controllersmetrics.NewControllerManagerMetrics("kube-controller-manager"),
   }
   controllersmetrics.Register()
   return ctx, nil
}
```

```go
// NewControllerInitializers is a public map of named controller groups (you can start more than one in an init func)
// paired to their InitFunc.  This allows for structured downstream composition and subdivision.
func NewControllerInitializers(loopMode ControllerLoopMode) map[string]InitFunc {
	controllers := map[string]InitFunc{}

	// All of the controllers must have unique names, or else we will explode.
	register := func(name string, fn InitFunc) {
		if _, found := controllers[name]; found {
			panic(fmt.Sprintf("controller name %q was registered twice", name))
		}
		controllers[name] = fn
	}

	register("endpoint", startEndpointController)
	register("endpointslice", startEndpointSliceController)
	register("endpointslicemirroring", startEndpointSliceMirroringController)
	register("replicationcontroller", startReplicationController)
	register("podgc", startPodGCController)
	register("resourcequota", startResourceQuotaController)
	register("namespace", startNamespaceController)
	register("serviceaccount", startServiceAccountController)
	register("garbagecollector", startGarbageCollectorController)
	register("daemonset", startDaemonSetController)
	register("job", startJobController)
	register("deployment", startDeploymentController)
	register("replicaset", startReplicaSetController)
	register("horizontalpodautoscaling", startHPAController)
	register("disruption", startDisruptionController)
	register("statefulset", startStatefulSetController)
	register("cronjob", startCronJobController)
	register("csrsigning", startCSRSigningController)
	register("csrapproving", startCSRApprovingController)
	register("csrcleaner", startCSRCleanerController)
	register("ttl", startTTLController)
	register("bootstrapsigner", startBootstrapSignerController)
	register("tokencleaner", startTokenCleanerController)
	register("nodeipam", startNodeIpamController)
	register("nodelifecycle", startNodeLifecycleController)
	if loopMode == IncludeCloudLoops {
		register("service", startServiceController)
		register("route", startRouteController)
		register("cloud-node-lifecycle", startCloudNodeLifecycleController)
		// TODO: volume controller into the IncludeCloudLoops only set.
	}
	register("persistentvolume-binder", startPersistentVolumeBinderController)
	register("attachdetach", startAttachDetachController)
	register("persistentvolume-expander", startVolumeExpandController)
	register("clusterrole-aggregation", startClusterRoleAggregrationController)
	register("pvc-protection", startPVCProtectionController)
	register("pv-protection", startPVProtectionController)
	register("ttl-after-finished", startTTLAfterFinishedController)
	register("root-ca-cert-publisher", startRootCACertPublisher)
	register("ephemeral-volume", startEphemeralVolumeController)
	if utilfeature.DefaultFeatureGate.Enabled(genericfeatures.APIServerIdentity) &&
		utilfeature.DefaultFeatureGate.Enabled(genericfeatures.StorageVersionAPI) {
		register("storage-version-gc", startStorageVersionGCController)
	}
	if utilfeature.DefaultFeatureGate.Enabled(kubefeatures.DynamicResourceAllocation) {
		controllers["resource-claim-controller"] = startResourceClaimController
	}

	return controllers
}
```

以 DeploymentInformer 为例，最终创建 SharedIndexInformer 的函数如下:

```go
// pkg/controller/deployment/deployment_controller.go
dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
	AddFunc:    dc.addDeployment,
	UpdateFunc: dc.updateDeployment,
	// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
	DeleteFunc: dc.deleteDeployment,
})
```

```go
// k8s.io/client-go/informers/apps/v1/deployment.go
func (f *deploymentInformer) Informer() cache.SharedIndexInformer {
	return f.factory.InformerFor(&appsv1.Deployment{}, f.defaultInformer)
}
```

```go
// k8s.io/client-go/informers/factory.go
// InternalInformerFor returns the SharedIndexInformer for obj using an internal
// client.
func (f *sharedInformerFactory) InformerFor(obj runtime.Object, newFunc internalinterfaces.NewInformerFunc) cache.SharedIndexInformer {
	f.lock.Lock()
	defer f.lock.Unlock()

	informerType := reflect.TypeOf(obj)
	informer, exists := f.informers[informerType]
	if exists {
		return informer
	}

	resyncPeriod, exists := f.customResync[informerType]
	if !exists {
		resyncPeriod = f.defaultResync
	}
	// newFunc 最终调用 NewFilteredDeploymentInformer
	informer = newFunc(f.client, resyncPeriod)
	f.informers[informerType] = informer

	return informer
}
```

```go
// k8s.io/client-go/informers/apps/v1/deployment.go
func NewFilteredDeploymentInformer(client kubernetes.Interface, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions internalinterfaces.TweakListOptionsFunc) cache.SharedIndexInformer {
   return cache.NewSharedIndexInformer(
      &cache.ListWatch{
         ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
            if tweakListOptions != nil {
               tweakListOptions(&options)
            }
            return client.AppsV1().Deployments(namespace).List(context.TODO(), options)
         },
         WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
            if tweakListOptions != nil {
               tweakListOptions(&options)
            }
            return client.AppsV1().Deployments(namespace).Watch(context.TODO(), options)
         },
      },
      &appsv1.Deployment{},
      resyncPeriod,
      indexers,
   )
}
```



## DynamicSharedInformerFactory

```go
type dynamicSharedInformerFactory struct {
   client        dynamic.Interface
   defaultResync time.Duration
   namespace     string

   lock      sync.Mutex
   informers map[schema.GroupVersionResource]informers.GenericInformer
   // startedInformers is used for tracking which informers have been started.
   // This allows Start() to be called multiple times safely.
   startedInformers map[schema.GroupVersionResource]bool
   tweakListOptions TweakListOptionsFunc
}
```

缓存informer的结构为：

`informers map[schema.GroupVersionResource]informers.GenericInformer`

<br>

先看一下 DynamicSharedInformerFactory 的用法：

使用 DynamicSharedInformerFactory 监听 `IPAMBlock` crd资源

```go
package main

import (
	"context"
	"fmt"
	"os"
	"path"
	"time"

	metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
	"k8s.io/apimachinery/pkg/apis/meta/v1/unstructured"
	"k8s.io/apimachinery/pkg/runtime/schema"
	"k8s.io/apimachinery/pkg/util/rand"
	"k8s.io/client-go/dynamic"
	"k8s.io/client-go/dynamic/dynamicinformer"
	"k8s.io/client-go/tools/cache"
	"k8s.io/client-go/tools/clientcmd"
)

var (
	namespace         = "default"
	label             = "informer-dynamic-simple-" + rand.String(6)
	IPAMBlockResource = schema.GroupVersionResource{
		Group:    "crd.projectcalico.org",
		Version:  "v1",
		Resource: "ipamblocks",
	}
)

func main() {
	home, err := os.UserHomeDir()
	if err != nil {
		panic(err)
	}

	config, err := clientcmd.BuildConfigFromFlags("", path.Join(home, ".kube/config"))
	if err != nil {
		panic(err.Error())
	}

	client, err := dynamic.NewForConfig(config)
	if err != nil {
		panic(err.Error())
	}

	// Create a shared informer factory.
	//   - A factory is essentially a struct keeping a map (type -> informer).
	//   - 5*time.Second is a default resync period (for all informers).
	factory := dynamicinformer.NewDynamicSharedInformerFactory(client, 5*time.Second)

	// When informer is requested, the factory instantiates it and keeps the
	// the reference to it in the internal map before returning.
	dynamicInformer := factory.ForResource(IPAMBlockResource)
	dynamicInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc: func(obj interface{}) {
			cm := obj.(*unstructured.Unstructured)
			fmt.Printf("Informer event: ADDED %s/%s\n", cm.GetNamespace(), cm.GetName())
		},
		UpdateFunc: func(old, new interface{}) {
			cm := old.(*unstructured.Unstructured)
			fmt.Printf("Informer event: UPDATED %s/%s\n", cm.GetNamespace(), cm.GetName())
		},
		DeleteFunc: func(obj interface{}) {
			cm := obj.(*unstructured.Unstructured)
			fmt.Printf("Informer event: DELETED %s/%s\n", cm.GetNamespace(), cm.GetName())
		},
	})

	ctx, cancel := context.WithCancel(context.Background())
	defer cancel()

	// Start the informers' machinery.
	//   - Start() starts every Informer requested before using a goroutine per informer.
	//   - A started Informer will fetch ALL the IPAMBlocks from all the namespaces
	//     (using a lister) and trigger `AddFunc`` for each found IPAMBlock object.
	//     Use NewSharedInformerFactoryWithOptions() to make the lister fetch only
	//     a filtered subset of objects.
	//   - All IPAMBlocks added, updated, or deleted after the informer has been synced
	//     will trigger the corresponding callback call (using a watch).
	//   - Every 5*time.Second the UpdateFunc callback will be called for every
	//     previously fetched IPAMBlock (so-called resync period).
	factory.Start(ctx.Done())

	// factory.Start() releases the execution flow without waiting for all the
	// internal machinery to warm up. We use factory.WaitForCacheSync() here
	// to poll for cmInformer.Informer().HasSynced(). Essentially, it's just a
	// fancy way to write a while-loop checking HasSynced() flags for all the
	// registered informers with 100ms delay between iterations.
	for gvr, ok := range factory.WaitForCacheSync(ctx.Done()) {
		if !ok {
			panic(fmt.Sprintf("Failed to sync cache for resource %v", gvr))
		}
	}

	// Stay for a couple more seconds to observe resyncs.
	time.Sleep(1000 * time.Second)
}

func createIPAMBlock(client dynamic.Interface) *unstructured.Unstructured {
	cm := &unstructured.Unstructured{
		Object: map[string]interface{}{
			"apiVersion": "v1",
			"kind":       "IPAMBlock",
			"metadata": map[string]interface{}{
				"namespace":    namespace,
				"generateName": "informer-dynamic-simple-",
				"labels": map[string]interface{}{
					"example": label,
				},
			},
			"data": map[string]interface{}{
				"foo": "bar",
			},
		},
	}

	cm, err := client.
		Resource(IPAMBlockResource).
		Namespace(namespace).
		Create(context.Background(), cm, metav1.CreateOptions{})
	if err != nil {
		panic(err.Error())
	}

	fmt.Printf("Created IPAMBlock %s/%s\n", cm.GetNamespace(), cm.GetName())
	return cm
}

func deleteIPAMBlock(client dynamic.Interface, cm *unstructured.Unstructured) {
	err := client.
		Resource(IPAMBlockResource).
		Namespace(cm.GetNamespace()).
		Delete(context.Background(), cm.GetName(), metav1.DeleteOptions{})
	if err != nil {
		panic(err.Error())
	}
	if err != nil {
		panic(err.Error())
	}

	fmt.Printf("Deleted IPAMBlock %s/%s\n", cm.GetNamespace(), cm.GetName())
}
```

```go
// client-go/dynamic/dynamicinformer/informer.go
func (f *dynamicSharedInformerFactory) ForResource(gvr schema.GroupVersionResource) informers.GenericInformer {
   f.lock.Lock()
   defer f.lock.Unlock()

   key := gvr
   informer, exists := f.informers[key]
   if exists {
      return informer
   }

   informer = NewFilteredDynamicInformer(f.client, gvr, f.namespace, f.defaultResync, cache.Indexers{cache.NamespaceIndex: cache.MetaNamespaceIndexFunc}, f.tweakListOptions)
   f.informers[key] = informer

   return informer
}
```

```go
// client-go/dynamic/dynamicinformer/informer.go
// NewFilteredDynamicInformer constructs a new informer for a dynamic type.
func NewFilteredDynamicInformer(client dynamic.Interface, gvr schema.GroupVersionResource, namespace string, resyncPeriod time.Duration, indexers cache.Indexers, tweakListOptions TweakListOptionsFunc) informers.GenericInformer {
   return &dynamicInformer{
      gvr: gvr,
      informer: cache.NewSharedIndexInformer(
         &cache.ListWatch{
            ListFunc: func(options metav1.ListOptions) (runtime.Object, error) {
               if tweakListOptions != nil {
                  tweakListOptions(&options)
               }
               return client.Resource(gvr).Namespace(namespace).List(context.TODO(), options)
            },
            WatchFunc: func(options metav1.ListOptions) (watch.Interface, error) {
               if tweakListOptions != nil {
                  tweakListOptions(&options)
               }
               return client.Resource(gvr).Namespace(namespace).Watch(context.TODO(), options)
            },
         },
         // ！！
         &unstructured.Unstructured{},
         resyncPeriod,
         indexers,
      ),
   }
}
```



## InformersMap

InformersMap 是 controller-runtime 中的内容，封装比较复杂，这里只了解 sharedIndexInformer 在其中的使用。

```go
// InformersMap create and caches Informers for (runtime.Object, schema.GroupVersionKind) pairs.
// It uses a standard parameter codec constructed based on the given generated Scheme.
type InformersMap struct {
   // scheme maps runtime.Objects to GroupVersionKinds
   scheme *runtime.Scheme

   // config is used to talk to the apiserver
   config *rest.Config

   // mapper maps GroupVersionKinds to Resources
   mapper meta.RESTMapper

   // informers is the cache of informers keyed by their type and groupVersionKind
   informers informers

   // codecs is used to create a new REST client
   codecs serializer.CodecFactory

   // paramCodec is used by list and watch
   paramCodec runtime.ParameterCodec

   // stop is the stop channel to stop informers
   stop <-chan struct{}

   // resync is the base frequency the informers are resynced
   // a 10 percent jitter will be added to the resync period between informers
   // so that all informers will not send list requests simultaneously.
   resync time.Duration

   // mu guards access to the map
   mu sync.RWMutex

   // start is true if the informers have been started
   started bool

   // startWait is a channel that is closed after the
   // informer has been started.
   startWait chan struct{}

   // namespace is the namespace that all ListWatches are restricted to
   // default or empty string means all namespaces
   namespace string

   // selectors are the label or field selectors that will be added to the
   // ListWatch ListOptions.
   selectors func(gvk schema.GroupVersionKind) Selector

   // disableDeepCopy indicates not to deep copy objects during get or list objects.
   disableDeepCopy DisableDeepCopyByGVK

   // transform funcs are applied to objects before they are committed to the cache
   transformers TransformFuncByGVK
}
```

缓存informer的结构为：

`informers informers`

其中，informers 结构体如下：

```go
type informers struct {
   Structured   map[schema.GroupVersionKind]*MapEntry
   Unstructured map[schema.GroupVersionKind]*MapEntry
   Metadata     map[schema.GroupVersionKind]*MapEntry
}
```

MapEntry 结构体如下：

```go
// MapEntry contains the cached data for an Informer.
type MapEntry struct {
   // Informer is the cached informer
   Informer cache.SharedIndexInformer

   // CacheReader wraps Informer and implements the CacheReader interface for a single type
   Reader CacheReader
}
```



```go
// controller-runtime/pkg/cache/internal/informers_map.go
// Get will create a new Informer and add it to the map of specificInformersMap if none exists.  Returns
// the Informer from the map.
func (ip *InformersMap) Get(ctx context.Context, gvk schema.GroupVersionKind, obj runtime.Object) (bool, *MapEntry, error) {
   // Return the informer if it is found
   i, started, ok := ip.get(gvk, obj)
   if !ok {
      var err error
      if i, started, err = ip.addInformerToMap(gvk, obj); err != nil {
         return started, nil, err
      }
   }

   if started && !i.Informer.HasSynced() {
      // Wait for it to sync before returning the Informer so that folks don't read from a stale cache.
      if !cache.WaitForCacheSync(ctx.Done(), i.Informer.HasSynced) {
         return started, nil, apierrors.NewTimeoutError(fmt.Sprintf("failed waiting for %T Informer to sync", obj), 0)
      }
   }

   return started, i, nil
}
```

```go
// controller-runtime/pkg/cache/internal/informers_map.go

func (ip *InformersMap) addInformerToMap(gvk schema.GroupVersionKind, obj runtime.Object) (*MapEntry, bool, error) {
   ip.mu.Lock()
   defer ip.mu.Unlock()

   // Check the cache to see if we already have an Informer.  If we do, return the Informer.
   // This is for the case where 2 routines tried to get the informer when it wasn't in the map
   // so neither returned early, but the first one created it.
   if i, ok := ip.informersByType(obj)[gvk]; ok {
      return i, ip.started, nil
   }

   // Create a NewSharedIndexInformer and add it to the map.
   lw, err := ip.makeListWatcher(gvk, obj)
   if err != nil {
      return nil, false, err
   }
   ni := cache.NewSharedIndexInformer(&cache.ListWatch{
      ListFunc: func(opts metav1.ListOptions) (runtime.Object, error) {
         ip.selectors(gvk).ApplyToList(&opts)
         return lw.ListFunc(opts)
      },
      WatchFunc: func(opts metav1.ListOptions) (watch.Interface, error) {
         ip.selectors(gvk).ApplyToList(&opts)
         opts.Watch = true // Watch needs to be set to true separately
         return lw.WatchFunc(opts)
      },
   }, obj, resyncPeriod(ip.resync)(), cache.Indexers{
      cache.NamespaceIndex: cache.MetaNamespaceIndexFunc,
   })

   // Check to see if there is a transformer for this gvk
   if err := ni.SetTransform(ip.transformers.Get(gvk)); err != nil {
      return nil, false, err
   }

   rm, err := ip.mapper.RESTMapping(gvk.GroupKind(), gvk.Version)
   if err != nil {
      return nil, false, err
   }

   i := &MapEntry{
      Informer: ni,
      Reader: CacheReader{
         indexer:          ni.GetIndexer(),
         groupVersionKind: gvk,
         scopeName:        rm.Scope.Name(),
         disableDeepCopy:  ip.disableDeepCopy.IsDisabled(gvk),
      },
   }
   ip.informersByType(obj)[gvk] = i

   // Start the Informer if need by
   // TODO(seans): write thorough tests and document what happens here - can you add indexers?
   // can you add eventhandlers?
   if ip.started {
      go i.Informer.Run(ip.stop)
   }
   return i, ip.started, nil
}
```


































