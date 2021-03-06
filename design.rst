.. _design:

Flask 中的设计决策
=========================

如果你好奇 Flask 为什么用它的方式做事情，而不是别的方法，那么这节是为你准
备的。这节应该给你一些设计决策的想法，也许起初是武断且令人惊讶的，特别是
直接与其它框架相比较。


显式的应用对象
-------------------------------

一个基于 WSGI 的 Python web 应用必须有一个中央的可调用对象来实现实际的应
用。在 Flask 中，这是一个 :class:`~flask.Flask` 类的实例。每个 Flask 应用
必须创建一个该类的实例，并传给它模块的名称，但是为什么 Flask 不自己这么
做？

当下面的代码不使用这样一个显式的应用对象时::

    from flask import Flask
    app = Flask(__name__)

    @app.route('/')
    def index():
        return 'Hello World!'

看起来会是这样::

    from hypothetical_flask import route

    @route('/')
    def index():
        return 'Hello World!'

这样做有三个主要的原因。最重要的一个事，显式的应用对象需要在同一时刻只存在
一个实例。有许多方法来用单个应用对象来仿造多个应用，像维护一个应用的栈一样，
但这导致一些问题，这里不会赘述。现在问题是：什么时候一个为框架在同一时刻需
要至少一个应用？一个很好的例子是单元测试。当你想要测试什么的时候，创建一个
最小化的应用来测试特定的行为非常有用。当应用对象删除时，它分配的一切都会被
再次释放。

当你的代码中有一个显式的对象时，继承基类（ :class:`~flask.Flask` ）来更改
特定行为将成为可能。如果基于一个不暴露给你的类的对象在你之前创建，这么做只
能通过 hack 。

此外， Flask 依赖于一个那个类的显式实例还有一个非常重要的原因是：包名称。无
论何时你创建一个 Flask 实例，你通常传给它 `__name__` 作为包名。 Flask 依赖
这个信息来妥善地加载相对于你模块的资源。在 Python 对反射的杰出支持下，它可
以访问包来找出模板和静态文件存储在哪（见 :meth:`~flask.Flask.open_resource`
）。当前显然有许多框架不需要任何配置，且能载入相对于你应用的模块的模板。但
是它们需要为此使用当前工作目录，一种非常不值得信赖的决定应用在哪的方式。当
前工作目录是进程间的，而且如果你想要在同一个进程中运行多个应用（这会在你不
知道的一个 web 服务器中发生），路径会断开。更可怕的是：许多 web 服务器不把
你应用的目录，而是文档根目录设定为工作目录，但两者不一定是一个文件夹。

第三个原因是“显明胜于隐含”。那个对象是你的 WSGI 应用，你不需要记住别东西。
如果你想要应用一个 WSGI 中间件，只需要封装它（虽然有更好的方式来这么做来不
丢失应用对象的引用 :meth:`~flask.Flask.wsgi_app` ）。

此外，这个设计使得用工厂函数来创建应用成为可能，这对单元测试和类似的东西
（ :ref:`app-factories` ）十分有用。


路由系统
------------------

Flask 使用 Werkzeug 路由系统，其被设计为按复杂度为路由排序。这意味着，你可
以任意顺序声明路由，而且他们仍会按期望工作。这在你想妥善地实现基于路由的装
饰器，自从当应用被分割为多个模块时装饰器可以以未定义的顺序调用时，是必要的。

另一个 Werkzeug 路由系统的设计决策是， Werkzeug 中的路由试图确保 URL 是唯
一的。 Werkzeug 对此会做的足够多，因为它在路由不明确时自动重定向到一个规
范的 URL 。


某个模板引擎
-------------------

Flask 在模板引擎上做了决定： Jinja2 。为什么 Flask 没有一个即插的模板引擎
接口？显然，你可以使用一个不同的模板引擎，但是 Flask 仍然会为你配置
Jinja2 。虽然 Jinja2 *总是* 配置的限制可能会消失，但绑定一个模板引擎并使用
的决策不会。

模板引擎与编程语言类似，每个模板引擎都有特定的理解事物工作的方式。表面上，
它们以相同方式工作：你给引擎一个变量的集合让它为模板求值，并返回一个字符
串。

然而，关于相同点的论述结束了。例如 Jinja2 有一个全面的过滤器系统，一个可靠
的模板继承方式，可以从模板内和 Python 代码内使用复用块（宏）的支持，对所有
操作使用 Unicode，支持迭代模板渲染，配置语法等等。其它的引擎，一个类似
Genshi——基于 XML 流求值的引擎，模板继承要考虑 XPath 可用性等等。而 Mako 像
对待 Python 模块一样处理模板。

当把一个模板引擎跟一个应用或框架联系到一起，就不只是渲染模板了。比如，
Flask 使用 Jinja2 全面的自动转义支持。同样，也提供了从 Jinja2 模板中
访问宏的途径。

不去掉模板引擎的独特特性的模板抽象层是一门对自身的科学，也是像 Flask
的微框架的巨大事业。

此外，扩展也可以简易地依赖于一个现有的模板语言。你可以简单地使用你自己的
模板语言，而扩展会始终依赖于 Jinja 本身。


微与依赖
-----------------------

为什么 Flask 把自己叫做微框架，并且它依赖于两个库（也就是 Werkzeug 和
Jinja 2）。为什么不能？如果我们仔细审查 Ruby 的 web 开发，有一个非常
类似 WSGI 的协议。被称作 Rack 的就是它，但是除此之外，它看起来非常像
一个 WSGI 的 Ruby 实现。但是几乎所有的 Ruby 应用不直接使用 Rack ，而是
基于一个相同名字的库。这个 Rack 库与 Python 中的两个库不相伯仲： WebOb
（以前叫 Paste ） 和 Werkzeug。 Paste 依然在使用，但是从我的理解，它有
些过时，而赞同 WebOb 。 WebOb 和 Werzeug 的开发是一起开始的，也有着同样
的理念：为其它应用的利用做一个 WSGI 的良好实现。

Flask 是一个受益于 Werkzeug 妥善实现 WSGI 接口（有时是一个复杂的任务）
既得成果的框架。感谢 Python 包基础建设中近期的开发，包依赖不再是问题，
并且只有很少的原因反对依赖其它库的库。


线程局域变量
-------------

Flask 为请求、会话和一个额外对象（你可以在 :data:`~flask.g` 上放置自己的东
西）使用线程局域对象（实际上是上下文局域对象，它们也支持 greenlet 上下文）。
为什么是这样，这不是一个坏主意吗？

是的，通常情况下使用线程局域变量不是一个明智的主意。它们在不基于线程概念的
服务器上会导致问题，并且使得大型应用难以维护。但 Flask 不仅为大型应用或异步
服务器设计。 Flask 想要使得编写一个传统 web 应用的过程快速而简单。

一些关于基于 Flask 大型应用的灵感，见文档的 :ref:`becomingbig` 一节。


Flask 是什么，不是什么？
--------------------------------

Flask 永远不会包含数据库层，也不会有表单库或是这个方向的其它东西。 Flask
只建立 Werkezug 和 Jinja2 的桥梁，前者实现一个合适的 WSGI 应用，后者处理
模板。 Flask 也绑定了一些通用的标准库包，比如 logging 。其它所有一切取决
于扩展。

为什么是这样？众口难调，因此 Flask 不强制把特异的偏好和需求囊括在核心里。
大多数 web 应用都可以说需要一个模板引擎，而并不是每个应用都需要一个 SQL
数据库。

Flask 的思想是为所有应用建立一个良好的基础，其余的一切都取决于你和扩展。

 of threads and make
large applications harder to maintain.  However Flask is just not designed
for large applications or asynchronous servers.  Flask wants to make it
quick and easy to write a traditional web application.

Also see the :ref:`becomingbig` section of the documentation for some
inspiration for larger applications based on Flask.


What Flask is, What Flask is Not
--------------------------------

Flask will never have a database layer.  It will not have a form library
or anything else in that direction.  Flask itself just bridges to Werkzeug
to implement a proper WSGI application and to Jinja2 to handle templating.
It also binds to a few common standard library packages such as logging.
Everything else is up for extensions.

Why is this the case?  Because people have different preferences and
requirements and Flask could not meet those if it would force any of this
into the core.  The majority of web applications will need a template
engine in some sort.  However not every application needs a SQL database.

The idea of Flask is to build a good foundation for all applications.
Everything else is up to you or extensions.
ou or extensions.
