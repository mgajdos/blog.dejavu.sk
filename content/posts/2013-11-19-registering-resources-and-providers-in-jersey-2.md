---
title: Registering Resources and Providers in Jersey 2
author: Michal Gajdoš
type: post
date: 2013-11-19T09:35:26+00:00
url: /registering-resources-and-providers-in-jersey-2/
aliases: /2013/11/19/registering-resources-and-providers-in-jersey-2/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey

---
There has been some confusion on [StackOverflow][1] lately about how to register JAX-RS resource classes and custom providers in Jersey 2. The main problem seemed to be finding and using correct properties and methods to configure a JAX-RS/Jersey application. In this article I&#8217;d like to discuss 3 ways how to configure such an application properly.

<!--more-->

> Biggest difference between Jersey 1 and Jersey 2 in this regard is used namespace. Jersey 1 uses prefix `com.sun.jersey` (even for property names) which is not true for the namespace of Jersey 2 that starts with `org.glassfish.jersey`. Besides that Jersey 2 does NOT support properties from Jersey 1 and you need to use specific ones (from Jersey 2) to make things work.

Let&#8217;s assume I am working on a Jersey 2 application and I already have some resource classes and providers. What I want to do is to register my custom <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/container/ContainerRequestFilter.html">ContainerRequestFilter</a> called `SecurityRequestFilter` that is responsible for setting correct <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/SecurityContext.html">SecurityContext</a> for incoming requests. Since I am still developing my application I&#8217;d also like to see and log communication (incl. request/response entity) between client/server and be able to [trace][2] my requests if needed. For logging purposes I can use proprietary filter from Jersey called <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/filter/LoggingFilter.html">LoggingFilter</a>. As mentioned I&#8217;d like to log the entity as well so I need to provide an instance of this filter created using the following constructor: [LoggingFilter][3]. Tracing support can be turned on by setting special property called [ServerProperties.TRACING][4].

> You can find a lot of other useful properties in *Properties classes like:
  
> <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/CommonProperties.html">CommonProperties</a>, <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ServerProperties.html">ServerProperties</a> or <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/ClientProperties.html">ClientProperties</a>.

To register my providers I can use one of these approaches:

## <a name="application"></a>Application (JAX-RS)

<a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Application.html">Application</a> defines the components of a JAX-RS application and supplies additional meta-data. With this approach you&#8217;re supposed to register everything (resource classes, providers, properties) the application will need:

{{< highlight java >}}
@ApplicationPath("/")
public class MyApplication extends ResourceConfig {

    @Override
    public Set<Class<?>> getClasses() {
        final Set<Class<?>> classes = new HashSet<Class<?>>();
        // Register my custom provider.
        classes.add(SecurityRequestFilter.class);
        // Register resources.
        classes.add(...);
        return classes;
    }

    @Override
    public Set<Object> getSingletons() {
        final Set<Object> singletons = new HashSet<Object>();
        // Register an instance of LoggingFilter.
        singletons.add(new LoggingFilter(LOGGER, true));
        return singletons;
    }

    @Override
    public Map<String, Object> getProperties() {
        final Map<String, Object> properties = new HashMap<String, Object>();
        // Enable Tracing support.
        properties.put(ServerProperties.TRACING, "ALL");
        return properties;
    }
}
{{< / highlight >}}

## <a name="resourceconfig"></a>ResourceConfig (Jersey specific)

<a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ResourceConfig.html">ResourceConfig</a> is itself an extension of JAX-RS Application class but it provides some convenience methods to make registering resources and providers more friendly:

{{< highlight java >}}
@ApplicationPath("/")
public class MyApplication extends ResourceConfig {

    public MyApplication() {
        // Register resources and providers using package-scanning.
        packages("my.package");

        // Register my custom provider - not needed if it's in my.package.
        register(SecurityRequestFilter.class);
        // Register an instance of LoggingFilter.
        register(new LoggingFilter(LOGGER, true));

        // Enable Tracing support.
        property(ServerProperties.TRACING, "ALL");
    }
}
{{< / highlight >}}

## <a name="webxml"></a>web.xml (JAX-RS & Jersey specific)

Web application descriptor allows you to define JAX-RS Application (JAX-RS spec) for non Servlet 3.0 apps (you can use it with Servlet 3.0 as well but in Servlet 3.0 it&#8217;s enough to have an Application subclass on your class-path) or define `contex-param`/`init-param` for package-scanning, properties, &#8230;

{{< highlight xml >}}
<web-app version="2.5"
        xmlns="http://java.sun.com/xml/ns/javaee"
        xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
        xsi:schemaLocation="http://java.sun.com/xml/ns/javaee http://java.sun.com/xml/ns/javaee/web-app_2_5.xsd">

    <servlet>
        <servlet-name>my.package.MyApplication</servlet-name>
        <servlet-class>org.glassfish.jersey.servlet.ServletContainer</servlet-class>

        <!-- Register JAX-RS Application, if needed. -->
        <init-param>
            <param-name>javax.ws.rs.Application</param-name>
            <param-value>my.package.MyApplication</param-value>
        </init-param>

        <!-- Register resources and providers under my.package. -->
        <init-param>
            <param-name>jersey.config.server.provider.packages</param-name>
            <param-value>my.package</param-value>
        </init-param>

        <!-- Register my custom provider (not needed if it's in my.package) AND LoggingFilter. -->
        <init-param>
            <param-name>jersey.config.server.provider.classnames</param-name>
            <param-value>my.package.SecurityRequestFilter;org.glassfish.jersey.filter.LoggingFilter</param-value>
        </init-param>

        <!-- Enable Tracing support. -->
        <init-param>
            <param-name>jersey.config.server.tracing</param-name>
            <param-value>ALL</param-value>
        </init-param>

        <load-on-startup>1</load-on-startup>
    </servlet>

    <servlet-mapping>
        <servlet-name>my.package.MyApplication</servlet-name>
        <url-pattern>/*</url-pattern>
    </servlet-mapping>
</web-app>
{{< / highlight >}}

## Further reading

  * [Deploying a RESTful Web Service][5]
  * [JAX-RS Application, Resources and Sub-Resources][6]

 [1]: http://stackoverflow.com/
 [2]: https://jersey.github.io/documentation/latest/monitoring_tracing.html#tracing
 [3]: https://jersey.github.io/apidocs/2.6/jersey/org/glassfish/jersey/filter/LoggingFilter.html#LoggingFilter(java.util.logging.Logger,%20boolean)
 [4]: https://jersey.github.io/apidocs/2.6/jersey/org/glassfish/jersey/server/ServerProperties.html#TRACING
 [5]: https://jersey.github.io/documentation/latest/user-guide.html#deployment
 [6]: https://jersey.github.io/documentation/latest/jaxrs-resources.html