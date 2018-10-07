---
title: Intercepting Jersey resource method calls
author: Michal Gajdoš
type: post
date: 2015-01-21T09:30:29+00:00
url: /intercepting-jersey-resource-method-calls/
aliases: /2015/01/21/intercepting-jersey-resource-method-calls/
categories:
  - Jersey
tags:
  - jersey
  - hk2

---
Sometimes it would be nice to have a possibility to wrap a resource method call. Imagine a use-case in which you need to process business code, placed in your resource method, in a transaction (open transaction before resource method, commit or rollback transaction at the end) or apply some advanced security constraints. If you&#8217;re using JAX-RS together with CDI in your application it&#8217;s not a problem to do such things. Fortunately, with Jersey it&#8217;s also possible to do those things outside of CDI.

<!--more-->

In Jersey 1 there are dedicated SPI classes to achieve this task, <a href="https://jersey.github.io/apidocs/1.18/jersey/com/sun/jersey/spi/container/ResourceMethodDispatchProvider.html">ResourceMethodDispatchProvider</a> and <a href="https://jersey.github.io/apidocs/1.18/jersey/com/sun/jersey/spi/dispatch/RequestDispatcher.html">RequestDispatcher</a>. Once you register your dispatch provider by placing an identifier in the resource directory (`META-INF/services`) you can start creating request dispatchers that wraps and affects the invocation of resource methods.

Jersey 2 doesn&#8217;t expose similar SPI classes but users are be able to leverage means provided by <a href="https://hk2.java.net/">HK2</a>, an injection framework Jersey uses internally. HK2 supports some AOP capabilities which means that Jersey 2 applications can take advantage of these capabilities as well.

In general, constructors and methods of each instance managed by HK2 can be intercepted but, in this article, we&#8217;re going to focus on intercepting just JAX-RS Resources and Providers.

To demonstrate the feature I&#8217;ve created a simple example, <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods">jersey-intercepting-resource-methods</a>, excerpts of which I&#8217;d be showing in the following sections.

## Setup

Interception entry point in HK2, that identifies instances containing constructors and methods to be intercepted, is called <a href="https://hk2.java.net/2.4.0-b08/apidocs/org/glassfish/hk2/api/InterceptionService.html">InterceptionService</a>.

In the simplest case you just need to, in your implementation, provide a filter (_getDescriptorFilter()_ method, see line 5 in the example below) to filter out managed objects on the class level and then you can assign each given constructor or method a list of interceptors to be invoked before the actual constructor or method is invoked.

{{< highlight java "linenos=inline,hl_lines=5 18 29" >}}
@Override
public Filter getDescriptorFilter() {
    // We're only interested in classes (resources, providers) from this
    // applications packages.
    return new Filter() {
        @Override
        public boolean matches(final Descriptor d) {
            final String clazz = d.getImplementation();
            return clazz.startsWith("sk.dejavu.blog.examples.intercepting.resources")
                    || clazz.startsWith("sk.dejavu.blog.examples.intercepting.providers");
        }
    };
}

@Override
public List<MethodInterceptor> getMethodInterceptors(final Method method) {
    // Apply interceptors only to methods annotated with @Intercept.
    if (method.isAnnotationPresent(Intercept.class)) {
        ...
    }
    return null;
}

@Override
public List<ConstructorInterceptor> getConstructorInterceptors(
        final Constructor<?> constructor) {

    // Apply interceptors only to constructors annotated with @Intercept.
    if (constructor.isAnnotationPresent(Intercept.class)) {
        ...
    }
    return null;
}
{{< / highlight >}}

> A working implementation of the service is available in <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/MyInterceptionService.java#L23">MyInterceptionService</a> class.

Now, when we have our custom implementation of the interception service we need to let HK2 know about it. There are two ways how to do the job. The first approach is to extend <a href="https://hk2.java.net/2.4.0-b08/apidocs/org/glassfish/hk2/utilities/binding/AbstractBinder.html">AbstractBinder</a> and register it&#8217;s instance in Jersey <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ResourceConfig.html">ResourceConfig</a> (or sub-class of JAX-RS Application).

{{< highlight java "hl_lines=6-7" >}}
public class MyInterceptionBinder extends AbstractBinder {

    @Override
    protected void configure() {
        bind(MyInterceptionService.class)
                .to(org.glassfish.hk2.api.InterceptionService.class)
                .in(Singleton.class);
    }
}
{{< / highlight >}}

> MyInterceptionService has to be bound to HK2&#8217;s InterceptionService contract in the Singleton scope.

{{< highlight java "hl_lines=8" >}}
@ApplicationPath("/")
public class Application extends ResourceConfig {

    public Application() {
        packages("sk.dejavu.blog.examples.intercepting");

        // Register Interception Service.
        register(new MyInterceptionBinder());
    }
}
{{< / highlight >}}

The second way is to annotate our custom interception service with <a href="https://hk2.java.net/2.4.0-b08/apidocs/org/jvnet/hk2/annotations/Service.html">@Service</a> annotation and use <a href="https://hk2.java.net/inhabitant-generator.html">hk2-inhabitant-generator</a> to look for all HK2 services in the application. References to HK2 services are saved during build into a file bundled with the application. The locator file is later used to automatically register services during runtime.

To try the latter approach in the demo application just uncomment the mentioned plugin in <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/pom.xml#L57">pom.xml</a> and disable the registration of <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/MyInterceptionBinder.java#L12">MyInterceptionBinder</a> in the <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/Application.java#L19">Application</a> class.

## Resources

Let&#8217;s take a look at intercepting JAX-RS Resources first. Assume that we have a simple resource with two resource methods that return plain String.

{{< highlight java "hl_lines=9 17 19" >}}
@Path("server")
public class ServerResource {

    /**
     * Resource method is not intercepted itself.
     */
    @GET
    public String get() {
        return "ServerResource: Non-intercepted method invoked\n";
    }

    /**
     * Resource method is intercepted by {@code ResourceInterceptor}.
     */
    @GET
    @Path("intercepted")
    @Intercept
    public String getIntercepted() {
        return "ServerResource: Intercepted method invoked\n";
    }
}
{{< / highlight >}}

The first method, _get()_, is not intercepted so when we invoke a request that is handled by this method, either via test (see <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/test/java/sk/dejavu/blog/examples/intercepting/InterceptingTest.java#L30">InterceptingTest</a>) or via `curl`, we&#8217;d get:

```bash
ServerResource: Non-intercepted method invoked
```

As expected we received the message defined in the resource method on the client side and nothing more.

The second method, _getIntercepted()_, is intercepted by a custom resource method interceptor, <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/ResourceInterceptor.java#L14">ResourceInterceptor</a>, which appends an additional line at the end of the message returned from the method. The <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/Intercept.java#L13">@Intercept</a> annotation placed above this method tells our interception/filtering service that this method should be considered for intercepting.

{{< highlight java "hl_lines=8-10" >}}
public class ResourceInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(final MethodInvocation methodInvocation) throws Throwable {
        // Modify the entity returned by the invoked resource method
        // by appending additional message.
        return methodInvocation.proceed()
                + "   ResourceInterceptor: Method \""
                + methodInvocation.getMethod().getName()
                + "\" intercepted\n";
    }
}
{{< / highlight >}}

When we fetch this resource the resulting entity would look like:

{{< highlight text "hl_lines=2" >}}
ServerResource: Intercepted method invoked
ResourceInterceptor: Method "getIntercepted" intercepted
{{< / highlight >}}

In the output we can clearly see that the method was intercepted and the additional line was added by our method interceptor.

## Providers

I wrote that all types/objects managed by HK2 can be intercepted. This also includes JAX-RS Providers that are registered in your application. For instance, lets create a message body provider (<a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/MessageBodyReader.html">MessageBodyReader</a>) that would be reading input stream and transforming it into String. (Yes, there is such a provider available in every JAX-RS implementation but with our implementation we&#8217;re going to suppress the default one and slightly modify its behavior.)

{{< highlight java "hl_lines=8 17 22 33 34" >}}
public class StringProvider extends AbstractMessageReaderWriterProvider<String> {

    @Context
    private Configuration config;

    private String providerName;

    @Intercept
    public StringProvider() {
    }

    public void setProviderName(final String providerName) {
        this.providerName = providerName;
    }

    @Override
    @Intercept
    public boolean isReadable(final Class<?> type,
                              final Type genericType,
                              final Annotation[] annotations,
                              final MediaType mediaType) {
        return false;
    }

    @Override
    public String readFrom(final Class<String> type,
                           final Type genericType,
                           final Annotation[] annotations,
                           final MediaType mediaType,
                           final MultivaluedMap<String, String> headers,
                           final InputStream entityStream) throws IOException, WebApplicationException {

        return readFromAsString(entityStream, mediaType)
                + "   " + providerName + "(" + getRuntime() + "): Entity reading intercepted\n";
    }

    private String getRuntime() {
        return config.getRuntimeType() == RuntimeType.SERVER
                ? "Server" : "Client";
    }
}
{{< / highlight >}}

As you can see we&#8217;re intercepting the processing here in two places – constructor and _isReadable(&#8230;)_ method (notice the presence of the <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/Intercept.java#L13">@Intercept</a> annotation). The reason the constructor is intercepted is because we&#8217;d like to set the `providerName` field, after an instance is created, directly in the interceptor.

Then we have the _isReadable(&#8230;)_ method with default implementation that always returns false. This prevents our custom <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/providers/StringProvider.java#L32">StringProvider</a> to do something useful in _readFrom(&#8230;)_ method as it is never called if _isReadable(&#8230;)_ returns false. However, the _isReadable(&#8230;)_ method is intercepted and we can alter its behavior in our interceptor and that&#8217;s exactly what we do.

The whole implementation of <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/ProviderInterceptor.java#L13">ProviderInterceptor</a> looks like the following.

{{< highlight java "hl_lines=10-11 28" >}}
public class ProviderInterceptor implements MethodInterceptor, ConstructorInterceptor {

    @Override
    public Object construct(final ConstructorInvocation invocation) throws Throwable {
        final Object proceed = invocation.proceed();

        // After an instance of StringProvider is ready,
        // set a name to that instance.
        if (proceed instanceof StringProvider) {
            ((StringProvider) proceed)
                    .setProviderName(StringProvider.class.getSimpleName());
        }

        return proceed;
    }

    @Override
    public Object invoke(final MethodInvocation invocation) throws Throwable {
        // This method intercepts either StringProvider.isReadable
        // or StringProvider.isWriteable methods.

        // First argument of intercepted methods is class
        // of instance to be written or produced.
        final Object type = invocation.getArguments()[0];

        // Do not invoke the provider's method itself. Return true
        // if a String is about to be written or produced, false otherwise.
        return type == String.class;
    }
}
{{< / highlight >}}

## Client

Since we can intercept HK2 managed JAX-RS Providers we can affect behavior of the client to the some degree as well. If the custom <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/providers/StringProvider.java#L32">StringProvider</a> and <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/intercept/MyInterceptionBinder.java#L12">MyInterceptionBinder</a> are registered in the client the _constructor_ and _isReadable(&#8230;)_ method of the provider would be intercepted as well.

{{< highlight java "hl_lines=4-5" >}}
@Path("client")
public class ClientResource {

    @ClientConfig
    @Uri("server")
    private WebTarget server;

    /**
     * Resource method is not intercepted itself.
     *
     * In the injected client the {@code StringProvider} is registered
     * and the provider is intercepted (constructor, isReadable).
     */
    @GET
    public String get() {
        return "ClientResource: Invoke request to non-intercepted"
                + " server resource method\n"
                + "   " + server.request().get(String.class);
    }
}
{{< / highlight >}}

Lets create a client resource on the server as illustrated above. At first, a configured instance of JAX-RS <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/client/WebTarget.html">WebTarget</a> is injected into our resource and then this instance is used to retrieve a message from (non-intercepted) server resource method. Entity returned from server is used to assemble a new response returned from this <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/main/java/sk/dejavu/blog/examples/intercepting/resources/ClientResource.java#L15">ClientResource</a>&#8216;s resource method.

> Injecting configured client (web target) is a Jersey custom feature. You can find more in [Managed JAX-RS Client][1] article.

On the other side of the wire the response would look like:

{{< highlight text "hl_lines=3" >}}
ClientResource: Invoke request to non-intercepted server resource method
ServerResource: Non-intercepted method invoked
StringProvider(Client): Entity reading intercepted
{{< / highlight >}}

The StringProvider was used on the client to transform the response coming from ServerResource and the last line was added to the result. A similar output (without the first line) is expected even in case when we use JAX-RS Client API outside of an application deployed on server – see <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods/blob/master/src/test/java/sk/dejavu/blog/examples/intercepting/InterceptingTest.java#L56">InterceptingTest.clientFetchingServerResource</a> test.

## Transactions

Intercepting JAX-RS resources can be useful in situations when you need to wrap  business code present in a resource method into a JTA transaction. Rather than starting and committing (rollbacking) a transaction directly in your resource method you can abstract the code into a method interceptor and annotate the resource method with an appropriate annotation (e.g. <a href="http://docs.oracle.com/javaee/7/api/javax/transaction/Transactional.html">@Transactional</a>) to denote that the transactional processing should be started.

{{< highlight java "hl_lines=8" >}}
@Path("server")
public class ServerResource {

    /**
     * Resource method wrapped in a transaction.
     */
    @POST
    @Transactional
    public String save(final Entity entity) {
        // Do something with DB.
        // ...

        return ...;
    }
}
{{< / highlight >}}

In the method interceptor you need an instance of a <a href="http://docs.oracle.com/javaee/7/api/javax/transaction/UserTransaction.html">UserTransaction</a> (e.g. provided by application server) obtained from e.g. JNDI via <a href="http://docs.oracle.com/javase/7/docs/api/javax/naming/InitialContext.html">InitialContext</a>. The method interceptor might in this case look like:

{{< highlight java "hl_lines=8 12 15 20" >}}
public class TransactionalResourceInterceptor implements MethodInterceptor {

    @Override
    public Object invoke(final MethodInvocation methodInvocation) throws Throwable {
        final UserTransaction utx = ...;

        // Begin the transaction.
        utx.begin();

        try {
            // Invoke JAX-RS resource method.
            final Object result = methodInvocation.proceed();
            
            // Commit the transaction.
            utx.commit();
          
            return result;
        } catch (final RuntimeException re) {
            // Something bad happened, rollback;
            utx.rollback();

            // Rethrow the Exception.
            throw re;
        }
    }
}
{{< / highlight >}}

Now, for every method annotated with _@Transactional_ a separate transaction would be started before the processing actually enters the resource method and committed (rollbacked) after we return from that method.

## Further Reading

  * <a href="https://hk2.java.net/2.4.0-b08/aop-example.html">HK2 – Aspect Oriented Programming (AOP) Example</a>
  * <a href="https://hk2.java.net/2.4.0-b08/aop-default.html">HK2 – Default Interception Service Implementation</a>
  * [Managed JAX-RS Client][1]

## Resources

  * Example – <a href="https://github.com/mgajdos/jersey-intercepting-resource-methods">jersey-intercepting-resource-methods</a>

### Thanks

To <a href="https://twitter.com/japod">Jakub</a> for reading draft of this.

 [1]: /2014/01/16/managed-jax-rs-client/ "Managed JAX-RS Client"