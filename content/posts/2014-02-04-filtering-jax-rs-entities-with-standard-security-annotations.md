---
title: Filtering JAX-RS Entities with Standard Security Annotations
author: Michal Gajdoš
type: post
date: 2014-02-04T09:35:13+00:00
url: /filtering-jax-rs-entities-with-standard-security-annotations/
aliases: /2014/02/04/filtering-jax-rs-entities-with-standard-security-annotations/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - client
  - entity-filtering
  - security

---
Few times I&#8217;ve ran into a situation in which I wanted to return only a part of entity I was working with, in my service, to the client-side. Feature that would allow me to send this partial set of fields should not only be driven by my own filtering rules but it also should take into consideration application roles the current user is in. It&#8217;s not an uncommon use-case I&#8217;d say so in Jersey we&#8217;d come up with the Entity Filtering concept that would allow you to do such a thing: sending subgraph of entities from server to client (and vice versa), define your own filtering rules and make sure users get only the fields they are supposed to see.

<!--more-->

You can, of course, define role-based access rules to JAX-RS resources to protect particular method, return specific DTO or filter the entity manually directly in the method. Both Jersey 1 and Jersey 2 allow you to make sure that particular resource method is being invoked only by users in certain user roles. There are multiple approaches to achieve this including annotating your resources and resource methods with standard security annotations from _javax.annotation.security_ package (see <a href="https://jersey.github.io/documentation/latest/user-guide.html#d0e10183">Authorization &#8211; securing resources</a> for example). However, for entities there wasn&#8217;t such an easy and straightforward approach to select and return only a part of entity.

<a href="https://twitter.com/kingsfleet‎">Gerard</a> covered the general concepts of Entity Filtering in his _Selecting level of detail returned by varying the content type_ (<a href="http://kingsfleet.blogspot.co.uk/2014/01/selecting-level-of-detail-returned-by.html">part I</a>, <a href="http://kingsfleet.blogspot.co.uk/2014/01/selecting-level-of-detail-returned-by_29.html">part II</a>) articles and in my article I&#8217;d like to focus on filtering entities based on standard security annotations from _javax.annotation.security_ package.

Examples showing these concepts are available in Jersey workspace (run them with _\`mvn compile exec:java\`_):

  * <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering">entity-filtering</a>
  * <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering-security">entity-filtering-security</a>

## Idea

Support for Entity Filtering in Jersey introduces a convenient facility for reducing the amount of data exchanged over the wire between client and server without a need to create specialized data view components. The main idea behind this feature is to give you APIs that will let you to selectively filter out any non-relevant data from the marshalled object model before sending the data to the other party based on the context of the particular message exchange. This way, only the necessary or relevant portion of the data is transferred over the network with each client request or server response, without a need to create special facade models for transferring these limited subsets of the model data.

Entity Filtering feature allows you to define your own entity-filtering rules for your entity classes based on the current context (e.g. matched resource method) and keep these rules in one place (directly in your domain model). It is also possible to assign security access rules to entity classes properties and property accessors.

## Role-based Entity Filtering

Filtering the content sent to the client (or server) based on the authorized security roles is a commonly required use case. By registering <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SecurityEntityFilteringFeature.html">SecurityEntityFilteringFeature</a> you can leverage the Jersey Entity Filtering facility in connection with standard _javax.annotation.security_ annotations plus you can define access rules to the resource methods. Supported security annotations are:

  * <a href="http://docs.oracle.com/javaee/6/api/javax/annotation/security/PermitAll.html">@PermitAll</a>,
  * <a href="http://docs.oracle.com/javaee/6/api/javax/annotation/security/RolesAllowed.html">@RolesAllowed</a>, and
  * <a href="http://docs.oracle.com/javaee/6/api/javax/annotation/security/DenyAll.html">@DenyAll</a>

Let&#8217;s assume we have a simple project-tracking JAX-RS application consisting of a few classes in domain model and a few resources. Back-end sends JSON messages (containing entities) to front-end and front-end displays web-pages with this data accordingly. Important thing is that you don&#8217;t need all the data from the server to correctly display pages to the users. Similarly you don&#8217;t want to send all the existing data connected with the entity to the users that are not supposed to see them. We define three user roles in our application: _user_, _manager_ and _client_.

I&#8217;ll illustrate the concept on the _Project_ entity from the domain model:

{{< highlight java >}}
public class Project {

    public Long getId() { ... }

    public String getName() { ... }

    public String getDescription() { ... }

    public String getSecret() { ... }

    public List<Task> getTasks() { ... }

    public List<User> getUsers() { ... }

    public List<Report> getReports() { ... }

    // fields and setters
}
{{< / highlight >}}

By defining the following access rules:

  * _secret_ is not visible to anyone and won&#8217;t be sent to the other party (it&#8217;s a custom back-end property)
  * _tasks_ are visible to managers and users
  * _users_ are visible only to managers
  * _reports_ are visible to managers and clients
  * other fields (_id_, _name_, _description_) are visible to anyone

We have a clear image how the annotated _Project_ entity should look like:

{{< highlight java "hl_lines=9 12 15 18" >}}
public class Project {

    public Long getId() { ... }

    public String getName() { ... }

    public String getDescription() { ... }

    @DenyAll
    public String getSecret() { ... }

    @RolesAllowed({"manager", "user"}) 
    public List<Task> getTasks() { ... }

    @RolesAllowed({"manager"})
    public List<User> getUsers() { ... }

    @RolesAllowed({"manager", "client"})
    public List<Report> getReports() { ... }

    // fields and setters
}
{{< / highlight >}}

Non-annotated properties or property accessors are present in the result &#8220;by default&#8221;. This means that _id_, _name_ and _description_ (not affected by any filtering annotation) will be returned every-time regardless the security role the current user is in. The processing of this fields is the same as if you put _@PermitAll_ on the class or on the property accessors.

Based on the annotations the returned entity will look different for each user role:

  * **manager** will see _id_, _name_, _description_, _tasks_, _users_ and _reports_
  * **user** will see _id_, _name_, _description_ and _tasks_
  * **client** will see _id_, _name_, _description_ and _reports_

> Entity Filtering is transitive which means that you can define access rules on sub-entities (i.e. Reports) as well. Different user roles will get different results for the whole entity graph.

We do not need any other restrictions to be defined at the moment so let&#8217;s take a look how the Entity Filtering works on the server and on the client.

## Server

Instead of creating one resource method per user-role (+ special DTO classes), creating various <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/ReaderInterceptor.html">Reader</a>/<a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/WriterInterceptor.html">Writer</a> interceptors or using another 3rd party library with Entity Filtering it&#8217;s enough to create a simple resource class like:

{{< highlight java "hl_lines=6" >}}
@Path("projects")
@Produces("application/json")
public class ProjectsResource {

    @POST
    @RolesAllowed("manager")
    public Project createProject(final Project project) {
        ...
    }

    @GET
    @Path("{id}")
    public Project getProject(@PathParam("id") final Long id) {
        ...
    }

    @GET
    public List<Project> getProjects() {
        ...
    }
}
{{< / highlight >}}

Jersey will take care of filtering fields as defined by security annotations.

> There is no need to annotate both resource method and domain entity with security annotations if you only want to restrict access to entity fields. Annotate only setters/getters of the entity and entity-filtering would work as expected even in this case.

Take a look at the resource one more time and you&#8217;ll see that only _createProject_ is annotated with security annotation. This method is intended to be invoked only by users in role _manager_ hence the annotation is needed to prevent unwanted calls from unprivileged users. Other two methods (_getProject_, _getProjects_) can be invoked by anyone yet entity-filtering rules on the returned entity will be still applied.

There are two mandatory steps needed to be done to make Entity Filtering work. The first one is to set the correct <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/SecurityContext.html">SecurityContext</a> in your application. If you&#8217;re running your application in a web-container that allows you to configure security constraints to deployed applications (i.e. enable BASIC auth) and you configured them, the _SecurityContext_ is set and you don&#8217;t need to do anything else. Otherwise you need to set it yourself in a request filter:

{{< highlight java >}}
@Provider
@PreMatching
public class SecurityRequestFilter implements ContainerRequestFilter {

    @Override
    public void filter(final ContainerRequestContext requestContext)
                throws IOException {
        requestContext.setSecurityContext(new SecurityContext() {

            @Override
            public Principal getUserPrincipal() {
                ...
            }

            @Override
            public boolean isUserInRole(final String role) {
                ...
            }

            @Override
            public boolean isSecure() {
                ...
            }

            @Override
            public String getAuthenticationScheme() {
                ...
            }
        });
    }
}
{{< / highlight >}}

The second step is to register <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SecurityEntityFilteringFeature.html">SecurityEntityFilteringFeature</a>:

{{< highlight java "hl_lines=6-7 10" >}}
@ApplicationPath("/")
public class MyApplication extends ResourceConfig {

    public MyApplication() {
        // Set entity-filtering scope via configuration.
        property(EntityFilteringFeature.ENTITY_FILTERING_SCOPE,
                    new Annotation[] {SecurityAnnotations.rolesAllowed("manager")});

        // Register the SecurityEntityFilteringFeature.
        register(SecurityEntityFilteringFeature.class);

        // Further configuration of ResourceConfig.
        register( ... );
    }
}
{{< / highlight >}}

Now everything should be set and ready to use entity-filtering.

#### Other ways to pass entity-filtering user roles to Jersey

There are also other ways than annotating resources to tell Jersey which entity-filtering scopes (in this case user roles) should be used when processing an entity.

You can see one of them in _MyApplication_ class above. It&#8217;s possible to set a property (<a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/EntityFilteringFeature.html#ENTITY_FILTERING_SCOPE">EntityFilteringFeature.ENTITY_FILTERING_SCOPE</a>) that defines entity-filtering scope (user role) for every request/response that is processed in the application (note: user still needs to be in roles defined by this property otherwise the role is ignored).

> Note that values of entity-filtering scope property are annotations. Since it&#8217;s not very easy to create an instance of an annotation in Java a <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SecurityAnnotations.html">SecurityAnnotations</a> helper class is available for you to create them when needed.

If you&#8217;d rather define &#8220;per-response&#8221; scope it is possible to provide entity-filtering scope annotations in the response, along with the entity:

{{< highlight java "hl_lines=10" >}}
@Path("projects")
@Produces("application/json")
public class ProjectsResource {

    @GET
    public Response getProjects() {
        return Response
                .ok()
                .entity(new GenericEntity<List<Project>>(projects) {},
                     new Annotation[]{SecurityAnnotations.rolesAllowed("manager")})
                .build();
    }
}
{{< / highlight >}}

> When the multiple approaches are combined, the priorities of calculating the applied scopes are as follows: `Entity annotations in request or response` > `Property passed through Configuration` > `Annotations applied to a resource method or class`.

## (Managed) Client

Entity Filtering is not restricted to be used only on server (resources / resource methods) but it can also be used with JAX-RS clients or <a href="/2014/01/16/managed-jax-rs-client/">injected web targets</a>. Similarly to the way we configured our JAX-RS application, we need to register <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SecurityEntityFilteringFeature.html">SecurityEntityFilteringFeature</a> in order to filter entities on client:

{{< highlight java "hl_lines=3 6" >}}
final ClientConfig config = new ClientConfig()
    // (optional) Set entity-filtering scope via configuration.
    .property(EntityFilteringFeature.ENTITY_FILTERING_SCOPE,
                  new Annotation[] {SecurityAnnotations.rolesAllowed("manager")})
    // Register the SecurityEntityFilteringFeature.
    .register(SecurityEntityFilteringFeature.class)
    // Further configuration of ClientConfig.
    .register( ... );

// Create new client.
final Client client = ClientClientBuilder.newClient(config);

// Use the client.
{{< / highlight >}}

Obviously there is one big difference between clients and server. Clients cannot infer user roles from _SecurityContext_ so you need to provide them in some other way. You can set either <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/EntityFilteringFeature.html#ENTITY_FILTERING_SCOPE">EntityFilteringFeature.ENTITY_FILTERING_SCOPE</a> property in _ClientConfig_ to apply entity-filtering scope to all incoming/outgoing entities (see example above) or you can set roles directly when creating requests:

{{< highlight java "hl_lines=5" >}}
// Use the client.
client.target(uri)
    .request()
    .post(Entity.entity(project,
             new Annotation[] {SecurityAnnotations.rolesAllowed("user")}));
{{< / highlight >}}

The _project_ entity instance will contain all fields allowed for _user_ security role when sent to the other party.

Besides the manually created client instances Jersey allows you to use injected _WebTarget_ instances in your resources (as described in <a href="/2014/01/16/managed-jax-rs-client/">Managed JAX-RS Client</a>) and you can use entity-filtering in them as well:

{{< highlight java "hl_lines=5-6 12" >}}
@Path("client")
@Produces("application/json")
public class ClientResource {

    @Uri("projects")
    private WebTarget target;

    @GET
    public List<Project> getProjects() {
        return target.request()
            .post(Entity.entity(project,
                      new Annotation[] {SecurityAnnotations.rolesAllowed("user")}));
    }
}
{{< / highlight >}}

## Limitations and Next Steps

Even though Entity Filtering is a universal feature and is not hard-wired with any media type or provider we currently support only JSON via <a href="http://www.eclipse.org/eclipselink/moxy.php">MOXy</a>. We understand that this is not enough and we plan to bring support also for <del>JSON via <a href="https://github.com/FasterXML/jackson">Jackson</a> and</del> XML via <a href="http://www.eclipse.org/eclipselink/moxy.php">MOXy</a>. JSON support via Jackson (2.x) provider is available since Jersey 2.16.

If you feel that something is missing or not working as expected we&#8217;d be happy for any comments or suggestions in the discussion below or in our <a href="https://jersey.github.io/mailing.html">mailing list</a>.

## Examples

  * <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering">entity-filtering</a>
  * <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering-security">entity-filtering-security</a>

## Further Reading

  * <a href="https://jersey.github.io/documentation/latest/entity-filtering.html">Jersey User Guide, Chapter 17. Entity Data Filtering</a>
  * [MOXy&#8217;s Object Graphs &#8211; Input/Output Partial Models to XML & JSON][1]
  * [MOXy&#8217;s Object Graphs &#8211; Partial Models on the Fly to/from XML & JSON][2]
  * <a href="http://kingsfleet.blogspot.co.uk/2014/01/selecting-level-of-detail-returned-by.html">Selecting level of detail returned by varying the content type</a>
  * <a href="http://kingsfleet.blogspot.co.uk/2014/01/selecting-level-of-detail-returned-by_29.html">Selecting level of detail returned by varying the content type, part II</a>

 [1]: http://blog.bdoughan.com/2013/03/moxys-object-graphs-inputoutput-partial.html
 [2]: http://blog.bdoughan.com/2013/03/moxys-object-graphs-partial-models-on.html