---
title: Handling JAX-RS and Bean Validation Errors with MVC
author: Michal Gajdoš
type: post
date: 2014-01-28T09:35:12+00:00
url: /handling-jax-rs-and-bean-validation-errors-with-mvc/
aliases: /2014/01/28/handling-jax-rs-and-bean-validation-errors-with-mvc/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - mvc
  - bean-validation

---
Since JAX-RS 2.0 you can use Bean Validation to validate inputs received from users and outputs created in your application. It&#8217;s a handy feature and in most cases it works great. But what if you&#8217;re using also MVC and you want to display custom error page if something goes wrong? Or what if you want to handle Bean Validation issues in a slightly different way than JAX-RS implementation you&#8217;re using does? This article will give you some answers to these questions.

<!--more-->

Topics covered by this article:

  * [Handling Error States with Jersey MVC][1],
  * [Handling Bean Validation errors with Jersey MVC][2], and
  * [Handling Bean Validation errors in a custom way in JAX-RS 2.0][3]

## <a name="errors"></a>Handling Error States with Jersey MVC

In addition to <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/Template.html">@Template</a> annotation that has been present in Jersey MVC since 2.0 release a <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/ErrorTemplate.html">@ErrorTemplate</a> annotation has been introduced in Jersey 2.3. The purpose of this annotation is to bind the model to an error view in case an exception has been raised during processing of a request. This is true for any exception thrown after the resource matching phase (i.e. this not only applies to JAX-RS resources but providers and even Jersey runtime as well). The model in this case is the thrown exception itself.

> All code samples in this section are taken from **shortener-webapp** example that is available as <a href="https://github.com/jersey/jersey/tree/master/examples/shortener-webapp">sources</a> or as a <a href="http://jersey-shortener-webapp.herokuapp.com/">live demo</a>.

The next snippet shows how to use _@ErrorTemplate_ on a resource method. If all goes well with the method processing, then the _/short-link_ template is used to as page sent to the user. Otherwise if an exception is raised then the _/error-form_ template is shown to the user.

{{< highlight java "hl_lines=4-5" >}}
@POST
@Produces("text/html")
@Consumes(MediaType.APPLICATION_FORM_URLENCODED)
@Template(name = "/short-link")
@ErrorTemplate(name = "/error-form")
public ShortenedLink createLink(@FormParam("link") final String link) {
    // ...
}
{{< / highlight >}}

> <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/ErrorTemplate.html">@ErrorTemplate</a> can be used on a resource class or a resource method to merely handle error states. There is no need to use <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/Template.html">@Template</a> or <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/Viewable.html">Viewable</a> with it.

The annotation is handled by custom <a href="http://jax-rs-spec.java.net/nonav/2.0/apidocs/javax/ws/rs/ext/ExceptionMapper.html">ExceptionMapper<E extends Throwable></a> which creates an instance of _Viewable_ that is further processed by Jersey. This exception mapper is registered automatically with any _*MvcFeature_.

## <a name="bv-errors"></a>Handling Bean Validation Errors with Jersey MVC

_@ErrorTemplate_ can be used in also with Bean Validation to display specific error pages in case the validation of input/output values fails for some reason. Everything works as described above except the model is not the thrown exception but rather a list of <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/validation/ValidationError.html">ValidationError</a>s. This list can be iterated in the template and all the validation errors can be shown to the user in a desirable way.

Let&#8217;s add some validation annotations to our resource method from above:

{{< highlight java "hl_lines=6-7" >}}
@POST
@Produces("text/html")
@Consumes(MediaType.APPLICATION_FORM_URLENCODED)
@Template(name = "/short-link")
@ErrorTemplate(name = "/error-form")
@Valid
public ShortenedLink createLink(@ShortenLink @FormParam("link") final String link) {
    // ...
}
{{< / highlight >}}

In case the method gets an invalid form parameter (or creates invalid output for a user) we can iterate _ValidationError_s in _/error-form_ as follows (<a href="https://github.com/spullara/mustache.java">Mustache</a> templating engine):

{{< highlight xhtml "hl_lines=6-8" >}}
<div>
    <div class="panel-heading">
        <h3 class="panel-title">Errors occurred during link shortening.</h3>
    </div>
    <div class="panel-body">
        {{#.}}
            {{message}} "<strong>{{invalidValue}}</strong>"<br/>
        {{/.}}
    </div>
</div>
{{< / highlight >}}

Support for Bean Validation in Jersey MVC Templates is provided by a <a href="https://jersey.github.io/project-info/2.5.1/jersey/project/jersey-mvc-bean-validation/dependencies.html">jersey-mvc-bean-validation</a> extension module. Coordinates of this module are:

> <a href="http://repo1.maven.org/maven2/org/glassfish/jersey/ext/jersey-mvc-bean-validation/2.5.1/jersey-mvc-bean-validation-2.5.1.jar">org.glassfish.jersey.ext:jersey-mvc-bean-validation:2.5.1</a>

The JAX-RS Feature provided by this module (<a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/beanvalidation/MvcBeanValidationFeature.html">MvcBeanValidationFeature</a>) has to be registered in order to use this functionality:

{{< highlight java "hl_lines=9 13" >}}
@ApplicationPath("/")
public class ShortenerApplication extends ResourceConfig {

    public ShortenerApplication() {
        // Resources.
        packages(ShortenerResource.class.getPackage().getName());

        // Features.
        register(MvcBeanValidationFeature.class);

        // Providers.
        register(LoggingFilter.class);
        register(MustacheMvcFeature.class);

        // Properties.
        property(MustacheMvcFeature.TEMPLATE_BASE_PATH, "/mustache");
    }
}
{{< / highlight >}}

## <a name="custom"></a>Custom Handling of Bean Validation Errors in JAX-RS 2.0

In case the approach described above doesn&#8217;t work for you for some reason (i.e. you have implemented your own MVC support or you&#8217;re using other JAX-RS 2.0 implementation) you need to follow these steps to achieve similar result as with _@ErrorTemplate_:

  1. (Jersey specific) add the dependencies on Bean Validation module (_jersey-bean-validation_) and, optionally, on one of the MVC modules providing support for custom templates (_jersey-mvc-*_)
  2. create an <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ExceptionMapper.html">ExceptionMapper</a> for <a href="http://docs.jboss.org/hibernate/beanvalidation/spec/1.1/api/javax/validation/ConstraintViolationException.html">ConstraintViolationException</a>
  3. register all your providers

I&#8217;ll skip the first step as it&#8217;s easy enough and we&#8217;re going to take a look at creating custom _ExceptionMapper_ and registering it.

### Custom ExceptionMapper {#dependencies}

Bean Validation runtime throws a _ConstraintViolationException_ when something goes wrong during validating an incoming entity, request parameters or even JAX-RS resource. Every JAX-RS 2.0 implementation is required to provide a standard _ExceptionMapper_ to handle this type of exceptions (_ValidationException_ to be precise) and based on the context return correct HTTP status code (_400_ or _500_). If you want to handle Bean Validation issues differently, you need to write an _ExceptionMapper_ of your own:

{{< highlight java >}}
@Provider
@Priority(Priorities.USER)
public class ConstraintViolationExceptionMapper
                 implements ExceptionMapper<ConstraintViolationException> {

    @Override
    public Response toResponse(final ConstraintViolationException exception) {
        return Response
                // Define your own status.
                .status(400)
                // Process given Exception and set an entity
                // i.e. Set an instance of Viewable to the response
                // so that Jersey MVC support can handle it.
                .entity(new Viewable("/error", exception))
                .build();
    }
}
{{< / highlight >}}

With the _ExceptionMapper_ above you&#8217;ll be handling all thrown _ConstraintViolationException_s and the final response will have HTTP _400_ response status. Entity set (in our case <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/mvc/Viewable.html">Viewable</a>) to the response will be processed by <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/MessageBodyWriter.html">MessageBodyWriter</a> from Jersey MVC module and it will basically output a processed web-page. The first parameter of the _Viewable_ is path to a template (you can use relative or absolute path), the second is the model the MVC will use for rendering. For more on this topic, refer to the section about <a href="https://jersey.github.io/documentation/latest/mvc.html">MVC</a> in Jersey Users Guide.

### Registering Providers {#registeringproviders}

The last step you need to make is to register your providers into your <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Application.html">Application</a> (I&#8217;ll show you an example using <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ResourceConfig.html">ResourceConfig</a> from Jersey which extends _Application_ class):

{{< highlight java >}}
new ResourceConfig()
    // Look for JAX-RS reosurces and providers.
    .package("my.package")
    // Register Jersey MVC Mustache processor.
    .register(MustacheMvcFeature.class)
    // Register custom ExceptionMapper (not needed as it's annotated with @Provider).
    .register(ConstraintViolationExceptionMapper.class)
    // Register Bean Validation (this is optional as BV is automatically registered
    // when jersey-bean-validation is on the class-path but it's good to know it's happening).
    .register(ValidationFeature.class);
{{< / highlight >}}

More information about how different ways how to register providers and resources is available in this <a href="/2013/11/19/registering-resources-and-providers-in-jersey-2/">article</a>.

## Resources

Sample Application:

  * <a href="http://jersey-shortener-webapp.herokuapp.com/">jersey-shortener-webapp</a> deployed on Heroku
  * <a href="https://github.com/jersey/jersey/tree/master/examples/shortener-webapp">examples/shortener-webapp</a> sources on Github

## Further Reading

  * <a href="https://jersey.github.io/documentation/latest/mvc.html">MVC Templates</a>
  * <a href="/2013/11/19/registering-resources-and-providers-in-jersey-2/">Registering Resources and Providers in Jersey 2</a>
  * <a href="/2014/01/09/running-jersey-2-applications-on-heroku/">Running Jersey 2 Applications on Heroku</a>

 [1]: #errors
 [2]: #bv-errors
 [3]: #custom