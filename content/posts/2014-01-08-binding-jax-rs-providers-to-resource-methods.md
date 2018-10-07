---
title: Binding JAX-RS Providers to Resource Methods
author: Michal Gajdoš
type: post
date: 2014-01-08T18:35:02+00:00
url: /binding-jax-rs-providers-to-resource-methods/
aliases: /2014/01/08/binding-jax-rs-providers-to-resource-methods/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey

---
When you register providers (such as filters and interceptors) in your application they&#8217;re associated and applied to each resource method by default. In other words they&#8217;re bound globally. For most cases this approach is sufficient but sometimes, especially when you want to use your provider only with a few methods, it&#8217;s not enough. JAX-RS 2.0 brings a solution for this situation: Name and Dynamic Binding.

<!--more-->

## <a name="name-binding"></a>Name Binding

Name Binding is an annotation-driven (_static_) provider binding based on the JAX-RS meta-annotation [@NameBinding][1]. Custom binding annotations annotated with _@NameBinding_ can be later used to decorate resource classes or resource methods as well as JAX-RS providers to define associations between them.

Let&#8217;s assume we have a logging filter (similar to [LoggingFilter][2]) responsible for logging incoming and outgoing communication (headers and/or entity) on our server:

{{< highlight java "hl_lines=2" >}}
@Provider
@Logged
public class LoggingFilter implements ContainerRequestFilter,
                                      ContainerResponseFilter { ... }
{{< / highlight >}}

Decorating this class with _@Logged_ (name binding) annotation we transformed a globally-bound provider to a name-bound one. To bind this provider to a resource method we need to decorate the method with _@Logged_ as well:

{{< highlight java "hl_lines=6" >}}
@Path("helloworld")
public class HelloWorldResource {

    @GET
    @Produces("text/plain")
    @Logged
    public String getHello() {
        return "Hello World!";
    }
}
{{< / highlight >}}

Name binding annotations can be applied to a resource method (as in our case), to a resource class (i.e. _HelloWorldResource_) or to an [Application][3] sub-class (a name-bound JAX-RS provider bound by the annotation will be applied to all (sub-)resource methods).

Finally to create a name binding annotation from _@Logged_ just put _@NameBinding_ on top of it:

{{< highlight java "hl_lines=1" >}}
@NameBinding
@Target({ ElementType.TYPE, ElementType.METHOD })
@Retention(value = RetentionPolicy.RUNTIME)
public @interface Logged { }
{{< / highlight >}}

Assigning multiple name-bound providers to a single resource method is also possible by decorating the method with all required name binding annotations. For example, if you want to use _AuthenticationFilter_:

{{< highlight java "hl_lines=2" >}}
@Provider
@Authenticated
public class AuthenticationFilter implements ContainerRequestFilter { ... }
{{< / highlight >}}

with the previously mentioned _HelloWorldResource.getHello()_ resource method decorate this method with _@Authenticated_ as well. Now both _AuthenticationFilter_ and _LoggingFilter_ will be used when the resource method is being invoked.

## <a name="dynamic-binding"></a>Dynamic Binding

To bind post-matching providers more dynamically a new interface, [DynamicFeature][4], has been introduced in JAX-RS 2.0. By implementing this interface you can assign particular providers programmatically to a subset of available resource methods.

> <a name="dynamic-binding-jersey1"></a>Similar mechanism exists in Jersey 1. [ResourceFilterFactory][5] / [ResourceFilter][6] allows you to create request/response filters for a resource method. These classes has been replaced with `DynamicFeature` in Jersey 2.

The interface contains a callback method providing

  * [ResourceInfo][7] &#8211; information about the resource method.
  * [FeatureContext][8] &#8211; configurable (sub-)resource method-level context.

The callback method is invoked once for every resource method, that exists in an application, to augment the set of filters and entity interceptors bound to this method.

Let&#8217;s take the example from the previous chapter and assign _LoggingFilter_ to all resource methods from package `my.package.admin` that are listening to HTTP GET requests.

{{< highlight java >}}
@Provider
public class MyDynamicFeature implements DynamicFeature {

    @Override
    public void configure(final ResourceInfo resourceInfo,
                          final FeatureContext context) {

        final String resourcePackage = resourceInfo.getResourceClass()
                .getPackage().getName();
        final Method resourceMethod = resourceInfo.getResourceMethod();

        if ("my.package.admin".equals(resourcePackage)
                && resourceMethod.getAnnotation(GET.class) != null) {
            context.register(LoggingFilter.class);
        }
    }
}
{{< / highlight >}}

Now every HTTP GET request sent to a resource from `my.package.admin` package will be logged.

Implementations of _DynamicFeature_ contract are still providers that need to be registered in your application before they can be used. [Registering Resources and Providers in Jersey 2][9] discusses how to do that.

## Further reading

  * [Filters and Interceptors &#8211; Name Binding][10]
  * [Filters and Interceptors &#8211; Dynamic Binding][11]

## References

  * [JAX-RS 2.0 specification][12]

 [1]: https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/NameBinding.html
 [2]: https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/filter/LoggingFilter.html
 [3]: https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Application.html
 [4]: https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/container/DynamicFeature.html
 [5]: https://jersey.github.io/apidocs/1.18/jersey/com/sun/jersey/spi/container/ResourceFilterFactory.html
 [6]: https://jersey.github.io/apidocs/1.18/jersey/com/sun/jersey/spi/container/ResourceFilter.html
 [7]: https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/container/ResourceInfo.html
 [8]: https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/FeatureContext.html
 [9]: /2013/11/19/registering-resources-and-providers-in-jersey-2/
 [10]: https://jersey.github.io/documentation/latest/filters-and-interceptors.html#d0e8137
 [11]: https://jersey.github.io/documentation/latest/filters-and-interceptors.html#d0e8210
 [12]: https://jcp.org/aboutJava/communityprocess/final/jsr339/index.html