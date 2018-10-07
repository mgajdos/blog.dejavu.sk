---
title: Jersey’s Entity Filtering meets Jackson
author: Michal Gajdoš
type: post
date: 2015-02-04T09:25:53+00:00
url: /jerseys-entity-filtering-meets-jackson/
aliases: /2015/02/04/jerseys-entity-filtering-meets-jackson/
categories:
  - Jersey
tags:
  - jersey
  - entity-filtering
  - security

---
Support for Entity Filtering in Jersey introduces a convenient facility for reducing the amount of data exchanged over the wire between client and server without a need to create specialized data view components. The main idea behind this feature is to give you means that will let you selectively filter out any non-relevant data from the model before sending the data to the other party.

Entity Data Filtering is not a new feature but so far the support was limited to MOXy JSON provider. Since Jersey 2.16, we support also <a href="https://github.com/FasterXML/jackson">Jackson</a> (2.x) JSON processor and in this article we&#8217;re going to take a look into it in more detail.

<!--more-->

## What do I need?

Assuming that you&#8217;re already using <a href="https://github.com/FasterXML/jackson">Jackson 2.x</a> support in your application via <a href="http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.glassfish.jersey.media%22%20AND%20a%3A%22jersey-media-json-jackson%22">jersey-media-json-jackson</a> module all you have to do is to update the version to at least 2.16 (-SNAPSHOT since Jersey 2.16 is not released at the time of writing this article). A new module, <a href="http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.glassfish.jersey.ext%22%20AND%20a%3A%22jersey-entity-filtering%22">jersey-entity-filtering</a>, would be transitively added to your class-path. With this setup you have all the dependencies needed to leverage the support of Entity Filtering.

Registering/Enabling Entity Filtering feature is done by [registering][1]

  * <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/jackson/JacksonFeature.html">JacksonFeature</a>
  * one of the entity filtering features (described below)

in your application, For example,

{{< highlight java "hl_lines=7-9 11" >}}
@ApplicationPath("/")
public class MyApplication extends ResourceConfig {

    public MyApplication() {
        // ...

        register(EntityFilteringFeature.class);
        // register(SecurityEntityFilteringFeature.class);
        // register(SelectableEntityFilteringFeature.class);

        register(JacksonFeature.class);
    }
}
{{< / highlight >}}

## How can I filter data with it?

Certainly, the most important question. Entity Filtering can be used in your application in three ways, via

  * [Entity Filtering Annotations][2],
  * [Security (javax.annotation.security) Annotations][3], or
  * [Query Parameters (Dynamic Filtering)][4].

Each of these ways comes with a fully working example already available on GitHub. The examples are, by default, configured to work with MOXy JSON provider but enabling Jackson instead of MOXy is very simple.

There are exactly the following lines

{{< highlight java "hl_lines=2 5" >}}
// Configure MOXy Json provider. Comment this line to use Jackson. Uncomment to use MOXy.
register(new MoxyJsonConfig().setFormattedOutput(true).resolver());

// Configure Jackson Json provider. Comment this line to use MOXy. Uncomment to use Jackson.
// register(JacksonFeature.class);
{{< / highlight >}}

in an Jersey application class present in every example. Comment the first highlighted line and uncomment the second highlighted line in the class. That&#8217;s all, now you&#8217;re using Jackson to work with JSON in your application.

### <a name="annotations"></a>Entity Filtering Annotations

> Full example available on GitHub: <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering">entity-filtering</a>

In it&#8217;s nature the Entity Filtering feature is designed in a very similar way the <a href="/2014/01/08/binding-jax-rs-providers-to-resource-methods/">Name Binding in JAX-RS</a> is. Simply create an annotation that could be attached to domain classes and resource methods (resource classes), bind these two concepts together and let the framework act upon it. Nothing too difficult. In fact, the simplest scenario consists of

  * [registering][1] Jersey&#8217;s <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/EntityFilteringFeature.html">EntityFilteringFeature</a> (and media provider) in your application
  * creating custom filtering annotation, and
  * applying this annotation to model and resource methods.

The first step is described in previous sections. The more interesting second step is to create a custom annotation and make Jersey to recognize it as entity filtering annotation. This is done by using meta-annotation <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/EntityFiltering.html">@EntityFiltering</a>.

{{< highlight java "hl_lines=4" >}}
@Target({ElementType.TYPE, ElementType.METHOD, ElementType.FIELD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
@EntityFiltering
public @interface ProjectDetailedView {

    // ...
}
{{< / highlight >}}

So far so good. And now, the third step. Use this new annotation to annotate the model &#8230;

{{< highlight java "hl_lines=9 12" >}}
public class Project {
 
    private Long id;
 
    private String name;
 
    private String description;
 
    @ProjectDetailedView
    private List<Task> tasks;
 
    @ProjectDetailedView
    private List<User> users;
 
    // getters and setters
}
{{< / highlight >}}

&#8230; and a resource class.

{{< highlight java "hl_lines=12" >}}
@Path("projects")
@Produces("application/json")
public class ProjectsResource {
 
    @GET
    public List<Project> getProjects() {
        return getDetailedProjects();
    }
 
    @GET
    @Path("detailed")
    @ProjectDetailedView
    public List<Project> getDetailedProjects() {
        return EntityStore.getProjects();
    }
}
{{< / highlight >}}

And &#8230; done! So what do we have here?

We defined two views on our _Project_ class – basic and detailed. &#8220;Basic&#8221; (or default) view is implicit and contains all non-annotated fields (_id_, _name_, _description_). Detailed view is explicit and is defined by placing _@ProjectDetailedView_ over some of the fields (classes/getters/setters/property-accessors). This view contains all of the basic view fields (since there are no other constraints defined) as well as annotated fields (_tasks_, _users_).

To bind (or assign) views together with resource methods we also use entity filtering annotations. If a resource method should return a _detailed_ view on our model (returned instance) we simply annotate the method with an entity filtering annotation (see _ProjectsResource#getDetailedProjects()_). If we don&#8217;t annotate a resource method with any entity filtering annotation then _basic_ (or default) view is assumed and only the fields tied with this view are sent over the wire.

To summarize this, even though the same instance of Project class is used to create response for both basic and detailed view the actual data sent over the wire differ for each of these two views.

Basic View on a Project (JSON representation):

{{< highlight json >}}
{
   "description" : "Jersey is the open source (under dual CDDL+GPL license) JAX-RS 2.0 (JSR 339) production quality Reference Implementation for building RESTful Web services.",
   "id" : 1,
   "name" : "Jersey"
}
{{< / highlight >}}

Detailed View on a Project (JSON representation):

{{< highlight json >}}
{
   "description" : "Jersey is the open source (under dual CDDL+GPL license) JAX-RS 2.0 (JSR 339) production quality Reference Implementation for building RESTful Web services.",
   "id" : 1,
   "name" : "Jersey",
   "tasks" : [ {
      "description" : "Entity Data Filtering",
      "id" : 1,
      "name" : "ENT_FLT"
   }, {
      "description" : "OAuth 1 + 2",
      "id" : 2,
      "name" : "OAUTH"
   } ],
   "users" : [ {
      "email" : "very@secret.com",
      "id" : 1,
      "name" : "Jersey Robot"
   } ]
}
{{< / highlight >}}

> More information about this approach is available in Jersey User Guide:
> 
>   * <a href="https://jersey.github.io/documentation/latest/entity-filtering.html#d0e14024">Components used to describe Entity Filtering concepts</a>
>   * <a href="https://jersey.github.io/documentation/latest/entity-filtering.html#ef.annotations">Using custom annotations to filter entities</a>

### <a name="security"></a>Security Annotations

> Full example available on GitHub: <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering-security">entity-filtering-security</a>

I have already written an article on <a href="/2014/02/04/filtering-jax-rs-entities-with-standard-security-annotations/">Filtering JAX-RS Entities with Standard Security Annotations</a> so if you&#8217;re interested in filtering data based on user roles, please, read <a href="/2014/02/04/filtering-jax-rs-entities-with-standard-security-annotations/">that</a> blog post.

### <a name="selectable"></a>Query Parameters (Dynamic Filtering)

> Full example available on GitHub: <a href="https://github.com/jersey/jersey/tree/master/examples/entity-filtering-selectable">entity-filtering-selectable</a>

Filtering the content sent over the wire dynamically based on query parameters is another commonly required use case. We need to register <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/EntityFilteringFeature.html">S</a>[electableEntityFilteringFeature][5] in the application and optionally define the name of the query parameter (via <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SelectableEntityFilteringFeature.html#QUERY_PARAM_NAME">SelectableEntityFilteringFeature.QUERY_PARAM_NAME</a> property) that would be used to list desired fields for further processing (marshalling).

And that&#8217;s all from the code point of view. Easy, isn&#8217;t it.

So lets rather take a look at some examples. Assume our domain model contains following classes:

{{< highlight java >}}
public class Person {
    
    private String givenName;

    private String familyName;

    private String honorificSuffix;

    private String honorificPrefix;

    // same name as Address.region
    private String region;

    private List<Address> addresses;

    private Map<String, PhoneNumber> phoneNumbers;

    // ...
}
{{< / highlight >}}

{{< highlight java >}}
public class Address {

    private String streetAddress;

    private String region;

    private PhoneNumber phoneNumber;

    // ...
}
{{< / highlight >}}

{{< highlight java >}}
public class PhoneNumber {

    private String areaCode;

    private String number;

    // ...
}
{{< / highlight >}}

The resource class is very simple as well:

{{< highlight java >}}
@Path("people")
@Produces("application/json")
public class PersonResource {

    @GET
    @Path("{id}")
    public Person getPerson() {
        // ...
        return person;
    }
}
{{< / highlight >}}

As you can see no entity filtering annotations are used this time and even though the same instance of Person class is used to create response the given values of `select` query parameter are used to select the fields that would be transferred over the wire.

For `people/1234?select=familyName,givenName` the response looks like:

{{< highlight json >}}
{
   "familyName": "Dowd",
   "givenName": "Andrew"
}
{{< / highlight >}}

And for `people/1234?select=familyName,givenName,addresses.phoneNumber.number` the response would be:

{{< highlight json >}}
{
   "addresses":[
      {
         "phoneNumber":{
            "number": "867-5309"
         }
      }
   ],
   "familyName": "Dowd",
   "givenName": "Andrew"
}
{{< / highlight >}}

> More information about this approach is available in Jersey User Guide:
> 
>   * <a href="https://jersey.github.io/documentation/latest/entity-filtering.html#ef.selectable.annotations">Entity Filtering based on dynamic and configurable query parameters</a>

Hope you’ll enjoy the Entity Data Filtering feature with Jackson JSON provider and if you have any comments or suggestions, please comment in the discussion below or send me a note at <michal@dejavu.sk>.

 [1]: /2013/11/19/registering-resources-and-providers-in-jersey-2/ "Registering Resources and Providers in Jersey 2"
 [2]: #annotations
 [3]: #security
 [4]: #selectable
 [5]: https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/message/filtering/SelectableEntityFilteringFeature.html