# client-go解析(7) - Reconcile


## 概述

使用过 kubebuilder 开发Operator项目的话，对 `Reconcile` 函数应该不陌生。

由于 kubebuilder(controller-runtime)的高度封装，我们编写 operator 控制器，重点只需要编写 `Reconcile` 函数，所以一开始总是不能正确理解 `Reconcile` 函数的意义。



## 控制器

控制器，可以理解为：

1，持有多个与关注资源相关 processorListener（调用 AddEventHandler 注册processorListener） 和 一个 workqueue，

2，processorListener 用于获取相关事件添加到 workqueue 中。

3，然后启动若干个 worker 消费 workqueue，而这里的 worker，主要就是调用 `Reconcile` 函数。

接下来，看一下这个模式在 kube-controller-manager 和 controller-runtinme 里面的具体实现

### DeploymentController

```go
type DeploymentController struct {
   rsControl controller.RSControlInterface
   client    clientset.Interface

   eventBroadcaster record.EventBroadcaster
   eventRecorder    record.EventRecorder

   syncHandler func(ctx context.Context, dKey string) error
   enqueueDeployment func(deployment *apps.Deployment)

   dLister appslisters.DeploymentLister
   rsLister appslisters.ReplicaSetLister
   podLister corelisters.PodLister

   dListerSynced cache.InformerSynced
   rsListerSynced cache.InformerSynced
   podListerSynced cache.InformerSynced

   queue workqueue.RateLimitingInterface
}
```

#### processorListener

DeploymentController 中，在 NewDeploymentController 方法初始化 processorListener，如下：

```go
// NewDeploymentController creates a new DeploymentController.
func NewDeploymentController(dInformer appsinformers.DeploymentInformer, rsInformer appsinformers.ReplicaSetInformer, podInformer coreinformers.PodInformer, client clientset.Interface) (*DeploymentController, error) {
	eventBroadcaster := record.NewBroadcaster()

	dc := &DeploymentController{
		client:           client,
		eventBroadcaster: eventBroadcaster,
		eventRecorder:    eventBroadcaster.NewRecorder(scheme.Scheme, v1.EventSource{Component: "deployment-controller"}),
		queue:            workqueue.NewNamedRateLimitingQueue(workqueue.DefaultControllerRateLimiter(), "deployment"),
	}
	dc.rsControl = controller.RealRSControl{
		KubeClient: client,
		Recorder:   dc.eventRecorder,
	}

	dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addDeployment,
		UpdateFunc: dc.updateDeployment,
		// This will enter the sync loop and no-op, because the deployment has been deleted from the store.
		DeleteFunc: dc.deleteDeployment,
	})
	rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		AddFunc:    dc.addReplicaSet,
		UpdateFunc: dc.updateReplicaSet,
		DeleteFunc: dc.deleteReplicaSet,
	})
	podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
		DeleteFunc: dc.deletePod,
	})

	dc.syncHandler = dc.syncDeployment
	dc.enqueueDeployment = dc.enqueue

	dc.dLister = dInformer.Lister()
	dc.rsLister = rsInformer.Lister()
	dc.podLister = podInformer.Lister()
	dc.dListerSynced = dInformer.Informer().HasSynced
	dc.rsListerSynced = rsInformer.Informer().HasSynced
	dc.podListerSynced = podInformer.Informer().HasSynced
	return dc, nil
}
```

#### workqueue



```go
dInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
   AddFunc:    dc.addDeployment,
   UpdateFunc: dc.updateDeployment,
   // This will enter the sync loop and no-op, because the deployment has been deleted from the store.
   DeleteFunc: dc.deleteDeployment,
})
rsInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
   AddFunc:    dc.addReplicaSet,
   UpdateFunc: dc.updateReplicaSet,
   DeleteFunc: dc.deleteReplicaSet,
})
podInformer.Informer().AddEventHandler(cache.ResourceEventHandlerFuncs{
   DeleteFunc: dc.deletePod,
})
```

简单查看几个方法

```go
func (dc *DeploymentController) addDeployment(obj interface{}) {
   d := obj.(*apps.Deployment)
   klog.V(4).InfoS("Adding deployment", "deployment", klog.KObj(d))
   dc.enqueueDeployment(d)
}
```

直接入队

```go
// addReplicaSet enqueues the deployment that manages a ReplicaSet when the ReplicaSet is created.
func (dc *DeploymentController) addReplicaSet(obj interface{}) {
   rs := obj.(*apps.ReplicaSet)

   if rs.DeletionTimestamp != nil {
      // On a restart of the controller manager, it's possible for an object to
      // show up in a state that is already pending deletion.
      dc.deleteReplicaSet(rs)
      return
   }

   // If it has a ControllerRef, that's all that matters.
   if controllerRef := metav1.GetControllerOf(rs); controllerRef != nil {
      d := dc.resolveControllerRef(rs.Namespace, controllerRef)
      if d == nil {
         return
      }
      klog.V(4).InfoS("ReplicaSet added", "replicaSet", klog.KObj(rs))
      dc.enqueueDeployment(d)
      return
   }

   // Otherwise, it's an orphan. Get a list of all matching Deployments and sync
   // them to see if anyone wants to adopt it.
   ds := dc.getDeploymentsForReplicaSet(rs)
   if len(ds) == 0 {
      return
   }
   klog.V(4).InfoS("Orphan ReplicaSet added", "replicaSet", klog.KObj(rs))
   for _, d := range ds {
      dc.enqueueDeployment(d)
   }
}
```

#### Reconcile





未完待续。。。

