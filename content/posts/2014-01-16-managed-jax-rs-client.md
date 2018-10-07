---
title: Managed JAX-RS Client
author: Michal Gajdoš
type: post
date: 2014-01-16T17:35:41+00:00
url: /managed-jax-rs-client/
aliases: /2014/01/16/managed-jax-rs-client/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - client

---
Common use-case in web-application development is aggregating data from multiple resources, combining them together and returning them to the used as XML/JSON or as a web page. In Java world these (external) resources can be approached via standardized <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/client/Client.html">Client</a>s from JAX-RS 2.0. Jersey 2 application can use so-called managed client mechanism that brings a convenient way to create JAX-RS <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/client/Client.html">clients</a> and <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/client/WebTarget.html">web targets</a> for such resources.

<!--more-->

In this article I&#8217;ll demonstrate the concepts on a simple application that displays ratings of movies obtained from different sources:

  * <a href="http://www.imdb.com/">IMDb</a> via <a href="http://omdbapi.com/">omdbapi.com</a> 
      * search & movie detail &#8211; <a href="http://www.omdbapi.com/?r=xml&t=fargo">http://www.omdbapi.com/?r=<strong>{format}</strong>&t=<strong>{title}</strong></a>
  * <a href="http://www.csfd.cz/">CSFD</a> (clone of IMDb; only in Czech) via <a href="http://csfdapi.cz/">csfdapi.cz</a> 
      * search &#8211; <a href="http://csfdapi.cz/movie?search=fargo">http://csfdapi.cz/movie?search=<strong>{title}</strong></a>
      * movie detail &#8211; <a href="http://csfdapi.cz/movie/1606">http://csfdapi.cz/movie/<strong>{id}</strong></a>

You can try the application at

<p style="padding-left: 30px;">
  <a href="http://jersey-managed-jaxrs-client.herokuapp.com">jersey-managed-jaxrs-client.herokuapp.com</a> (deployed on <a href="/2014/01/09/running-jersey-2-applications-on-heroku/">Heroku</a>)<br /> for example: <a href="http://jersey-managed-jaxrs-client.herokuapp.com/rating/fargo">jersey-managed-jaxrs-client.herokuapp.com/rating/fargo</a>
</p>

and browse the code

<p style="padding-left: 30px;">
  <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client">github.com/mgajdos/jersey-managed-jaxrs-client</a>
</p>

## Injecting WebTarget via @Uri

Injecting a resource target pointing at a resource identified by the resolved URI can be done via new annotation <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/Uri.html">@Uri</a>. The annotation can be placed on <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/client/WebTarget.html">WebTarget</a> that represents

  * a method parameter, i.e. <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/FormResource.java#L20">FormResource</a>: {{< highlight java "hl_lines=8" >}}
@Path("/{greeting: .*}")
public class FormResource {

    @GET
    @Produces("text/html")
    @Template(name = "/form")
    public String getForm(
            @Uri("internal/greeting/{greeting}") final WebTarget greeting) {
        return greeting.request().get(String.class);
    }
}
{{< / highlight >}}

  *  a class field for example in resources or providers (filters, interceptors, ..), i.e. <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/RatingsResource.java#L34">RatingsResource</a>: {{< highlight java "hl_lines=6 10" >}}
@Path("/rating")
@Produces("text/html")
public class RatingsResource {

    @CsfdClient
    @Uri("movie/{id}")
    private WebTarget csfdMovie;

    @ImdbClient
    @Uri("http://omdbapi.com/?r={format}&t={search}")
    private WebTarget imdbMovie;

    ...
}
{{< / highlight >}}

  * or a bean property.

As you probably noticed the value of _@Uri_ can be either absolute or relative URI. The relative ones are resolved into absolute URIs in the context of application (context path) or in a context of enclosing resource class. Web target from the first example (`internal/greeting/{greeting}`) is pointing at an internal resource (present in the same application) but web target from the second example (`movie/{id}`) is not relative to our application as you&#8217;ll see in the next section.

Provided URIs can also contain template parameters (`internal/greeting/{greeting}`, `r={format}&t={search}`) which are resolved automatically (`internal/greeting/{greeting}`), if they are represented as path params (see <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/Path.html">@Path</a>):

{{< highlight java "hl_lines=1 8" >}}
@Path("/{greeting: .*}")
public class FormResource {

    @GET
    @Produces("text/html")
    @Template(name = "/form")
    public String getForm(
            @Uri("internal/greeting/{greeting}") final WebTarget greeting) {
        return greeting.request().get(String.class);
    }
}
{{< / highlight >}}

> The `greeting` parameter is used on the front page in the header (it&#8217;s enough to set it, i.e. <a href="http://jersey-managed-jaxrs-client.herokuapp.com/Greetings,%20Commander">jersey-managed-jaxrs-client/Greetings, Commander</a>, you don&#8217;t need to resolve it by hand).

or you can resolve them manually (`r={format}&t={search}`):

{{< highlight java "hl_lines=2-3" >}}
final Response response = imdbMovie
        .resolveTemplate("search", search)
        .resolveTemplate("format", format)
        .request("application/" + format)
        .get();
{{< / highlight >}}

## Configuring

By default, injected clients are configured with features and providers from the <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Configuration.html">configuration</a> of an application (i.e. <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/MyApplication.java#L13">MyApplication</a>) that are also applicable to clients. In our case only <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/jackson/JacksonFeature.html">JacksonFeature</a> would be taken into consideration.

In case this is not enough for you need to take a look at <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ClientBinding.html">@ClientBinding</a>.

#### @ClientBinding

Meta-annotation that provides a facility for creating bindings between an _@Uri_-injectable web target instances and clients (and their configurations) that are used to create the injected web target instances.

So, basically, the first step is to create a custom client-binding annotation and put <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ClientBinding.html">@ClientBinding</a> on top of it, i.e. <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/CsfdClient.java">CsfdClient</a>:

{{< highlight java "hl_lines=4-6" >}}
@Retention(RetentionPolicy.RUNTIME)
@Target({ElementType.FIELD, ElementType.PARAMETER})
@ClientBinding(
        baseUri = "http://csfdapi.cz/",
        inheritServerProviders = false,
        configClass = ManagedClientConfig.class
)
public @interface CsfdClient {
}
{{< / highlight >}}

Second step is to annotate _WebTarget_ with custom client-binding annotation along with _@Uri_, i.e. <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/RatingsResource.java#L34">RatingsResource</a>:

{{< highlight java "hl_lines=5-6 9-10" >}}
@Path("/rating")
@Produces("text/html")
public class RatingsResource {

    @CsfdClient
    @Uri("movie")
    private WebTarget csfdSearch;

    @CsfdClient
    @Uri("movie/{id}")
    private WebTarget csfdMovie;

    ...
}
{{< / highlight >}}

> Notice that one client (created for @CsfdClient) is used to create two WebTarget instances that are used separately.

Third step is to use the injected web target.

Custom client-binding annotation can be configured programmatically (see _CsfdClient_) via parameters of _@ClientBinding_ which are:

  * <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/CsfdClient.java#L16">baseUri</a> &#8211; base URI which is used to absolutize relative URI provided in _@Uri_ (this is the reason why `csfdMovie` / `csfdSearch` web targets aren&#8217;t pointing at internal resource).
  * <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/CsfdClient.java#L17">inheritServerProviders</a> &#8211; determines whether server providers ought to be registered in the client (and web targets) as well.
  * <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/CsfdClient.java#L18">configClass</a> &#8211; configuration class to configure client (and web targets), in our case it&#8217;s <a href="https://github.com/mgajdos/jersey-managed-jaxrs-client/blob/master/src/main/java/sk/dejavu/jersey/sample/ManagedClientConfig.java">ManagedClientConfig</a> (that extends <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/client/ClientConfig.html">ClientConfig</a>) where we&#8217;re registering <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/filter/LoggingFilter.html">LoggingFilter</a> and <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/jackson/JacksonFeature.html">JacksonFeature</a>.

#### web.xml

Parameters of _@ClientBinding_ for particular client-binding annotation can be set declaratively in `web.xml` via `init-param`-eters. You need to remember to prefix parameter name with fully-qualified name of custom client-binding annotation:

{{< highlight xml "hl_lines=10-21" >}}
<servlet>
    <servlet-name>sk.dejavu.jersey.sample.MyApplication</servlet-name>
    <servlet-class>org.glassfish.jersey.servlet.ServletContainer</servlet-class>
    <init-param>
        <param-name>javax.ws.rs.Application</param-name>
        <param-value>sk.dejavu.jersey.sample.MyApplication</param-value>
    </init-param>

    <!-- Imdb client configuration -->
    <init-param>
        <param-name>sk.dejavu.jersey.sample.ImdbClient.configClass</param-name>
        <param-value>sk.dejavu.jersey.sample.ManagedClientConfig</param-value>
    </init-param>
    <init-param>
        <param-name>sk.dejavu.jersey.sample.ImdbClient.inheritServerProviders</param-name>
        <param-value>false</param-value>
    </init-param>
    <init-param>
        <param-name>sk.dejavu.jersey.sample.ImdbClient.property.format</param-name>
        <param-value>xml</param-value>
    </init-param>

    <load-on-startup>1</load-on-startup>
</servlet>
{{< / highlight >}}

The last highlighted init-parameter (`sk.dejavu.jersey.sample.ImdbClient.property.format`) shows special way how to define a property for the newly created client. In this case we can obtain value of the property from configuration of our web target and set the format of entities received from `imdbMovie`:

{{< highlight java "hl_lines=1 5" >}}
final String format = imdbMovie.getConfiguration().getProperty("format").toString();

final Response response = imdbMovie
        .resolveTemplate("search", search)
        .resolveTemplate("format", format)
        .request("application/" + format)
        .get();
{{< / highlight >}}

## Examples

Besides the example I&#8217;ve been using in this article there are few others available in the Jersey workspace:

  * <a id="9d260187adf25affd29b1b8000716457-3f5e4373b3edab85e9091a54b38a091c8b399d74" href="https://github.com/jersey/jersey/tree/master/examples/managed-client">managed-client</a>
  * <a id="d1d63b524bea4ef923e6e95c38927650-d1992824ecec1df39b1efbc5254db3335b19a752" href="https://github.com/jersey/jersey/tree/master/examples/managed-client-webapp">managed-client-webapp</a> (+ GF4 <a href="http://repo1.maven.org/maven2/org/glassfish/jersey/examples/managed-client-webapp/2.5.1/managed-client-webapp-2.5.1-gf-project-src.zip">project</a>)
  * <a id="3eeb23af528f70ad7dbf3cdd132a8880-0ec0be24965fc8066dce9139d58a2c5e635462b8" href="https://github.com/jersey/jersey/tree/master/examples/managed-client-simple-webapp">managed-client-simple-webapp</a> (+ GF4 <a href="http://repo1.maven.org/maven2/org/glassfish/jersey/examples/managed-client-simple-webapp/2.5.1/managed-client-simple-webapp-2.5.1-gf-project-src.zip">project</a>)

## Further reading

  * <a href="/2014/01/09/running-jersey-2-applications-on-heroku/">Running Jersey 2 Applications on Heroku</a>