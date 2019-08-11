---
layout: post
title: "Kubernetes服务目录简要介绍"
date: 2019-08-10 08:14:56 +0800
comments: true
categories: Kubernetes
---
在一些场景下，运行在`CloudFoundry`和`Kubernetes`等平台上的Cloud Native应用需要使用各种各样的外部服务，如数据库，消息队列等。但应用开发者并不想关心这些服务的配置、管理和位置等信息，以便将精力集中在业务逻辑开发上。为了满足这种需要，`CloudFoundry`及`Google`等一系列公司创建了[`Open Service Broker API(OSBAPI)`项目](https://www.openservicebrokerapi.org/)，致力于将外部服务的生命周期管理，应用平台与服务代理的交互，服务实例的访问等内容标准化。

`OSBAPI`规范定义了高度抽象的应用平台与外部服务代理的交互逻辑，规范化外部服务的使用，主要包括:

* 注册服务代理: 应用平台会获取服务代理提供的服务列表及相应规格，`OSBAPI`使用术语`Plan`来表示规格。
* `Provision`服务实例: `OSBAPI`使用术语`Provision`来统一指定服务资源的供应。在外部服务具体实现上，可以是创建虚拟机实例，也可以是分配资源，也可以是创建SaaS帐号等等。
* 绑定应用及服务实例: 当服务实例创建完成后，开发者想要其应用与该服务实例进行交互。从服务代理的角度来看，就是将应用与服务实例进行绑定。具体实现逻辑可以非常灵活，一般情况下，会生成访问该服务实例所需的新凭证。比如，生成数据库实例的IP、端口、用户名、密码等等。这些凭证会被返回给应用平台以提供给具体应用使用。

`OSBAPI`的具体工作模式如下图:

{% img /images/2019-08-10/1.png %}

<!--more-->

Kubernetes的[Service Catalog项目](https://svc-cat.io/)实现了`OSBAPI`规范，Kubernetes平台的应用通过service catalog来使用外部服务。
大体实现架构如图:
{% img /images/2019-08-10/2.png %}

Service catalog实现将service catalog资源对象的管理映射到`OSBAPI`相应资源的操作, 架构上主要包括两部分:

* API Server:
它基于[API-aggregation](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/apiserver-aggregation/)实了一个扩展API Server, 使用[etcd](https://etcd.io/)做为后端存储。
Kubernetes用户和service catalog controller通过与API Server交互完成Service Catalog资源的CRUD操作。
* Controller:
Controller实现了`OSBAPI`规范。它监控API Server上资源对象的事件，并完成与`OSBAPI`对应的操作。比如，当用户创建一个ClusterServiceBroker对象, API Server会存储该资源并产生一个事件。Controller会接收到该事件，接着去请求指定的服务代理获取该服务代理所提供的资源。

Service catalog的API组为: `servicecatalog.k8s.io`, 当前的API资源版本为: `v1beta1`，提供的主要资源有: 

* ClusterServiceBroker:
Service Broker在Kubernetes集群内的表示, 包含Service Broker的相关信息。
* ClusterServiceClass:
表示特定ServiceBroker所提供的外部服务，ClusterServiceBroker创建后，controller会自动从ClusterServiceBroker所提供的URL拉取服务列表，为每个外部服务创建一个ClusterServiceClass。
* ClusterServicePlan:
表示外部服务的规格，ClusterServiceBroker创建后，controller也会自动创建ClusterServicePlan对象。
* ServiceInstance:
外部服务的服务实例，创建该对象时，controller会发送`OSBAPI` `Provision`请求。
* ServiceBinding:
创建该对象后，controller发送`OSBAPI`的Binding请求，并将返回的凭证信息存储在kubernetes的Secret对象中。该Secret对象需要被挂载到需要使用该服务的Pod中。

其他的一些资源具体可参考: https://svc-cat.io/docs/resources/

当存储有相关凭证信息的`Secret`对象挂载到特定应用的`Pod`中，该应用则可以使用这些信息与外部服务进行交互, 如下图所示:
{% img /images/2019-08-10/3.png %}

下面通过一个实例来说明Kubernetes与外部服务的交互。

首先，我们基于python的[openbrokerapi](https://pypi.org/project/openbrokerapi/0.3.1/)来实现一个demo broker, 它只是简单的返回服务信息，并不真实分配服务资源。

源码`broker.py`内容如下:
```python
from openbrokerapi import api
from openbrokerapi.catalog import ServicePlan
from openbrokerapi.service_broker import (
    ServiceBroker,
    Service,
    ProvisionedServiceSpec,
    ProvisionState,
    UpdateServiceSpec,
    BindState,
    Binding,
    DeprovisionServiceSpec,
    LastOperation,
    UnbindDetails,
    ProvisionDetails,
    UpdateDetails,
    BindDetails,
    DeprovisionDetails
)

class ExampleServiceBroker(ServiceBroker):
    def catalog(self):
        return Service(
            id='6b6b5306-5c37-4fba-a402-186f8a0dfa0a',
            name='demo-service',
            description='demo service does nothing',
            bindable=True,
            plans=[
                ServicePlan(
                    id='9a411f4c-01fa-4996-a780-a0d1b8dcb234',
                    name='default',
                    description='default service plan',
                ),
            ],
            tags=['demo', 'service'],
            plan_updateable=True,
        )
    def provision(self, instance_id: str, service_details: ProvisionDetails,
                  async_allowed: bool) -> ProvisionedServiceSpec:
        return ProvisionedServiceSpec(
            state=ProvisionState.SUCCESSFUL_CREATED,
            dashboard_url="http://demo.local/{}".format(instance_id)
        )
    def bind(self, instance_id: str, binding_id: str, details: BindDetails) -> Binding:
        return Binding(
            state=BindState.SUCCESSFUL_BOUND,
            credentials = {
                "uri": "http://demo.local/{}".format(binding_id),
                "username": "abc",
                "password": "123456"
            }
        )
    def update(self, instance_id: str, details: UpdateDetails, async_allowed: bool) -> UpdateServiceSpec:
        pass
    def unbind(self, instance_id: str, binding_id: str, details: UnbindDetails):
        return details
    def deprovision(self, instance_id: str, details: DeprovisionDetails, async_allowed: bool) -> DeprovisionServiceSpec:
        return DeprovisionServiceSpec(
            is_async=False,
            operation=None
        )
    def last_operation(self, instance_id: str, operation_data: str) -> LastOperation:
        pass

api.serve(ExampleServiceBroker(), None, port=5000)
```

运行该broker:
```bash
python broker.py
```
该虚拟机IP为`10.0.0.231`，它监听`5000`端口。

我们假定Kubernetes集群搭建和Service Catalog已经安装完成。

首先，在Kubernetest集群创建ClusterServiceBroker:

`demo-broker.yaml`的内容如下:

```yaml
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ClusterServiceBroker
metadata:
  name: demo-broker
spec:
  url: http://10.0.0.231:5000
```

创建demo-broker:
```plain
[root@fg-t1 yaml]# kubectl create -f demo-broker.yaml
clusterservicebroker.servicecatalog.k8s.io/demo-broker created
```

查看`clusterservicebrokers`列表:
```plain
[root@fg-t1 yaml]# kubectl get clusterservicebrokers
NAME          URL                        STATUS   AGE
demo-broker   http://10.0.0.231:5000   Ready    70s
```
查看`clusterserviceclasses`列表, 可以看到demo-broker所提供的demo-service:
```plain
[root@fg-t1 yaml]# kubectl get clusterserviceclasses
NAME                                   EXTERNAL-NAME   BROKER        AGE
6b6b5306-5c37-4fba-a402-186f8a0dfa0a   demo-service    demo-broker   34s
```
查看`clusterserviceplans`列表:
```plain
[root@fg-t1 yaml]# kubectl get clusterserviceplans
NAME                                   EXTERNAL-NAME   BROKER        CLASS                                  AGE
9a411f4c-01fa-4996-a780-a0d1b8dcb234   default         demo-broker   6b6b5306-5c37-4fba-a402-186f8a0dfa0a   40s
```

接下来，创建服务实例。
`demo-instance.yaml`内容如下:
```yaml
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceInstance
metadata:
  name: demo-instance
spec:
  clusterServiceClassExternalName: demo-service
  planName: default
```
创建ServiceInstance对象:
```plain
[root@fg-t1 yaml]# kubectl create -f demo-instance.yaml
serviceinstance.servicecatalog.k8s.io/demo-instance created
```

查看实例列表，可以看到新创建的demo-instance实例:
```plain
[root@fg-t1 yaml]# kubectl get serviceinstances
NAME            CLASS                              PLAN      STATUS   AGE
demo-instance   ClusterServiceClass/demo-service   default   Ready    11s
```

接着，创建Binding:
`demo-binding.yaml`内容如下:
```yaml
apiVersion: servicecatalog.k8s.io/v1beta1
kind: ServiceBinding
metadata:
  name: demo-binding
spec:
  instanceRef:
    name: demo-instance
  secretName: demo-instance-credentials
```
创建ServiceBinding对象:
```plain
[root@fg-t1 yaml]# kubectl create -f demo-binding.yaml
servicebinding.servicecatalog.k8s.io/demo-binding created
```

查看ServiceBindings列表可以看到demo-binding创建成功:
```plain
[root@fg-t1 yaml]# kubectl get servicebindings
NAME           SERVICE-INSTANCE   SECRET-NAME                 STATUS   AGE
demo-binding   demo-instance      demo-instance-credentials   Ready    7s
```

我们在`demo-binding.yaml`中指定凭证信息存储在Secret对象`demo-instance-credentials`中，我们来查看这个Secret对象:
```plain
[root@fg-t1 yaml]# kubectl get secret demo-instance-credentials -o yaml
apiVersion: v1
data:
  password: MTIzNDU2
  uri: aHR0cDovL2RlbW8ubG9jYWwvYmY2YWRlZTItYmJkNC0xMWU5LWFhYmYtMDI0MmFjMTEwMDA3
  username: YWJj
kind: Secret
metadata:
  creationTimestamp: "2019-08-11T01:10:03Z"
  name: demo-instance-credentials
  namespace: default
  ownerReferences:
  - apiVersion: servicecatalog.k8s.io/v1beta1
    blockOwnerDeletion: true
    controller: true
    kind: ServiceBinding
    name: demo-binding
    uid: bf6adf2b-bbd4-11e9-aabf-0242ac110007
  resourceVersion: "307730"
  selfLink: /api/v1/namespaces/default/secrets/demo-instance-credentials
  uid: 7c159d9c-0cb0-4a4e-86da-ea7480255e5e
type: Opaque
```
```plain
[root@fg-t1 yaml]# echo 'aHR0cDovL2RlbW8ubG9jYWwvYmY2YWRlZTItYmJkNC0xMWU5LWFhYmYtMDI0MmFjMTEwMDA3' | base64 --decode
http://demo.local/bf6adee2-bbd4-11e9-aabf-0242ac110007
[root@fg-t1 yaml]# echo 'MTIzNDU2' |base64 --decode
123456
[root@fg-t1 yaml]# echo 'YWJj' |base64 --decode
abc
```

可以看到，返回的内容正是我们在`broker.py`中所指定的内容。

最后，我们清理掉这些实例内容:
```plain
[root@fg-t1 yaml]# kubectl delete servicebindings demo-binding
servicebinding.servicecatalog.k8s.io "demo-binding" deleted
[root@fg-t1 yaml]# kubectl delete serviceinstance demo-instance
serviceinstance.servicecatalog.k8s.io "demo-instance” deleted
[root@fg-t1 yaml]# kubectl delete clusterservicebroker demo-broker
clusterservicebroker.servicecatalog.k8s.io "demo-broker” deleted
```
