.. _views:

=====
Views
=====

译者：alswl_

.. _alswl: http://log4d.com/

.. image:: _static/pylon4.jpg
   :alt: 
   :align: left
   :height: 434px
   :width: 368px

In the MVC paradigm the *view* manages the presentation of the model. 

在 MVC 模型中， *视图* 用来管理模型的表现形式。

用户看到并与之交互的是视图。对于 web 应用来说，一般是 HTML 界面。
目前 Web 应用界面绝大部分还是 HTML，但也有一些变得常见的其他形式。

这些界面技术包括 Macromedia Flash，JSON 和另外一些标记型语言比如 XHTML、XML/XSL、
WML 和 Web services。另外在界面层提供 REST API 形式的 web 应用也越来越多，
它们提供实效的方法从数据模型读写数据。

还有一些其他稍微复杂的数据展示方法，比如通过 SOAP 服务。

The growing adoption of RDF, the graph-based representation scheme that underpins the Semantic Web, brings a perspective that is strongly weighted towards machine-readability.

.. NOTE: As much as I love RDF I think the following paragraph is too verbose for our intro docs, maybe we can put this elsewhere -pjenvey
.. RDF model data is serialized into an undecorated, standardized format that can readily be processed and rendered by client applications of increasing sophistication, such as the MIT `Simile`__ project's "`Fresnel`__", "`Longwell`__" and "`Welkin`__" browser extensions.

.. .. __: http://simile.mit.edu/
.. .. __: http://simile.mit.edu/fresnel/
.. .. __: http://simile.mit.edu/longwell/
.. .. __: http://simile.mit.edu/welkin/

如何把这些应用界面都处理好是一种大挑战。
MVC 的一大优势就是可以轻松在这一堆方法中创建多种视图。

通常情况下，无论是在线商店或雇员列表应用，
视图不会对数据进行重大操作，它仅仅提供展现数据并向用户提供操作。

.. _templates:

*********
模板
*********

模板渲染引擎经常被用来处理视图展现。

控制器会调用模板引擎来渲染并返回::

    from helloworld.lib.base import BaseController, render

    class HelloController(BaseController):
        def sample(self):
            return render('/sample.mako')

使用 Mako 引擎的话，Mako 会主动查找 :file:`helloworld/templates` 目录
（假设项目的名字叫做 'helloworld'）下面叫做 file:`sample.mako` 的文件。

:func:`render` 方法是在项目中 `base.py` 中定义的，是
:func:`~pylons.templating.render_mako` 的别名。

内建支持的模板引擎
===================================

Pylons 内建支持的模板渲染引擎有 `Mako`__, `Genshi`__ 和 `Jinja2`__ 。
它们会在新项目创建时自动配置好，也可以随时手动添加并配置。

.. __: http://www.makotemplates.org/
.. __: http://genshi.edgewall.org/
.. __: http://jinja.pocoo.org/


******************************
向模板引擎传递变量
******************************

Pylons 在控制器中使用 :term:`tmpl_context` 来传递变量到模板
（默认情况是使用 `c` 作为别名）::

    import logging

    from pylons import request, response, session, tmpl_context as c, url
    from pylons.controllers.util import abort, redirect

    from helloworld.lib.base import BaseController, render

    log = logging.getLogger(__name__)
    
    class HelloController(BaseController):

        def index(self):
            c.name = "Fred Smith"
            return render('/sample.mako')

在模板中使用变量：

.. code-block:: html+mako
    
    Hi there ${c.name}!

tmpl_context 模板上下文对象的严格模式和非严格模式
=============================================

`tmpl_context` 对象是在每次请求时候被创建的。它是
:class:`~pylons.util.AttribSafeContextObj` 的子类，也是属性读取安全的。
这意味着当尝试获取一个 **不** 存在的属性时，它不会抛出异常
:exec:`AttribSafeContextObj` ，而是返回一个空字符串。

这种默认设置，可以为开发者带来很多方便：

.. code-block:: html+mako
    
    Hi there ${c.name}

当 `c.name` 不存在时候，上述代码依然可以工作，这种写法比使用
严格的上下文内容对象 :class:`~pylons.util.ContextObj` 代码要精简很多。

如果想切换到严格版本的 :term:`tmpl_context` ，在 :file:`config/environment.py`
中（在 config.init_app）中加入::

    config['pylons.strict_c'] = True


.. _template-globals:

**************************
默认模板参数
**************************

默认状态下，所有的模板页都自带一些通用对象来帮助开发。完整的变量列表如下：

- :term:`c` -- 模板页上下文对象 (是 :term:`tmpl_context` 的别名)
- :term:`tmpl_context` -- 模板页上下文对象
- :data:`config` -- Pylons :class:`~pylons.configuration.PylonsConfig`
  对象 (字典结构)
- :term:`g` -- 项目全局对象 (:term:`app_globals` 的别名)
- :term:`app_globals` -- 项目全局对象
- :term:`h` -- 项目助手模块
- :data:`request` -- Pylons :class:`~pylons.controllers.util.Request`
  当前请求对象
- :data:`response` -- Pylons :class:`~pylons.controllers.util.Response`
  当前响应对象
- :class:`session` -- Pylons session 对象 (除非 Session 被禁用了)
- :class:`translator` -- 从配置文件获取当前 i18 对象
- :func:`ungettext` -- gettext 的 ngettext Unicode 版本 (用来处理多语言的复数)
- :func:`_` -- Unicode 版本的获取多语言函数
- :func:`N_` -- gettext no-op function to mark a string for
  translation, but doesn't actually translate
- :class:`url <routes.util.URLGenerator>` -- :class:`routes.util.URLGenerator`
  的实例


****************************
配置模板引擎
****************************

新的 Pylons 项目在项目文件 :file:`config/environment.py` 中配置模板引擎。
在这里面会创建 Mako 模板查找器，并将其载入 :term:`app_globals` 对象，
这样就可以使用模板的渲染方法了。

.. code-block:: python

    # these imports are at the top
    from mako.lookup import TemplateLookup
    from pylons.error import handle_mako_error
    
    # this section is inside the load_environment function
    # Create the Mako TemplateLookup, with the default auto-escaping
    config['pylons.app_globals'].mako_lookup = TemplateLookup(
        directories=paths['templates'],
        error_handler=handle_mako_error,
        module_directory=os.path.join(app_conf['cache_dir'], 'templates'),
        input_encoding='utf-8', default_filters=['escape'],
        imports=['from webhelpers.html import escape'])


使用多个模板引擎
===============================

Since template engines are configured in the :file:`config/environment.py` section, then used by render functions, it's trivial to setup additional template engines, or even differently configured versions of a single template engine. However, custom render functions will frequently be needed to utilize the additional template engine objects.

Example of additional Mako template loader for a different templates directory for admins, which falls back to the normal templates directory::
    
    # Add the additional path for the admin template
    paths = dict(root=root,
                 controllers=os.path.join(root, 'controllers'),
                 static_files=os.path.join(root, 'public'),
                 templates=[os.path.join(root, 'templates')],
                 admintemplates=[os.path.join(root, 'admintemplates'),
                                 os.path.join(root, 'templates')])
    
    config['pylons.app_globals'].mako_admin_lookup = TemplateLookup(
        directories=paths['admin_templates'],
        error_handler=handle_mako_error,
        module_directory=os.path.join(app_conf['cache_dir'], 'admintemplates'),
        input_encoding='utf-8', default_filters=['escape'],
        imports=['from webhelpers.html import escape'])

That adds the additional template lookup instance, next a :ref:`custom render function <custom-render>` is needed that utilizes it::
    
    from pylons.templating import cached_template, pylons_globals
    
    def render_mako_admin(template_name, extra_vars=None, cache_key=None, 
                          cache_type=None, cache_expire=None):
        # Create a render callable for the cache function
        def render_template():
            # Pull in extra vars if needed
            globs = extra_vars or {}

            # Second, get the globals
            globs.update(pylons_globals())

            # Grab a template reference
            template = globs['app_globals'].mako_admin_lookup.get_template(template_name)

            return template.render(**globs)

        return cached_template(template_name, render_template, cache_key=cache_key,
                               cache_type=cache_type, cache_expire=cache_expire)

The only change from the :func:`~pylons.templating.render_mako` function that comes with Pylons is to use the `mako_admin_lookup` rather than the `mako_lookup` that is used by default.


.. _custom-render:

*******************************
自定义 :func:`render` 函数
*******************************

Writing custom render functions can be used to access specific features in a template engine, such as Genshi, that go beyond the default :func:`~pylons.templating.render_genshi` functionality or to add support for additional template engines.

Two helper functions for use with the render function are provided to make it easier to include the common Pylons globals that are useful in a template in addition to enabling easy use of cache capabilities. The :func:`pylons_globals` and :func:`cached_template` functions can be used if desired.

Generally, the custom render function should reside in the project's
``lib/`` directory, probably in :file:`base.py`.

Here's a sample Genshi render function as it would look in a project's
``lib/base.py`` that doesn't fully render the result to a string, and
rather than use :data:`c` assumes that a dict is passed in to be used
in the templates global namespace. It also returns a Genshi stream
instead the rendered string.

.. code-block:: python
    
    from pylons.templating import pylons_globals
    
    def render(template_name, tmpl_vars):
        # First, get the globals
        globs = pylons_globals()

        # Update the passed in vars with the globals
        tmpl_vars.update(globs)
        
        # Grab a template reference
        template = globs['app_globals'].genshi_loader.load(template_name)
        
        # Render the template
        return template.generate(**tmpl_vars)

Using the :func:`~pylons.templating.pylons_globals` function also makes it easy to get to the :term:`app_globals` object which is where the template engine was attached in :file:`config/environment.py`.

.. versionchanged:: 0.9.7
    Prior to 0.9.7, all templating was handled through a layer called 'Buffet'. This layer frequently made customization of the template engine difficult as any customization required additional plugin modules being installed. Pylons 0.9.7 now deprecates use of the Buffet plug-in layer.

.. seealso::
    :mod:`pylons.templating` - Pylons templating API


********************
使用 Mako 模板引擎
********************

介绍
============

这个模板库是用来处理 *视图* 的，他会生成 (X)HTML、CSS 和 Javascript 代码。
*（在这节案例中，项目根目录是 ``myapp`` 。）*

静态还是动态？
------------------

需要动态生成的模板保存在 `myapp/templates` 里，
静态文件存放在 `myapp/public` 里。

服务器运行后，他们都是相对于根目录的， **如果名称冲突，静态文件会优被使用** 。

使用结构化的模板
===========================

创建公共基础模板
----------------------

在 `myapp/templates` 下创建名为 `base.mako` 的文件并按如下编辑：

.. code-block:: html+mako

    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
    "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
    <html>
      <head>
        ${self.head_tags()}
      </head>
      <body>
        ${self.body()}
      </body>
    </html>

一个公共基础模板会被所有页面是使用，这样可以给应用提供一个统一的界面。


* `${...}` 中的表达式将被执行并返回文本结果
* `${` 和 `}` may span several lines but the closing brace should not be on a line by itself (or Mako throws an error)
* `self` 下面的方法是在 Mako 模板中定义的方法

创建子模板
----------------------

在 `myapp/templates` 里创建一个名为 `my_action.mako` 的文件，并编辑：

.. code-block:: html+mako

    <%inherit file="/base.mako" />

    <%def name="head_tags()">
      <!-- add some head tags here -->
    </%def>

    <h1>My Controller</h1>

    <p>Lorem ipsum dolor ...</p>

这个文件中定义了一个教 `base.mako` 的方法。

* `inherit` 指明继承的模板
* Mako 用 `<%def name="function_name()">...</%def>` 定义方法，
  返回的结果将显示出来
* Mako 标签之外的所有内容将被自动输出到 `body()` 方法。

如果所有的页面都继承自同一个文件，那么很容易形成统一的风格。
（在这里这个文件是 `base.mako` ）

检查是否正常工作
-------------------

在控制器的 action 里面，使用 `return` 来返回模板数据。

.. code-block:: python

    return render('/my_action.mako')

现在我们来运行这个 action，通常是访问类似
``http://localhost:5000/my_controller/my_action`` 的网页。
在浏览器点击「查看源码」应该会看到下面这样的输出：

.. code-block:: html

    <!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Transitional//EN"
    "http://www.w3.org/TR/xhtml1/DTD/xhtml1-transitional.dtd">
    <html>
      <head>
      <!-- add some head tags here -->
      </head>
      <body>

    <h1>My Controller</h1>

    <p>Lorem ipsum dolor ...</p>

      </body>
    </html>

.. seealso::

    The `Mako documentation <http://www.makotemplates.org/docs/>`_
        Reasonably straightforward to follow

    See the :ref:`i18n` 
        Provides more help on making your application more worldly.

