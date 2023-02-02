# client-go解析(7) - Reconcile


## 概述

使用过 kubebuilder 开发Operator项目的话，对 `Reconcile` 函数应该不陌生。

由于 kubebuilder(controller-runtime)的高度封装，我们编写 operator 控制器，重点只需要编写 `Reconcile` 函数，所以一开始总是不能正确理解 `Reconcile` 函数的意义。



## 控制器

控制器，可以理解为，

1，持有多个与关注资源相关 processorListener 和 一个 workqueue，

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

#### workqueue

#### Reconcile


