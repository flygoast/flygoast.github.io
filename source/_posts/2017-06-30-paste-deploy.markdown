---
layout: post
title: "PasteDeploy介绍"
date: 2017-06-30 00:04:03 +0800
comments: true
categories: OpenStack
---
之前的文章<<[WSGI介绍](http://www.just4coding.com/blog/2017/01/18/wsgi/)>>简要介绍了WSGI中的`server`, `application`，`middleware`等概念。`WSGI Server`收到请求后调用`WSGI application`的入口点(一般为`callable`对象或者函数)来处理请求。比如，`uWSGI`和`mod_wsgi`默认调用名为application的入口点。

PasteDeploy是一套发现和配置WSGI应用的系统。它根据指定的配置文件动态生成入口点和组织`WSGI application`间的逻辑关系。配置文件为INI格式。每个配置文件可以包含多个`section`, `section`以格式为`[type:name]`形式的段名开始，段名行之后为具体配置。示例如下:

```ini
[DEFAULT]
email = alice@example.org

[app:main]
use = egg:BlogApp
database = sqlite:/home/me/blog.db
set email = bob@example.org

[app:wiki]
use = call:mywiki.main:application
database = sqlite:/home/me/wiki.db
```

其中，`[DEFAULT]`段中为全局配置，各section中可以覆盖`[DEFAULT]`段中配置。如示例中的`set email`配置。

<!--more-->

常用的type有:

* app: `WSGI application`
* composite: WSGI应用的组合，它可以根据规则将请求分发给不同的应用
* filter：`WSGI middleware`
* pipeline: 按顺序执行的一系列`middleware`和应用

下面以示例来说明PasteDeploy的用法:
我们使用`uwsgi`做为`WSGI Server`，首先安装uwsgi环境:

```bash
yum install -y uwsgi uwsgi-plugin-python
```

安装Paste、WebOb，PasteDeploy:

```bash
pip install Paste WebOb PasteDeploy
```

创建三个文件:app.py, paste.ini, driver.py:

```bash
mkdir wsgi && cd wsgi
touch app.py paste.ini driver.py
```

app.py内容如下:

```python
from webob import Response
from webob.dec import wsgify

@wsgify
def demo_app(req):
    return Response('Hello Demo\n')

def demo_app_factory(global_config, **local_config):
    return demo_app
```

paste.ini内容如下:

```ini
[app:main]
paste.app_factory = app:demo_app_factory
```

driver.py内容如下:

```python
from paste.deploy import loadapp
application = loadapp('config:/home/vagrant/wsgi/paste.ini’)
```

启动uwsgi加载driver.py:

```bash
uwsgi --http-socket 127.0.0.1:8080 --plugins-dir /usr/lib64/uwsgi/ --plugin python_plugin.so -w driver
```

打开另一终端访问8080端口，结果如下:

```plain
[root@localhost src]# curl http://127.0.0.1:8080
Hello Demo
```

下面具体各文件内容。首先来看`driver.py`文件, 我们使用`paste.deploy.loadapp`来动态生成名为`application`的WSGI应用入口点。`loadapp`参数为`config:`形式的配置文件路径和以及`#name`形式的section名称，比如:

```plain
config:/path/to/paste.ini#start
```

若没有指定名称，则使用名称为`main`的section。


接下来看paste.ini。paste.ini中只包含一个app section。
section中有两种方法来指定要执行的WSGI应用。
一种是使用另外的URI或者应用名，如:

```ini
[app:myapp]
use = config:another_config_file.ini#app_name
```

另外一种方式是直接指定需要执行的Python代码，如：

```ini
[app:myapp]
paste.app_factory = myapp.modulename:app_factory
```

这种方式通过调用工厂函数来生成入口点。上述示例表示需要`import`模块`myapp.modulename`, 再调用`app_factory`获取WSGI应用入口点。`app_factory`的原型如下:

```python
def app_factory(global_config, **local_conf):
    return wsgi_app
```

`global_config`是一个字典类型，存储INI配置文件中放在`[DEFAULT]` section中的参数。`local_conf`表示关键字参数，存储相应section侧段中参数。

下面看另一个示例。修改app.py为:

```python
from webob import Response
from webob.dec import wsgify

@wsgify
def demo_app(req):
    if req.environ.has_key('test'):
        req.environ["test"] += "app\n" 
    else:
        req.environ["test"] = "app\n" 

    return Response(req.environ['test'])

def demo_app_factory(global_config, **local_config):
    return demo_app
```

再创建文件foo_filter.py，内容如下:

```python
from webob.dec import wsgify

@wsgify.middleware()
def foo_filter(req, app):
    if req.environ.has_key('test'):
        req.environ["test"] += "foo\n" 
    else:
        req.environ["test"] = "foo\n" 

    return app(req)

def filter_factory(global_config, **local_config):
    return foo_filter
```

创建文件bar_filter.py，内容如下:

```python
from webob.dec import wsgify

@wsgify.middleware()
def bar_filter(req, app):
    if req.environ.has_key('test'):
        req.environ["test"] += "bar\n" 
    else:
        req.environ["test"] = "bar\n" 

    return app(req)

def filter_factory(global_config, **local_config):
    return bar_filter
```

修改paste.ini为:

```ini
[pipeline:main]
pipeline = foo_filter bar_filter demo

[filter:foo_filter]
paste.filter_factory = foo_filter:filter_factory

[filter:bar_filter]
paste.filter_factory = bar_filter:filter_factory

[app:demo]
paste.app_factory = app:demo_app_factory
```

重启uwsgi, 再次访问8080端口:

```plain
[root@localhost src]# curl http://127.0.0.1:8080
foo
bar
app
```

首先来看`foo_filter.py`，它根据`req.environ[“test”]`是否存在，来直接赋值或者追加字符串”foo\n”。bar_filter.py逻辑类似。
再来看app.py, 它追加字符串”app\n”到`req.environ[“test”]`, 并将该字符串做为Response返回。

paste.ini中我们使用pipeline类型来定义filter执行顺序，从结果可以看到foo_filter,bar_filter和app按我们在INI配置文件中指定的顺序来执行。我们再次修改paste.ini中的pipeline为:

```plain
pipeline = bar_filter foo_filter demo
```

再次访问8080端口, 看到结果顺序也跟着调整:

```plain
[root@localhost src]# curl http://127.0.0.1:8080/
bar
foo
app
```

下面再以实例介绍`composite`类型:
修改paste.ini为:

```ini
[composite:main]
use = egg:Paste#urlmap
/foo = foo
/bar = bar

[app:foo]
paste.app_factory = foo_app:foo_app_factory

[app:bar]
paste.app_factory = bar_app:bar_app_factory
```

创建文件foo_app.py，内容如下:

```python
from webob import Response
from webob.dec import wsgify

@wsgify
def foo_app(req):
    return Response("foo\n")

def foo_app_factory(global_config, **local_config):
    return foo_app
```

创建bar_app.py，内容如下:

```python
from webob import Response
from webob.dec import wsgify

@wsgify
def bar_app(req):
    return Response("bar\n")

def bar_app_factory(global_config, **local_config):
    return bar_app
```

重启uwsgi, 分别访问`/foo`和`/bar`, 结果如下:

```plain
[root@localhost src]# curl http://127.0.0.1:8080/foo
foo
[root@localhost src]# curl http://127.0.0.1:8080/bar
bar
```

paste.ini中使用`paste`包中提供的应用`urlmap`根据请求的URL前缀将请求映射到不同的WSGI应用上, 这完成WEB框架的基本路由功能。

从上述几个例子，我们可以体会到PasteDeploy给WSGI应用开发带来的便利。OpenStack各项目实现都使用了PasteDeploy, 理解PasteDeploy是理解OpenStack API实现的基础。

详细说明参考官方文档:

* http://pastedeploy.readthedocs.io/en/latest/#

