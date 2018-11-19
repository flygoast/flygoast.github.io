---
layout: post
title: "Python TUI库npyscreen简要介绍"
date: 2018-11-19 15:18:36 +0800
comments: true
categories: MISC
---
之前的文章<<[TUI库newt和snack简要介绍](/blog/2018/09/07/newt/)>>介绍了如何使用`newt`库开发控制台应用程序。`newt`库是一个非常简单的库，只适合开发逻辑为顺序流程的程序，比如，典型的程序安装向导。当程序逻辑较为复杂时，可以直接使用`ncurses`来实现。但`ncurses`是一个较为底层的库，使用它开发工作量会较大。本文介绍一个使用更为简易但可实现复杂功能的TUI库:npyscreen。

当前最新版本为4.10.5，如图:

{% img /images/2018-11-19/1.png %}

<!--more-->

`Npyscreen`应用由三种对象构成:

* `Application`对象:负责应用程序的开始、结束，创建`Form`对象，处理事件, 主要有两种类型:
    * `NPSAppManaged`: 该类提供了开始和结束应用的框架，管理各类Form对象的显示，不需要自己完成该程序的主循环逻辑。
    * `NPSApp`:该类的实例需要在.main()中提供自定义的主循环逻辑，该类实例执行.run()时，该主循环逻辑会被执行。不推荐直接使用该类。
* `Form`对象:表示包含`Widget`对象的屏幕区域，主要有以下类型:
    * `FormBaseNew`: 空Form
    * `Form`: 带有`<<OK>>`按钮的Form
    * `ActionForm`: 带有`<<OK>>`和`<<Cancel>>`两个按钮的Form
    * `FormWithMenus`: 支持Menu组件的Form
* `Widget`对象:表示各种与用户交互的组件

具体细节，可以参考[官网文档](https://npyscreen.readthedocs.io/index.html)

`Npyscreen`更推荐使用面向对象的编程方法，继承库中的类来自定义所需要的类，通过覆盖默认方法来扩展应用程序需要的功能。

下面我们依然实现上篇文章中的登录示例来说明`npyscreen`的简要用法。

首先直接使用`PIP`安装`npyscreen`:
```bash
pip install npyscreen
```

我们的程序由两个`FORM`构成，第一个`FORM`用于使用者输入用户名和密码，如下图:

{% img /images/2018-11-19/2.png %}

当用户名和密码输入正确时，显示第二个`FORM`, 它用于显示登录成功的消息，如图:

{% img /images/2018-11-19/3.png %}

程序源代码如下:

```python
import npyscreen

class SuccessForm(npyscreen.Form):
    def create(self):
        self.name = "SUCCESS"
        self.wgFixedText = self.add(npyscreen.FixedText, value = "")

    def beforeEditing(self):
        username = self.parentApp.getForm('MAIN').username
        value = "User [\"%s\"] logined successed!" % (username)
        self.wgFixedText.value = value

    def afterEditing(self):
        self.parentApp.switchForm(None)

class LoginForm(npyscreen.ActionForm):
    def create(self):
        self.name = "LOGIN"
        self.parentApp.value = ""
        self.username = None
        self.password = None
        self.wgUsername = self.add(npyscreen.TitleText, name = "Username:")
        self.wgPassword = self.add(npyscreen.TitlePassword, name = "Password:")

    def beforeEditing(self):
        self.username = None
        self.password = None
        self.wgUsername.value = ''
        self.wgPassword.value = ''

    def on_ok(self):
        self.username = self.wgUsername.value
        self.password = self.wgPassword.value

        if len(self.username) == 0 or len(self.password) == 0:
            npyscreen.notify_confirm("USERNAME AND PASSWORD MUST NOT BE EMPTY!")
        else:
            if self.username != "root" or self.password != "123456":
                npyscreen.notify_confirm("INVALID USERNAME OR PASSWORD!”)
            else:
                self.parentApp.switchForm('SUCCESSFORM')

    def on_cancel(self):
        self.parentApp.switchForm(None)

class LoginApp(npyscreen.NPSAppManaged):
    def onStart(self):
        self.addForm("MAIN", LoginForm, name = "LOGIN", color = "IMPORTANT")
        self.addForm("SUCCESSFORM", SuccessForm, name = "SUCCESS", color = "WARNING")

if __name__ == '__main__':
    LoginApp().run()
```

首先，我们创建了一个`SuccessForm`类，它继承`npyscreen.Form`类, 它自带一个`<<OK>>`按钮。它用于显示登录成功消息。
`create`方法用于初始化该`Form`, 我们在其中创建了一个`FixedText`用于显示登录消息。术语`Edit`表示`Form`与用户的交互。方法`beforeEditing`在进入`Form`自身的`Edit`循环前调用，而`afterEditing`在交互完成`Form`要退出时被调用。我们在`beforeEditing`方法中将要显示的内容准备好。在`afterEditing`中通过调用`NPSAppManaged.switchForm(None)`退出程序。

另一个类`LoginForm`表示用户输入用户名和密码的`Form`。它继承`npyscreen.ActionForm`, 带有一个`<<OK>>`和`<<Cancel>>`按钮。`create`方法用于初始化该`Form`, 创建了两个输入框来接收用户输入。`beforeEditing`方法用于在显示前清空内容。`on_ok`和`on_cancel`方法分别处理`<<OK>>`按钮和`<<Cancel>>`按钮被触发后的逻辑。`<<OK>>`按钮触发后，我们获取并检查用户输入的内容。若输入的用户名和密码正确，则调用`NPSAppManaged.switchForm(’SUCCESSFORM’)`显示第二个`FORM`。当`<<Cancel>>`按钮触发时，退出应用程序。

`LoginApp`类表示整个应用，在`onStart`方法中，我们添加了以上两个`FORM`，默认显示名称为`MAIN`的`FORM`。
最后直接调用`NPSAppManaged.run()`方法开始该程序。

整体用法上非常简单，实现多屏应用及菜单等较为复杂的功能也不困难。只是官方文档上对于很多类及方法的描述并不是很详细，很多使用细节往往需要去翻阅源码，具体的一些示例可以参考:

https://github.com/vtr0n/npyscreen

这个GITHUB作者还实现了一个基于npyscreen的Telegram客户端，如图:

{% img /images/2018-11-19/4.png %}

是一个非常好的复杂应用的参考示例，具体代码参考: https://github.com/vtr0n/TelegramTUI


