---
title: 'Jersey 2.6 has been Released: New and Noteworthy'
author: Michal Gajdoš
type: post
date: 2014-02-21T09:35:10+00:00
expirydate: 2018-09-01T00:00:00+00:00
url: /jersey-2-6-has-been-released-new-and-noteworthy/
aliases: /2014/02/21/jersey-2-6-has-been-released-new-and-noteworthy/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - release

---
As some of you may have already noticed, we&#8217;ve recently released <a href="https://jersey.java.net/">Jersey</a> 2.6 (Reference Implementation of JAX-RS &#8211; Java API for RESTful Web Services). In this article I&#8217;d like to introduce some of the new features, important bug fixes and other noteworthy changes.

<!--more-->

> Jersey 2.6.x is the latest version of Jersey targeted to JDK 6. Refer to [Latest Jersey Versions][1] to find out the latest version that supports JDK 7/8.

## Injections into Client Provider Instances

In some cases you might need to inject some custom types into your client provider instances. JAX-RS types do not need to be injected as they are passed as arguments into API methods. Injections into client providers (filters, interceptor) are possible as long as the provider is registered as a class. If the provider is registered as an instance then runtime will not inject the provider. The reason is that this provider instance might be registered into multiple client configurations. For example one instance of <a href="https://jersey.github.io/apidocs/latest/jersey/javax/ws/rs/client/ClientRequestFilter.html">ClientRequestFilter</a> can be registered to two <a href="https://jersey.github.io/apidocs/latest/jersey/javax/ws/rs/client/Client.html">Client</a>s.

To solve injection of a custom type into a client provider instance use <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/ServiceLocatorClientProvider.html">ServiceLocatorClientProvider</a> to extract <a href="https://hk2.java.net/apidocs/org/glassfish/hk2/api/ServiceLocator.html">ServiceLocator</a> which can return the required injection. The following example shows how to utilize _ServiceLocatorClientProvider_:

{{< highlight java "hl_lines=11-12 15" >}}
public class MyRequestFilter implements ClientRequestFilter {

    // this injection does not work as filter is registered as an instance:
    // @Inject
    // private MyInjectedService service;

    @Override
    public void filter(ClientRequestContext requestContext) throws IOException {
        // use ServiceLocatorClientProvider to extract HK2 ServiceLocator
        // from request
        ServiceLocator locator = ServiceLocatorClientProvider
                                     .getServiceLocator(requestContext);

        // and ask for MyInjectedService:
        MyInjectedService service = locator.getService(MyInjectedService.class);

        String name = service.getName();
        ...
    }
}
{{< / highlight >}}

For more information see javadoc of <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/ServiceLocatorClientProvider.html">ServiceLocatorClientProvider</a> (and javadoc of <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/ServiceLocatorProvider.html">ServiceLocatorProvider</a> which supports common JAX-RS components).

## Repackaged Guava and ASM

Jersey, from versions 2.6 for JAX-RS 2.0 and 1.18.1 for JAX-RS 1.1, no longer transitively brings <a href="https://code.google.com/p/guava-libraries/">Guava</a> and <a href="http://asm.ow2.org/">ASM</a> libraries to your application. This means that even when we still use them internally you can use different versions of these libraries. Classes from both of these libraries has been repackaged, _jersey.repackaged.com.google.common_ and _jersey.repackaged.objectweb.asm_ respectively, and shrinked to lower the footprint. ASM 5 is now part of <a href="https://github.com/jersey/jersey/tree/2.6/core-server">jersey-server</a> core module and for Guava we&#8217;ve created a separate bundle module <a href="https://github.com/jersey/jersey/tree/2.6/bundles/repackaged/jersey-guava">jersey-guava</a> as this dependency is widely used in multiple Jersey modules.

## Transparent PATCH Support Example

<a href="https://twitter.com/marek_potociar">Marek</a> created <a href="https://github.com/jersey/jersey/tree/2.6/examples/http-patch">http-patch</a> example based on a very nice <a href="https://twitter.com/kingsfleet">Gerard</a>&#8216;s blog-post <a href="http://kingsfleet.blogspot.co.uk/2014/02/transparent-patch-support-in-jax-rs-20.html">Transparent PATCH support in JAX-RS 2.0</a>. The example demonstrates how to implement generic support for <a href="https://tools.ietf.org/html/rfc5789">HTTP PATCH Method (RFC-5789)</a> via JAX-RS reader interceptor (see <a href="https://github.com/jersey/jersey/blob/2.6/examples/http-patch/src/main/java/org/glassfish/jersey/examples/httppatch/PATCH.java#L68">PATCH</a>, <a href="https://github.com/jersey/jersey/blob/2.6/examples/http-patch/src/main/java/org/glassfish/jersey/examples/httppatch/PatchableResource.java#L85">resource</a> and <a href="https://github.com/jersey/jersey/blob/2.6/examples/http-patch/src/main/java/org/glassfish/jersey/examples/httppatch/PatchingInterceptor.java#L73">interceptor</a>). The patch format supported by the example is [JSON Patch (RFC-6902)][2].

The example contains one resource that represents a _patchable_ state, which consists of 3 properties:

  * a _title_ text property,
  * a _message_ text property, and
  * a _list_ property that represents a list/array of text strings.

The default state in JSON format looks like:

{{< highlight json >}}
{
   "list" : [ ],
   "message" : "",
   "title" : ""
}
{{< / highlight >}}

This state can be patched via HTTP PATCH method by sending the desired patch/diff in the JSON Patch format as defined in RFC-6902, for example:

{{< highlight json "hl_lines=5 10" >}}
[
  {
    "op" : "replace",
    "path" : "/message",
    "value" : "patchedMessage"
  },
  {
    "op" : "add",
    "path" : "/list/-",
    "value" : "one"
  }
]
{{< / highlight >}}

And the result:

{{< highlight json "hl_lines=2-3" >}}
{
   "list" : [ "one" ],
   "message" : "patchedMessage",
   "title" : ""
}
{{< / highlight >}}

## Defining Entity Filtering output Fields via Query Parameters

<a href="http://www.andypemberton.com/">Andy</a> enhanced our <a href="/2014/02/04/filtering-jax-rs-entities-with-standard-security-annotations/">Entity Filtering</a> feature with an option to define output fields of a bean dynamically via query parameters. Suppose you have a JAXB bean like:

{{< highlight java >}}
@XmlRootElement
public class Address {

    private String streetAddress;

    private String region;

    private PhoneNumber phoneNumber;

    // getters, setters
}
{{< / highlight >}}

And you only need _streetAddress_ and _region_ on your client. By registering <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SelectableEntityFilteringFeature.html">SelectableEntityFilteringFeature</a> it&#8217;s possible to define these fields in a query parameter (supported in comma delimited &#8220;dot notation&#8221; style). For example, the following URL

{{< highlight url >}}
http://jersey.example.com/addresses/51234?select=region,streetAddress
{{< / highlight >}}

will do the magic:

{{< highlight json >}}
{
   "region" : "CA",
   "streetAddress" : "1234 Fake St."
}
{{< / highlight >}}

For more information about _Selectable Entity Filtering_ refer to our <a href="https://jersey.java.net/documentation/latest/entity-filtering.html">documentation</a> and <a href="https://github.com/jersey/jersey/tree/2.6/examples/entity-filtering-selectable">entity-filtering-selectable</a> example.

## Using Custom ObjectMapper in Jackson 2.x Providers

Jersey provides support for Jackson 1.x JSON providers via a separate media module <a href="https://github.com/jersey/jersey/tree/2.6/media/json-jackson">jersey-media-json-jackson</a>. For Jackson 2.x we don&#8217;t need any special module at the moment because Jackson 2.x JSON providers are registered automatically when they&#8217;re present on the class-path of JAX-RS application (see <a href="https://github.com/FasterXML/jackson-jaxrs-providers">jackson-jaxrs-providers</a>). Due to a bug in previous versions of Jersey 2.x it wasn&#8217;t possible to pass an <a href="http://fasterxml.github.io/jackson-databind/javadoc/2.3.0/com/fasterxml/jackson/databind/ObjectMapper.html">ObjectMapper</a> instance to message body readers and writers via <a href="https://jersey.github.io/apidocs/latest/jersey/javax/ws/rs/ext/ContextResolver.html">ContextResolver</a> mechanism:

{{< highlight java >}}
@Provider
public class MyObjectMapperResolver implements ContextResolver<ObjectMapper> {

    @Override
    public ObjectMapper getContext(Class<?> type) {
        return new ObjectMapper()
                .configure(SerializationFeature.WRAP_ROOT_VALUE, true)
                .configure(DeserializationFeature.UNWRAP_ROOT_VALUE, false);
    }
}
{{< / highlight >}}

Thanks to <a href="https://twitter.com/japod">Jakub</a> the bug has been fixed and you can configure and use Jackson 2.x providers without any issues.

## Jersey Libraries in WARs deployed on a CDI-enabled Container

When deploying a Jersey (< 2.6) application with Jersey bits bundled in the web-app archive (WAR) on a container with enabled CDI (i.e. GlassFish) you can run into a bunch of exceptions. These exceptions are trying to tell you that some of the (CDI) dependencies couldn&#8217;t be satisfied. The rationale behind this is that every class present in the WAR that uses _@Inject_ annotation are considered to be a CDI bean which needs to be resolved immediately. Even before our dependency injection framework, HK2, has a chance to say whether it can satisfy the injection point instead of CDI.

Fortunately, via implementing the <a href="http://docs.jboss.org/cdi/api/1.1/javax/enterprise/inject/spi/Extension.html">Extension</a> contract, it&#8217;s possible to let the CDI know that it should ignore certain classes so you can handle these injection points by yourself somewhere else.

Fixing this issue brings a nice little &#8220;side-effect&#8221; about which I&#8217;ll write another blog post (soon).

## Declarative Linking has been Migrated from Jersey 1.x

Thanks to <a href="http://kingsfleet.blogspot.co.uk/">Gerard</a> the &#8220;declarative linking&#8221; feature has been migrated from Jersey 1.x into Jersey 2.x and is available in our <a href="https://github.com/jersey/jersey/tree/2.6/incubator">incubator</a> in <a href="https://github.com/jersey/jersey/tree/2.6/incubator/declarative-linking">declarative-linking</a> module.

## Release Notes

To see the complete list of changes and bug-fixes refer to <a href="https://jersey.java.net/release-notes/2.6.html">Jersey 2.6 Release Notes</a>.

 [1]: /latest-jersey-version/ "Latest Jersey Versions"
 [2]: http://tools.ietf.org/html/rfc6902