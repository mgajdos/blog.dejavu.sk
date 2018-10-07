---
title: JSON in Query Params or How to Inject Custom Java Types via JAX-RS Parameter Annotations
author: Michal Gajdoš
type: post
date: 2014-02-11T09:35:29+00:00
url: /inject-custom-java-types-via-jax-rs-parameter-annotations/
aliases: /2014/02/11/inject-custom-java-types-via-jax-rs-parameter-annotations/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey

---
Although I am not a big fan of sending JSON in other places than in the message body as the entity, for example in query parameter in case of requests, it&#8217;s not a rare use-case and the way how it can be solved in JAX-RS 2.0 gives me a nice opportunity to illustrate usage of two new interfaces, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ParamConverterProvider.html">ParamConverterProvider</a> and <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ParamConverter.html">ParamConverter</a>. You can then re-use this approach to inject other media-types or formats your application relies on via @*Param annotations (@MatrixParam, @QueryParam, @PathParam, @CookieParam, @HeaderParam).

<!--more-->

## Use-Case: Sending JSON as Query Parameter

> All snippets listed in this article are taken from <a href="https://github.com/mgajdos/jersey-param-converters">jersey-param-converters</a> (see [Example][1]) available at GitHub.

#### Java POJO <-> JSON

In the examples below I&#8217;ll be using the following <a href="https://github.com/mgajdos/jersey-param-converters/blob/master/src/main/java/sk/dejavu/blog/examples/paramconverters/model/Entity.java">Entity</a> class as an input and also as output in my resource methods:

{{< highlight java >}}
public class Entity {

    private String foo;
    private String bar;

    // ...

}
{{< / highlight >}}

In JSON format an instance of this class would, more-or-less, look like:

{{< highlight json >}}
{"Entity":{"foo":"foo","bar":"bar"}}
{{< / highlight >}}

or

{{< highlight json >}}
{
  "foo" : "foo",
  "bar" : "bar"
}
{{< / highlight >}}

#### Client

Let&#8217;s assume that you want to send a JSON string as a query or a header parameter to your REST service. The client code, in case of query parameter, could look like this:

{{< highlight java "hl_lines=4-5" >}}
final Response response = ClientBuilder.newClient()
        .target("json-param-converter")
        .path("query")
        .queryParam("entity",
            UriComponent.encode(json, UriComponent.Type.QUERY_PARAM_SPACE_ENCODED))
        .request()
        .get();
{{< / highlight >}}

> To make sure the resulting JSON is properly encoded in URI I am using <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/uri/UriComponent.html">UriComponent</a> helper class from Jersey to encode JSON as <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/uri/UriComponent.Type.html#QUERY_PARAM_SPACE_ENCODED">QUERY_PARAM_SPACE_ENCODED</a> type.

The actual request would look like:

{{< highlight text "hl_lines=3" >}}
Feb 09, 2014 4:55:44 PM org.glassfish.jersey.filter.LoggingFilter log
INFO: 1 * Sending client request on thread main
1 > GET http://localhost:9998/json-param-converter/query?entity=%7B%22Entity%22:%7B%22foo%22:%22foo%22,%22bar%22:%22bar%22%7D%7D
{{< / highlight >}}

In case we want to send our JSON as a header value the client code is little bit simpler:

{{< highlight java "hl_lines=5" >}}
final Response response = ClientBuilder.newClient()
        .target("json-param-converter")
        .path("header")
        .request()
        .header("Entity", json)
        .get();
{{< / highlight >}}

And the request:

{{< highlight text "hl_lines=4" >}}
Feb 09, 2014 4:55:44 PM org.glassfish.jersey.filter.LoggingFilter log
INFO: 1 * Sending client request on thread main
1 > GET http://localhost:9998/json-param-converter/header
1 > Entity: {"Entity":{"foo":"foo","bar":"bar"}}
{{< / highlight >}}

#### Server

We want the resource class, handling requests described above, to be as simple as possible. No custom conversion of JSON strings to Java objects in our resource methods. JAX-RS runtime should do this for us. So, all we need to do is write a resource class similar to:

{{< highlight java "hl_lines=8-10 16" >}}
@Path("json-param-converter")
@Produces("application/json")
public class JsonParamConverterResource {

    @GET
    @Path("query")
    public Entity getViaQueryParam(
            @QueryParam("entity")
            @DefaultValue("{\"Entity\":{\"foo\":\"bar\",\"bar\":\"foo\"}}")
            final Entity entity) {
        return entity;
    }

    @GET
    @Path("header")
    public Entity getViaHeaderParam(@HeaderParam("Entity") final Entity entity) {
        return entity;
    }
}
{{< / highlight >}}

As you can see there are two _@GET_ methods. Both are returning injected entity, transformed to JSON, to the client. The first one is trying to inject value of a query parameter named _entity_ (see <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/QueryParam.html">@QueryParam</a>) into the method parameter of type _Entity_. If JAX-RS runtime cannot find such a query parameter the value from <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/DefaultValue.html">@DefaultValue</a> is used to create _Entity_ instead. The second one is, similarly, injecting value of a request header named _Entity_ (see <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/HeaderParam.html">@HeaderParam</a>).

From both of the resource methods we&#8217;d like to return a response similar to:

{{< highlight text "hl_lines=7-10" >}}
Feb 09, 2014 4:55:44 PM org.glassfish.jersey.filter.LoggingFilter log
INFO: 2 * Client response received on thread main
2 < 200
2 < Content-Length: 36
2 < Content-Type: application/json
2 < Date: Fri, 09 Feb 2014 15:55:44 GMT
{
  "foo" : "foo",
  "bar" : "bar"
}
{{< / highlight >}}

OK, enough of what we want to achieve, let&#8217;s take a look at how to actually do it.

## Param Converters (and Providers)

To make the injection work we need to create implementations of two interfaces from JAX-RS 2.0, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ParamConverterProvider.html">ParamConverterProvider</a> and <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ParamConverter.html">ParamConverter</a>. With the first one we simply tell the runtime whether we&#8217;re able to convert _String_ to a particular type and with the second one we&#8217;re doing the actual conversion.

So, for our use-case of converting JSON into custom Java types we&#8217;re going to create a slightly more powerful version of converter provider and converter itself. For the purpose of converting we&#8217;ll be using <a href="https://github.com/FasterXML/jackson">Jackson</a> library, particularly it&#8217;s <a href="http://fasterxml.github.io/jackson-databind/javadoc/2.1.0/com/fasterxml/jackson/databind/ObjectMapper.html">ObjectMapper</a> class. As you might know it&#8217;s possible to pass your own instances of _ObjectMapper_ to Jackson message body providers via custom <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ContextResolver.html">ContextResolver</a> (see below). We want to use this mechanism even with converting JSON to parameters so both approaches (converting request entity and converting parameters) have the configuration of _ObjectMapper_ in one place:

{{< highlight java "hl_lines=6-8" >}}
@Provider
public class ObjectMapperContextResolver implements ContextResolver<ObjectMapper> {

    @Override
    public ObjectMapper getContext(final Class<?> type) {
        return new ObjectMapper()
                .configure(DeserializationFeature.UNWRAP_ROOT_VALUE, true)
                .configure(SerializationFeature.INDENT_OUTPUT, true);
    }
}
{{< / highlight >}}

And our implementations of _ParamConverterProvider_ and _ParamConverter_ are:

{{< highlight java "linenos=inline,hl_lines=12-17 20-24 32" >}}
@Provider
public class JacksonJsonParamConverterProvider implements ParamConverterProvider {

    @Context
    private Providers providers;

    @Override
    public <T> ParamConverter<T> getConverter(final Class<T> rawType,
                                              final Type genericType,
                                              final Annotation[] annotations) {
        // Check whether we can convert the given type with Jackson.
        final MessageBodyReader<T> mbr = providers.getMessageBodyReader(rawType,
                genericType, annotations, MediaType.APPLICATION_JSON_TYPE);
        if (mbr == null
              || !mbr.isReadable(rawType, genericType, annotations, MediaType.APPLICATION_JSON_TYPE)) {
            return null;
        }

        // Obtain custom ObjectMapper for special handling.
        final ContextResolver<ObjectMapper> contextResolver = providers
                .getContextResolver(ObjectMapper.class, MediaType.APPLICATION_JSON_TYPE);

        final ObjectMapper mapper = contextResolver != null ?
                contextResolver.getContext(rawType) : new ObjectMapper();

        // Create ParamConverter.
        return new ParamConverter<T>() {

            @Override
            public T fromString(final String value) {
                try {
                    return mapper.reader(rawType).readValue(value);
                } catch (IOException e) {
                    throw new ProcessingException(e);
                }
            }

            @Override
            public String toString(final T value) {
                try {
                    return mapper.writer().writeValueAsString(value);
                } catch (JsonProcessingException e) {
                    throw new ProcessingException(e);
                }
            }
        };
    }
}
{{< / highlight >}}

As you can see it&#8217;s not complicated at all and hence it&#8217;s pretty useful. In the first part (lines 12–17) I am making sure that the JSON<->Java message body reader (the one from Jackson as I didn&#8217;t register any other JSON providers) is capable of unmarshalling JSON into Java POJO of given type. If this is the case I know that I can also convert JSON into this Java POJO and inject it via parameter annotations.

In the second part (lines 20–24) I am looking for a custom _ObjectMapper_. If there is one I&#8217;ll use it during conversion otherwise I&#8217;ll create a default one.

And finally on lines 27–46 I am creating new _ParamConverter_ which is actually converting the JSON string into an instance of given type (line 32). I could use the retrieved message body reader to the job and the result would be the same, of course. But I think my approach is better to illustrate how this mechanism really works.

## Default JAX-RS Param Converters

In some (basic) cases it&#8217;s not necessary to create your own param converters (and providers) to inject parameters via _@*Param_ annotations (<a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/MatrixParam.html">@MatrixParam</a>, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/QueryParam.html">@QueryParam</a>, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/PathParam.html">@PathParam</a>, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/CookieParam.html">@CookieParam</a>, <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/HeaderParam.html">@HeaderParam</a>) because JAX-RS 2.0 implementations have to support the following types:

  1. Primitive types.
  2. Types that have a constructor that accepts a single _String_ argument.
  3. Types that have a static method named _valueOf_ or _fromString_ with a single _String_ argument that return an instance of the type. If both methods are present then _valueOf_ MUST be used unless the type is an enum in which case _fromString_ MUST be used.
  4. _List<T>_, _Set<T>_, or _SortedSet<T>_, where _T_ satisfies 2 or 3 above.

This basically tells us that when there are only a few classes which we want to inject it&#8217;s sufficient to create static methods _fromString_ / _valueOf_ in these classes (or make sure the classes have a constructor which accepts one string argument) and we can use them as injection targets.

## Additional Jersey Param Converters

In addition to converters mentioned above Jersey provides two more param converters for the following types:

  1. _Date_ &#8211; for following date format patterns 
      * `EEE, dd MMM yyyy HH:mm:ss zzz` (RFC 1123)
      * `EEEE, dd-MMM-yy HH:mm:ss zzz` (RFC 1036)
      * `EEE MMM d HH:mm:ss yyyy` (ANSI C)
  2. JAXB bean &#8211; has to be annotated either with _@XmlRootElement_ or _@XmlType_

The _Date_ param converter has the highest priority and the JAXB one the lowest priority from all the default converters. This means than in case you have a JAXB class like:

{{< highlight java "hl_lines=1 6-8" >}}
@XmlRootElement
public class JaxbBean {

    private String value;

    public static JaxbBean fromString(final String bean) {
        // ...
    }

    // getters and setters
}
{{< / highlight >}}

The injected _JaxbBean_ would be created using _fromString_ JAX-RS default param converter and not the JAXB one.

## <a name="example"></a>Example

As I mentioned earlier there is an example you can try (both client and server sides) available on GitHub: <a href="https://github.com/mgajdos/jersey-param-converters">jersey-param-converters</a>. It&#8217;s a simple JAX-RS <a href="https://github.com/mgajdos/jersey-param-converters/tree/master/src/main/java/sk/dejavu/blog/examples/paramconverters">web-application</a> with <a href="https://github.com/mgajdos/jersey-param-converters/blob/master/src/test/java/sk/dejavu/blog/examples/paramconverters/JsonParamConverterResourceTest.java#L24">3 test-cases</a> that are representing the client side story.

## Further Reading

  * Jersey User Guide &#8211; <a href="https://jersey.github.io/documentation/latest/jaxrs-resources.html#d0e1811">Chapter 3. Parameter Annotations (@*Param)</a>

 [1]: #example