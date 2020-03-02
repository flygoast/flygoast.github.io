---
title: Kubernetes CDR和Custom Controller
date: 2020-03-02 22:19:18
tags: 
- Kubernetes
- CloudNative
- CNCF
categories: Kubernetes
---

![Photo by Lukas from Pexels](/images/2020-03-02/1.jpg "Photo by Lukas from Pexels")

在`Kubernetes`中, 所有的功能实现都以资源(`Resource`)形式体现。简单来说，资源就是Kubernetes的API端点(`endpoint`)，用于存储某种类型的对象。用户通过对资源对象的`CRUD`操作完成相应功能管理。Kubernetes 1.7之后添加了自定义资源(`Custom Resource`)的扩展能力，允许使用者通过定义自己的资源类型来扩展Kubernetes API，大大增强扩展Kubernetes的灵活性。这些自定义资源和`Pod`、`Deployment`等原生资源对象的操作方法基本一样。

用户使用`CRD(Custom Resource Definitions)`来定义自定义资源(`Custom Resource`)。`CRD`被创建后，`Kubernetes APIServer`会为`CRD`中指定的资源版本创建相应的RESTAPI端点。

`CRD`可以通过`YAML`文件来创建，`YAML`文件中指定`CRD`对象的规范，其中比较重要的字段有:

* **metadata.name**: 要创建的`CRD`名称
* **spec.group**: `CRD`对象所属的API组
* **spec.version**: `CRD`对象所属的API版本
* **spec.names**: 管理自定义资源所需要的名称描述，可以定义`plural`(复数名称), `singular`(单数名称)和`shortnames`(简称)等字段
* **spec.names.kind**: 指定自定义资源的类型名称
* **spec.scope**: 指明自定义资源有效范围，可以是整个集群有效(`clustered`)或命名空间内有效(`namespaced`)

<!--more-->

下面我们通过示例来说明创建`CRD`及相应的自定义资源。

首先创建CRD YAML文件`ac-crd.yaml`:
```yaml
apiVersion: apiextensions.k8s.io/v1beta1
kind: CustomResourceDefinition
metadata:
  name: appconfigs.stable.flygoast.com
spec:
  group: stable.flygoast.com
  version: v1beta1
  scope: Namespaced
  names:
    plural: appconfigs
    singular: appconfig
    kind: AppConfig
    shortNames:
    - ac
```
使用`kubectl`创建类型名称为`AppConfig`的`CRD`对象并查看结果, 我们的`CRD`对象已经创建完成。
```plain
[fg@kubernetes ~]$ kubectl create -f ac-crd.yaml
customresourcedefinition.apiextensions.k8s.io/appconfigs.stable.flygoast.com created
[fg@kubernetes ~]$ kubectl get crds
NAME                             CREATED AT
appconfigs.stable.flygoast.com   2020-03-01T09:41:23Z
```
此时所创建的`CRD`所对应的API端点是: `/apis/stable.flygoast.com/v1beta1/namespaces/*/appconfigs/`

接着创建一个`YAML`文件来创建`AppConfig`类型的对象。我们没有限制自定义对象的字段，字段也可以包含任意的数据。`ac1.yaml`代码如下:
```yaml
apiVersion: "stable.flygoast.com/v1beta1"
kind: AppConfig
metadata:
  name: ac1
spec:
  foo: "bar"
  comment: "This is ac1."
```
使用`kubectl`创建`AppConfig`对象, 并查看相应对象信息:
```plain
[fg@kubernetes ~]$ kubectl create -f ac1.yaml
appconfig.stable.flygoast.com/ac1 created
[fg@kubernetes ~]$ kubectl get ac -o yaml
apiVersion: v1
items:
- apiVersion: stable.flygoast.com/v1beta1
  kind: AppConfig
  metadata:
    creationTimestamp: "2020-03-01T09:44:58Z"
    generation: 1
    name: ac1
    namespace: default
    resourceVersion: "36942"
    selfLink: /apis/stable.flygoast.com/v1beta1/namespaces/default/appconfigs/ac1
    uid: d96307cf-50dc-44e9-b230-962b5cee8a5e
  spec:
    comment: This is ac1.
    foo: bar
kind: List
metadata:
  resourceVersion: ""
  selfLink: ""
```
使用起来看上去非常简单。

Kubernetes的哲学是声明式API，使用者只关心资源的最终状态，而不关心执行哪些行为使资源对象达到最终状态，这些变化过程由Kubernetes来完成。现在的AppConfig对象操作只是完成数据对象的存储与获取，完整的声明式API实现还需要实现一个Custom Controller去保证当前资源状态与所期望状态的同步。

Controller需要监视Kubernetes的API对象，当相应的资源对象发生改变时，实现相应的逻辑完成资源对象的状态更新。Kubernetes对外提供了丰富的API资源，Controller可以基于RESTAPI实现，也可以使用客户端库来实现。

已经实现的客户端库可以参考:
https://kubernetes.io/docs/reference/using-api/client-libraries/

实现一个完善的Custom Controller较为复杂，可以参考文后所列的一些参考文章，本文以一个简要的Controller实现说明下基本逻辑。我们的Custom Controller简单的监视AppConfig对象，当对象发生变化时，输出相应的日志。

我们这里使用Python库来实现。
首先编写controller代码`main.py`:
```python
import logging
import sys
from kubernetes import client, config, watch


log = logging.getLogger(__name__)
out_hdlr = logging.StreamHandler(sys.stdout)
out_hdlr.setFormatter(logging.Formatter('%(asctime)s %(message)s'))
out_hdlr.setLevel(logging.INFO)
log.addHandler(out_hdlr)
log.setLevel(logging.INFO)


def main():
    log.info("appconfig controller start")
    config.load_incluster_config()


    api = client.CustomObjectsApi()


    objs = api.list_namespaced_custom_object('stable.flygoast.com', 'v1beta1', 'default', 'appconfigs', watch=False)


    for o in objs["items"]:
        log.info("Object: {} content:\"{}\"".format(o["metadata"]["name"], o["spec"]["foo"]))


    rv = objs["metadata"]["resourceVersion"]


    w = watch.Watch()
    for item in w.stream(api.list_namespaced_custom_object, 'stable.flygoast.com', 'v1beta1', 'default', 'appconfigs', resource_version=rv):
        if item["type"] == "ADDED":
            log.info("Object:{} content:\"{}\" ADDED.".format(item["object"]["metadata"]["name"], item["object"]["spec"]["foo"]))
        elif item["type"] == "MODIFIED":
            log.info("Object:{} content:\"{}\" MODIFIED.".format(item["object"]["metadata"]["name"], item["object"]["spec"]["foo"]))
        elif item["type"] == "DELETED":
            log.info("Object:{} content:\"{}\" DELETED.".format(item["object"]["metadata"]["name"], item["object"]["spec"]["foo"]))


if __name__ == '__main__':
    main()
```
我们准备将该Controller运行在Kubernetes集群中，因而先打包成Docker镜像。
创建`Dockerfile`:
```plain
FROM python:latest
RUN pip install kubernetes
COPY main.py /main.py

ENTRYPOINT [ "python", "/main.py” ]
```

BUILD镜像并上传:
```plain
docker build -t flygoast/ac-controller .
docker push flygoast/ac-controller
```

由于我们的controller将在`POD`中运行并访问Kubernetes APIServer, 它需要访问Kubernetes API的权限。权限的分配过程如下:

* 创建专用`ServiceAccount`, 我们的POD将使用它运行
* 创建专用`Role`，指定所需要的API访问权限
* 创建`RoleBinding`, 将上述`ServiceAccount`和`Role`绑定

我们将上述的过程描述在`rabc.yaml`文件中:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: appconfig

---
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  namespace: default
  name: appconfig
rules:
- apiGroups: ["stable.flygoast.com"]
  resources: ["appconfigs"]
  verbs: ["get", "list", "watch”]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: appconfig
  namespace: default
subjects:
- kind: ServiceAccount
  name: appconfig
  namespace: default
roleRef:
  kind: Role
  name: appconfig
  apiGroup: rbac.authorization.k8s.io
```

通过kubectl完成权限添加操作:
```plain
[fg@kubernetes ~]$ kubectl apply -f rbac.yaml
serviceaccount/appconfig created
role.rbac.authorization.k8s.io/appconfig created
rolebinding.rbac.authorization.k8s.io/appconfig created
```

创建运行controller的`Deployment` YAML文件`ac-deployment.yaml`:
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ac-controller
  labels:
    app: custom-controller
spec:
  selector:
    matchLabels:
      app: custom-controller
  replicas: 1
  template:
    metadata:
      labels:
        app: custom-controller
    spec:
      serviceAccount: appconfig
      containers:
      - name: ac-controller
        image: flygoast/ac-controller:latest
        imagePullPolicy: IfNotPresent
```
完成controller部署:
```plain
[fg@kubernetes ~]$ kubectl apply -f ac-deployment.yaml
deployment.apps/ac-controller created
```

查看并找到相应pod:
```plain
[fg@kubernetes ~]$ kubectl get pod
NAME                             READY   STATUS    RESTARTS   AGE
ac-controller-7c996fddb6-66t9x   1/1     Running   0          45s
```

然后，我们查看该POD的日志, 可以看到controller获取到了当前已存在的对象:
```plain
[fg@kubernetes ~]$ kubectl logs -f ac-controller-7c996fddb6-66t9x
2020-03-01 13:59:16,586 appconfig controller start
2020-03-01 13:59:16,621 Object: ac1 content:"bar"
```

打开另一终端，创建另一个`AppConfig`对象的`YAML`文件`ac2.yaml`:
```yaml
apiVersion: "stable.flygoast.com/v1beta1"
kind: AppConfig
metadata:
  name: ac2
spec:
  foo: "AppConfig 2."
  comment: "This is ac2."
```
创建对象:
```plain
[fg@kubernetes ~]$ kubectl apply -f ac2.yaml
appconfig.stable.flygoast.com/ac2 created
```

可以之前的终端上查看到新的日志输出:
```plain
2020-03-01 14:02:06,596 Object:ac2 content:"AppConfig 2" ADDED.
```

新增对象的事件捕获到了。接着修改`ac2.yaml`:
```yaml
apiVersion: "stable.flygoast.com/v1beta1"
kind: AppConfig
metadata:
  name: ac2
spec:
  foo: "AppConfig 2. Changed."
  comment: "This is ac2.”
```

然后更新对象:
```plain
[fg@kubernetes ~]$ kubectl apply -f ac2.yaml
appconfig.stable.flygoast.com/ac2 configured
```

从之前的日志输出可以看到:
```plain
2020-03-01 14:10:16,319 Object:ac2 content:"AppConfig 2. Changed." MODIFIED.
```

对象更新的事件也被捕获到。接下来，我们删除对象`ac2`:
```plain
[fg@kubernetes ~]$ kubectl delete ac ac2
appconfig.stable.flygoast.com "ac2” deleted
```
从日志输出可以看到:
```plain
2020-03-01 14:11:05,511 Object:ac2 content:"AppConfig 2. Changed." DELETED.
```

简单的controller实现可以基于相应对象的`ADDED`、`MODIFIED`、`DELETED`事件调用回调函数，完成相应逻辑的处理。更为完善的Controller实现可以参考文后的链接。

参考链接:
* https://www.magalix.com/blog/extending-the-kubernetes-controller
* https://engineering.bitnami.com/articles/a-deep-dive-into-kubernetes-controllers.html
* https://engineering.bitnami.com/articles/kubewatch-an-example-of-kubernetes-custom-controller.html
* https://medium.com/programming-kubernetes/building-stuff-with-the-kubernetes-api-toc-84d751876650
* https://blog.meain.io/2019/accessing-kubernetes-api-from-pod/
* http://techgenix.com/crd-in-kubernetes/
* https://kubernetes.io/docs/concepts/extend-kubernetes/operator/
* https://flugel.it/building-kubernetes-operators-part-1-operator-pattern-and-concepts/
* https://stackoverflow.com/questions/47848258/kubernetes-controller-vs-kubernetes-operator
* https://octetz.com/docs/2019/2019-10-13-controllers-and-operators/

