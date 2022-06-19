[TOC]

- Servlet Container
- Servlets
	- Servlet lifecycle
	- ServletConfig
	- ServletConfig
	- ServletContext
	- ServletResponse, ServletRequest
	- Servlet attribute scopes
	- HttpSession & Cookies
	- RequestDispatcher
- Filters
- Listeners
- Async Servlets
- ServletContainerInitializer & web.xml


### Servlet Container

- **Communication Support**: container provides easy way of communication between web client (Browsers) and the servlets/JSPs. Because of container, we don’t need to build a server socket to listen for any request from web client, parse the request and generate response.
- **Lifecycle and Resource Management**
- **Multithreading Support**: container creates new thread for every request to the servlet and provide them request and response objects to process. So servlets are not initialized for each request and saves time and memory.
- **JSP Support**: every JSP in the application is compiled by container and converted to Servlet and then container manages them like other servlets.
- **Miscellaneous Task**: Servlet container manages the resource pool, perform memory optimizations, execute garbage collector, provides security configurations, support for multiple applications, hot deployment and several other tasks behind the scene that makes a developer life easier.


### Servlets

***Servlet*** is an interface defining what a servlet must implement.

- ***GenericServlet*** is just that, a generic, protocol-independent servlet (only with *service()* method)
- ***HttpServlet*** is a servlet tied specifically to the HTTP protocol (*doGet(), doPost()*, etc)
- ***FacesServlet*** is a servlet that manages JavaServer Faces

To declare a servlet we can use `@WebServlet` annotation or
```xml
<servlet>
  <servlet-name>cms</servlet-name>
  <servlet-class>com.SomeServlet</servlet-class>
  <init-param>
    <param-name>debug</param-name>
    <param-value>true</param-value>
  </init-param>
  <load-on-startup>5</load-on-startup> // priority to load
</servlet>
```


#### Servlet lifecycle

1. load servlet class
	- loaded on first request or can be loaded on the container startup with `<load-on-startup>1</load-on-startup>`)
2. create instance of servlet
	- typically, only a single isntance of the servlet is created that serves all concurrent requests
3. call `init(servletConfig)` method
4. call `service(req, res)` method
	- to forward the request to corresponding HTTP method: *doGet(), doPost(), doHead(), doPut(), doDelete(), doOptions(), doTrace()*
5. call `destroy()` method
	- when the container shuts down or the application shuts down
    - when the container decides that there is a shortage of memory
    - when this servlet hasn't got a request in a long time


#### ServletConfig

Used to pass configuration information to servlet. Every servlet has it’s own **ServletConfig** object and servlet container is responsible for instantiating this object.
We can provide servlet init parameters via `<init-param>` or through use of `@WebInitParam` annotation.


#### ServletContext

`ServletContext context = request.getSession().getServletContext();`

Contains meta information about the web application. For instance, you can access context parameters set in the `web.xml` file via `<context-param>`, store application wide parameters (are available to all servlets in your application, and between requests and sessions).

In case of an architecture with 2+ web servers in a cluster, keep in mind that values stored in the ServletContext of one server, may not be available in the ServletContext on the other server.

In Servlet 3.0 it posible programmatically add servlets, listeners and filters: `addFilter()`, `addListener()`, `addServlet()`.

```xml
<context-param>
  <param-name>name</param-name>
  <param-value>value</param-value>
</context-param>
```


#### ServletResponse, ServletRequest

- Writting text `PrintWriter writer = response.getWriter();`
- Writting binary `ServletOutputStream outputStream = response.getOutputStream();`
- Redirecting `response.sendRedirect("https://google.com");`

We can’t get instances of both **PrintWriter** and **ServletOutputStream** in a single servlet method

**HttpServletResponse** extends the **ServletResponse** interface to provide HTTP-specific functionality in sending a response. For example, it has methods to access HTTP headers and cookies.
**HttpServletRequest** extends the **ServletRequest** interface to provide request information for HTTP servlets (service methods (doGet, doPost, etc)).

**HttpServletRequestWrapper** and **HttpServletResponseWrapper** are wrapper classes that provided to help developers with custom implementation of servlet request and response types.

#### Servlet attribute scopes

- request scope
- session scope
- application scope


#### HttpSession

Represents a user session. A user session contains information about the user across multiple HTTP requests. When a user enters your site for the first time, the user is given a unique ID to identify his session by. This ID is typically stored in a cookie (**jsessionid**) or in a request parameter.

`encodeURL(url)` escape special characters, whitespaces and aslo used to ensure session management is handled properly. It takes a URL in, and if the user has cookies turned off, it attaches the **jsessionid** to the URL in a proper format to be recognized as the session identifier.

```xml
<session-config>
  <session-timeout>120</session-timeout>
</session-config>
```


#### Cookies

```java
response.addCookie(new Cookie("myCookie", "myCookieValue"));
...
Cookie[] cookies = request.getCookies();
```

To remove a cookie set the expiration time to 0 (will be removed immediately) or -1 (will be deleted when the browser shuts down).


#### RequestDispatcher

`RequestDispatcher dispatch = request.getRequestDispatcher("/someurl");`

Enables your servlet to "call" another servlet from inside another servlet. The other servlet is called as if an HTTP request was sent to it by a browser.

- `forward(req, res)` transfers the control to the next servlet/jsp you are calling, meaning after the response of the calling servlet has been committed
- `include(req, res)` retains the control with the current servlet, it just includes the processing done by the calling servlet/jsp


### Filters

- `init(filterConfig)`
- `destroy()`
- `doFilter(req, res, chain)`
	- to forward request `chain.doFilter(req, res);`, if this method is not called, the request is not forwarded, but just blocked. This is a great example of **Chain of Responsibility Pattern**.

The supported values for dispatcher are:

- **REQUEST** The request comes directly from the client. Default value.
- **FORWARD** The request is being processed under a request dispatcher  using a *forward()* call.
- **INCLUDE** The request is being processed under a request dispatcher using an *include()* call.
- **ERROR** The request that is being processed with the error handling  mechanism.
- **ASYNC** The request is being processed with the async context dispatch mechanism.

To declare a filter we can use `@WebFilter` annotation or
```xml
<filter>
  <filter-name>image</filter-name>
  <filter-class>com.MyImageFilter</filter-class>
  <init-param>
    <param-name>name</param-name>
    <param-value>value</param-value>
  </init-param>
</filter>
<filter-mapping>
  <filter-name>image</filter-name>
  <url-pattern>/images/*</url-pattern>
</filter-mapping>
```


### Listeners

Servlet Listener is used for listening to events in a web application, such as when you create a session, or place an attribute in an session or if you passivate and activate in another container.

There are two levels of servlet events:

- Servlet context-level (application-level) event
	- this event involves resources or state held at the level of the application servlet context object.
- Session-level event
	- this event involves resources or state associated with the series of requests from a single user session; that is, associated with the HTTP session object.

Each of these two levels has two event categories:

- Lifecycle changes
- Attribute changes


- Listeners
	- `ServletContextListener` – Interface for receiving notification events about `ServletContext` lifecycle changes.
	- `ServletContextAttributeListener` – Interface for receiving notification events about `ServletContext` attribute changes.
	- `ServletRequestListener` – Interface for receiving notification events about requests coming into and going out of scope of a web application.
	- `ServletRequestAttributeListener` – Interface for receiving notification events about `ServletRequest` attribute changes.
	- `HttpSessionListener` – Interface for receiving notification events about `HttpSession` lifecycle changes.
	- `HttpSessionBindingListener` – Causes an object to be notified when it is bound to or unbound from a session.
- Events
	- `HttpSessionEvent` – This is the class representing event notifications for changes to sessions within a web application.
	- `ServletContextAttributeEvent` – Event class for notifications about changes to the attributes of the `ServletContext` of a web application.
	- `ServletContextEvent` – This is the event class for notifications about changes to the servlet context of a web application.
	- `ServletRequestEvent` – Events of this kind indicate lifecycle events for a `ServletRequest`. The source of the event is the `ServletContext` of this web application.

To declare a listener we can use `@WebListener` annotation or
```xml
<listener>
  <listener-class>com.AppContextListener</listener-class>
</listener>
```


### Async Servlets

- First of all the servlet should have `@WebServlet(asyncSupported = true)`
- Since the actual work is to be delegated to another thread, we should have a **thread pool** implementation. We can create thread pool using Executors framework and use servlet context listener to initiate the thread pool.
- We need to get instance of **AsyncContext** through `ServletRequest.startAsync()` method which. AsyncContext provides methods to get the **ServletRequest** and **ServletResponse** object references. It also provides method to forward the request to another resource using `dispatch()` method.


### ServletContainerInitializer

Introduced in Servlet 3.0. Implementors of this interface are notified during the context startup phase and can perform any programatic registration through the provided **ServletContext**, rather than adding this configuration through a `web.xml` file.



### web.xml

The web.xml file is the deployment descriptor for a Servlet-based Java web application.

```xml
<login-config>
  <auth-method>BASIC</auth-method>
  <realm-name>Editor Login</realm-name>
</login-config>

<login-config>
  <auth-method>FORM</auth-method>
  <form-login-config>
    <form-login-page>/login.jsp</form-login-page>
    <form-error-page>/error.jsp</form-error-page>
  </form-login-config>
</login-config>

<security-constraint>
  <web-resource-collection>
    <web-resource-name>Protected Area</web-resource-name>
    <url-pattern>/private/*</url-pattern>

    <http-method>DELETE</http-method>
    <http-method>GET</http-method>
    <http-method>POST</http-method>
    <http-method>PUT</http-method>
  </web-resource-collection>
  <auth-constraint>
    <role-name>admin</role-name>
    <role-name>superuser</role-name>
  </auth-constraint>
</security-constraint>

<error-page>
  <error-code>404</error-code>
  <location>/404.jsp</location>
</error-page>
<error-page>
  <exception-type>java.lang.Throwable</exception-type>
  <location>/throwable.jsp</location>
</error-page>

<mime-mapping>
  <extension>.txt</extension>
  <mime-type>text/html</mime-type>
</mime-mapping>
```
