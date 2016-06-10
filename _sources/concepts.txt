.. _concepts:

==================
Pylons 理念
==================

译者：alswl_

.. _alswl: http://log4d.com/


我们需要了解一些 Pylons 的基本概念：请求和响应是怎么贯穿框架栈结构的，
Pylons 是如何让定制变得简单，还有出于什么原因这么设计的。

这一章将介绍 :term:`WSGI` 应用的基本概念和 :term:`WSGI Middleware` ，
后者用来解释 Pylons 如何利用它们组合完成一个 web 框架。

在开始阅读下面的内容之前，请先在 :ref:`getting_started` 的指引下创建一个工程。

*****************************
Pylons 溯源
*****************************

Pylons 项目和其他大多数 web 框架有点不一样。
它没有和其他框架那样凭空创建一个新项目。而是在它运行的时候，
从 Pylons 中载入模块，声明 WSGI 应用和应用栈，再返回结果。

只要你愿意，新工程可以使用任意 WSGI 应用来替代 Pylons 的包引入。
开发者可以按他们的想法自行打造一个自由可伸缩的 web 应用，。

默认情况下，项目会将大部分开发者所需要的标准组件配置好，比如 sessions、
模板引擎、缓存、请求/响应对象和 :term:`ORM` 。开发者也可以自由定制自己所需要的，
而不是让项目自身搞定（在框架内部代码里实现）。

依据这种理念，Pylons 会设定一些它认为 *必须* 的组件，
然后开发者可以根据项目需要自行设定。
通过保留大量标准化的 :term:`WSGI` 组件接口，Pylons 提供一种前所未有的可定制级别，

*****************
WSGI 应用
*****************

WSGI 描述了如何和 HTTP 服务器协同工作，在 :pep:`333` 被声明。
它涉及到如何从请求中获取 HTTP 头和如何在 HTTP 头中返回内容。

一个「Hello World」 WSGI 应用：

.. code-block :: python
    
    def simple_app(environ, start_response):
        start_response('200 OK', [('Content-type', 'text/html')])
        return ['<html><body>Hello World</body></html>']

其实这个 WSGI 没做什么东西，仅仅设定了 200 返回状态码、HTTP 'Content-type'
头和一些 HTML。

WSGI 标准规定了 `一组在环境变量中设定的键值集 <http://www.python.org/dev/peps/pep-0333/#environ-variables>`_.

WSGI 接口其实就是调用一个方法（或者一个类），这个方法有两个参数，
然后在和上面范例那样返回一个结果。这种思想贯穿了 Pylons：
一个标准化的接口将控制权传递到下一个组件。

在新创建的项目的 :file:`config/middleware.py` 文件中， `make_app`
就是用来负责创建 WSGI 应用的。它被包裹在 WSGI 中间件（下文会解释）中，
然后再被返回到上层来处理 HTTP 服务器的请求。

.. _wsgi-middleware:

***************
WSGI 中间件
***************

在 :file:`config/middleware.py` 中可以看到，一个 Pylons 应用是层层包裹的，
每一层有自己的功能。通过中间件里包裹 Pylons 应用的过程看上去很像洋葱结构。

.. image:: _static/pylons_as_onion.png
   :alt: Pylons middleware onion analogy
   :align: center

一旦中间件完成对 Pylons 应用的包裹，make_app 函数就会返回如下完整的结构
（先列出最外层的）：

.. code-block:: text

    Registry Manager
        Status Code Redirect
            Error Handler
                Cache Middleware
                    Session Middleware
                        Routes Middleware
                            Pylons App (WSGI Application)

WSGI 中间层被广泛运用在 Pylons 中，它会给 基础 WSGI 应用添加各种功能。
在 Pylons 中，基础 WSGI 应用是类 :class:`~pylons.wsgiapp.PylonsApp` 。
It's responsible for looking in the
`environ` dict that was passed in (from the Routes Middleware).

要理解它如何运作，看下面这个中间层类范例：模拟处理 Google 跳转来的 `HTTP_REFERER` ：

.. code-block :: python
    
    class GoogleRefMiddleware(object):
        def __init__(self, app):
            self.app = app
        
        def __call__(self, environ, start_response):
            environ['google'] = False
            if 'HTTP_REFERER' in environ:
                if environ['HTTP_REFERER'].startswith('http://google.com'):
                    environ['google'] = True
            return self.app(environ, start_response)

这个类行为和 WSGI 中间件很像，它添加了一些变量到环境上下文、初始化的 WSGI 参数。
一个新的 Pylons 工程会按照这样来建造 `WSGI 协议栈` 。

有些中间层只向 `environ` 、 HTTP 头（比如 Session 会添加 cookie 头信息）
和返回项添加内容，比如 Session、Routes 和 Cache。而另外有一些则返回错误跳转，
错误可以在整个流程被处理，然后改变输出内容。

*******************
Controller 调度
*******************

当一个请求通过中间件时候，请求 URL 会被 RoutesMiddleware 解析，
如果它符合某个范式（参看 :ref:`url-config` ），那么将要被调用的控制器会在
:class:`~pylons.wsgiapp.PylonsApp` 中被设定到 `environ` 。

然后 :class:`~pylons.wsgiapp.PylonsApp` 尝试在 :file:`controllers`
中查找匹配名字的控制器（控制器名字加上 Controller，比如 HelloController）。
一旦找到控制器，就会和其他 WSGI 应用一样被 :class:`~pylons.wsgiapp.PylonsApp`
调用。

.. versionadded:: 1.0
    Controller name can also be a dotted path to the module / callable that
    should be imported and called. For example, to use a controller named
    'Foo' that is in the 'bar.controllers' package, the controller name
    would be `bar.controllers:Foo`.

这也是为什么工程中的 :file:`lib/base.py` 继承了
:class:`~pylons.controllers.core.WSGIController` ，并且有 `__call__`
方法来来设定 `environ` 和 `start_response` 。
:class:`~pylons.controllers.core.WSGIController` 会查找路由给出的 `action`
所在的类并调用它，然后返回请求的结果。
 
******
Paster
******

执行命令 :command:`paster` 就可以看到所有命令参数：

.. code-block :: bash
    
    $ paster
    Usage: paster [paster_options] COMMAND [command_options]

    Options:
      --version         show program's version number and exit
      --plugin=PLUGINS  Add a plugin to the list of commands (plugins are Egg
                        specs; will also require() the Egg)
      -h, --help        Show this help message

    Commands:
      create          Create the file layout for a Python distribution
      grep            Search project for symbol
      help            Display help
      make-config     Install a package and create a fresh config file/directory
      points          Show information about entry points
      post            Run a request for the described application
      request         Run a request for the described application
      serve           Serve the described application
      setup-app       Setup an application, given a config file

    pylons:
      controller      Create a Controller and accompanying functional test
      restcontroller  Create a REST Controller and accompanying functional test
      shell           Open an interactive shell with the Pylons app loaded

如果在 Pylons 项目目录下执行命令 :command:`paster` ，你会看上上述打印信息。
如果不在目录下，最后一段 `pylons` 信息就不会被显示出来。这个判断依赖于
:command:`pylons` 脚本的一个动态插件。

Pylons 工程中，有一个目录是以 `.egg-info` 结尾，目录里有一个文件
:file:`paster_plugins.txt` 。这个文件会被 :command:`paster` 读取，
从而一些库被动态加载到命令中去。通过它们，Pylons 提供了上述几个快捷命令。

***********************
载入应用
***********************

只需键入 :command:`paster` 即可载入运行应用：

.. code-block :: bash
    
    $ paster serve development.ini

这条命令让 paster 启动服务器模式。它会尝试根据指定的配置文件加载服务器和 WSGI
应用。 `[server]` 用来指定使用什么服务器， `[app]` 则用来指定哪个 WSGI 应用。

`helloworld` 的一个简单的 Python 包声明（蟒蛇蛋） :file:`development.ini` ：

.. code-block :: ini
    
    [app:main]
    use = egg:helloworld

它将告诉 paster 需要加载 helloworld :term:`egg` 来找到 WSGI 应用。
每个 Pylons 应用都会在 :file:`setup.py` 中使用一行指定如何打包 WSGI 应用：

.. code-block :: python
    
    entry_points="""
    [paste.app_factory]
    main = helloworld.config.middleware:make_app

    [paste.app_install]
    main = pylons.util:PylonsInstaller
    """,

如上， `make_app` 方法指定 Paster（在 :command:`paster` 包中） 需要加载
`main` 这个 WSGI 应用。

`make_app` 方法会在稍后被调用，然后服务器（默认是 HTTP 服务器）就会运行这个
WSGI 应用。
