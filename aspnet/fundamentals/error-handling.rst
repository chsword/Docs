:version: 1.0.0-rc1

异常处理
==============

By `Steve Smith`_ 

在你的ASP.NET应用程序发生错误时，你可以用多种方法来处理它们，此文章即描述此事。

.. contents:: 本文章节
	:local:
	:depth: 1

`查看下载示例代码 <https://github.com/aspnet/Docs/tree/master/aspnet/fundamentals/error-handling/sample>`_

配置一个异常处理页面
--------------------------------------

你可以在 ``Startup`` 类中的 ``Configure()`` 方法中为每个请求配置管道（pipeline） (更多请参阅 :doc:`startup`)。 你可以轻易地添加一个简单的仅开发时可见的异常页面。 所必须的是在项目中添加 ``Microsoft.AspNetCore.Diagnostics`` 的依赖，然后在 ``Startup.cs`` 类中的 ``Configure()`` 中添加一行代码:

.. literalinclude:: error-handling/sample/src/ErrorHandlingSample/Startup.cs
	:language: c#
	:lines: 21-29
	:dedent: 8
	:emphasize-lines: 6,8

上述代码在调用 ``UseDeveloperExceptionPage`` 之前有一个判断当前环境是否为开发环境的判断。如果你想在开发环境显示详细的异常信息，而在生产环境显示友好的提示的话，这是一个比较好的方式。  :doc:`Learn more about configuring environments <environments>`.

示例应用程序包含了一个产生异常的简单机制：

.. literalinclude:: error-handling/sample/src/ErrorHandlingSample/Startup.cs
	:language: c#
	:lines: 58-77
	:dedent: 8
	:emphasize-lines: 5-8

如果请求的 QueryString 中存在一个非空的 Key 为 ``throw`` 的变量的话 (例如 URL 为 ``/?throw=true``)，这样就会触发一个异常。如果当前运行环境是开发环境： ``Development``, 那么异常页面将如下图所示:

.. image:: error-handling/_static/developer-exception-page.png

当支行环境是非开发环境 , 则可以使用 ``UseExceptionHandler`` 中间件来处理异常:

.. code-block:: c#

  app.UseExceptionHandler("/Error");

使用开发模式异常处理页面
----------------------------------

开发模式异常处理页面将在 Web 处理的管道发生未捕获的异常时显示，其中包含了很多有用的诊断信息。其中包含了一些标签页，包含请求信息和导致异常的信息，第一个标签页包含了异常的堆栈信息：

.. image:: error-handling/_static/developer-exception-page.png

第二个标签页显示了 QueryString 参数信息：

.. image:: error-handling/_static/developer-exception-page-query.png

在这里你可以看到本例中 QueryString 的参数 ``throw`` 。当前请求未包含 Cookies，如果请求中存在 Cookies，你可以在 Cookies 标签页中查看。如果想要查看请求的 Headers ，则可以在 Headers 标签页中查看。

.. image:: error-handling/_static/developer-exception-page-headers.png

.. _status-code-pages:

配置 Status Code 页面
-----------------------------

默认情况下，应用程序都没有提供一个比较好的 HTTP 状态码错误页面，例如 500 (Internal Server Error) 或 404 (Not Found) 的错误显示页面。在这里，可以通过在 ``Configure`` 方法中添加 ``StatusCodePagesMiddleware`` 来进行定义:

.. code-block:: c#

  app.UseStatusCodePages();

默认情况下添加中间件是很容易的，可以写一个纯文本的 status code 处理程序,例如像下面这个 404 Not Found 的状态码页面:

.. image:: error-handling/_static/default-404-status-code.png

中间件支持数种不同的扩展方法 ，你可以通过自定义的 lambda 表达式进行传参：

.. code-block:: c#

  app.UseStatusCodePages(context => 
    context.HttpContext.Response.SendAsync("Handler, status code: " +
    context.HttpContext.Response.StatusCode, "text/plain"));

当然，你可以使用另一种更简单的方式直接使用 content type 和一个 format string 来进行传值:

.. code-block:: c#

  app.UseStatusCodePages("text/plain", "Response, status code: {0}");

中间件也可以处理重定向 (支持绝对、相对 URL 路径), 你可以通过下列这种方式来传递 status code:

.. code-block:: c#

  app.UseStatusCodePagesWithRedirects("~/errors/{0}");

在上面的例子中，浏览器中将会看到一个 ``302 / Found`` 的状态，并且将会重定向到对应的 URL 页面。

Alternately, the middleware can re-execute the request from a new path format string:

.. code-block:: c#

  app.UseStatusCodePagesWithReExecute("/errors/{0}");

The ``UseStatusCodePagesWithReExecute`` method will still return the original status code to the browser, but will also execute the handler given at the path specified.

如果需要对某些请求禁用 status code 页面的话, 你可以使用以下代码:

.. code-block:: c#

  var statusCodePagesFeature = context.Features.Get<IStatusCodePagesFeature>();
  if (statusCodePagesFeature != null)
  {
    statusCodePagesFeature.Enabled = false;
  }

在客户端-服务器交互过程中异常处理的限制
------------------------------------------------------------------

Web apps have certain limitations to their exception handling capabilities because of the nature of disconnected HTTP requests and responses. Keep these in mind as you design your app's exception handling behavior.

#. Once the headers for a response have been sent, you cannot change the response's status code, nor can any exception pages or handlers run. The response must be completed or the connection aborted.
#. If the client disconnects mid-response, you cannot send them the rest of the content of that response.
#. There is always the possibility of an exception occuring one layer below your exception handling layer.
#. Don't forget, exception handling pages can have exceptions, too. It's often a good idea for production error pages to consist of purely static content.

Following the above recommendations will help ensure your app remains responsive and is able to gracefully handle exceptions that may occur.

服务器端的异常处理
-------------------------

In addition to the exception handling logic in your app, the server hosting your app will perform some exception handling. If the server catches an exception before the headers have been sent it will send a 500 Internal Server Error response with no body. If it catches an exception after the headers have been sent it must close the connection. Requests that are not handled by your app will be handled by the server, and any exception that occurs will be handled by the server's exception handling. Any custom error pages or exception handling middleware or filters you have configured for your app will not affect this behavior.

.. _startup-error-handling:

Startup 中的异常处理
--------------------------

One of the trickiest places to handle exceptions in your app is during its startup. Only the hosting layer can handle exceptions that take place during app startup. Exceptions that occur in your app's startup can also impact server behavior. For example, to enable SSL in Kestrel, one must configure the server with ``KestrelServerOptions.UseHttps()``. If an exception happens before this line in ``Startup``, then by default hosting will catch the exception, start the server, and display an error page on the non-SSL port. If an exception happens after that line executes, then the error page will be served over HTTPS instead.

ASP.NET MVC 异常处理
--------------------------

:doc:`MVC </mvc/index>` apps have some additional options when it comes to handling errors, such as configuring exception filters and performing model validation.

异常过滤器
^^^^^^^^^^^^^^^^^

Exception filters can be configured globally or on a per-controller or per-action basis in an :doc:`MVC </mvc/index>` app. These filters handle any unhandled exception that occurs during the execution of a controller action or another filter, and are not called otherwise. Exception filters are detailed in :doc:`filters </mvc/controllers/filters>`.

.. tip:: Exception filters are good for trapping exceptions that occur within MVC actions, but they're not as flexible as error handling middleware. Prefer middleware for the general case, and use filters only where you need to do error handling *differently* based on which MVC action was chosen.

处理 Model State 的异常错误
^^^^^^^^^^^^^^^^^^^^^^^^^^^

:doc:`Model validation </mvc/models/validation>` occurs prior to each controller action being invoked, and it is the action method’s responsibility to inspect ``ModelState.IsValid`` and react appropriately. In many cases, the appropriate reaction is to return some kind of error response, ideally detailing the reason why model validation failed. 

Some apps will choose to follow a standard convention for dealing with model validation errors, in which case a :doc:`filter </mvc/controllers/filters>` may be an appropriate place to implement such a policy. You should test how your actions behave with valid and invalid model states (learn more about :doc:`testing controller logic </mvc/controllers/testing>`).
