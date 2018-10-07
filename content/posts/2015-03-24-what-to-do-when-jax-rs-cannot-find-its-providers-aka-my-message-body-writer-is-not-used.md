---
title: What to do when JAX-RS cannot find it’s Providers aka My message body writer is not used
author: Michal Gajdoš
type: post
date: 2015-03-24T09:16:52+00:00
url: /what-to-do-when-jax-rs-cannot-find-its-providers-aka-my-message-body-writer-is-not-used/
aliases: /2015/03/24/what-to-do-when-jax-rs-cannot-find-its-providers-aka-my-message-body-writer-is-not-used/
categories:
  - Jersey
tags:
  - jax-rs

---
One of the reasons why I wrote this article is my, rather unusual, encounter with a missing message body writer in a Jersey 1 test application. This problem of mine occurred during collecting of data to get a Jersey 1 performance baseline in order to see where Jersey 2 stands when compared to its predecessor. The problem is described at the end of this article because I think it&#8217;s pretty rare, but still not impossible :-), and not many of you will (hopefully) face it. But first I&#8217;d like to take a look at more common issues regarding missing providers and what you can do to solve them.

<!--more-->

The first thing to try when your application doesn&#8217;t behave as it should is to turn on logging (on CONFIG level at least) to get more information from the underlying JAX-RS implementation (in case you use Jersey, take a look at this <a href="http://yatel.kramolis.cz/2013/11/how-to-configure-jdk-logging-for-jersey.html">post</a>). For example, Jersey would print out all registered resources and providers which may show whether there is or isn&#8217;t problem in your configuration very quickly.

When you find out that your provider (or resource) is registered and there is still something wrong with your application, try registering <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/filter/LoggingFilter.html">LoggingFilter</a> or using <a href="https://jersey.github.io/documentation/latest/monitoring_tracing.html#tracing">Tracing Support</a>. However, if you realize that your application knows nothing about the provider you need then the solution may fall into one of these categories:

### Class-path scanning – @Provider (JAX-RS)

If you rely on <a href="https://blogs.oracle.com/japod/entry/when_to_use_jax_rs">class-path scanning</a> during deployment of your application make sure the classes are visible (have appropriate constructors) and the providers are annotated with <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/ext/Provider.html">@Provider</a> annotation otherwise they won&#8217;t be discovered (<a href="https://github.com/jersey/jersey/blob/master/examples/flight-mgmt-webapp/src/main/java/org/glassfish/jersey/examples/flight/providers/ContainerAuthFilter.java#L67">example</a>).

### No class-path scanning – Application (JAX-RS)

In case you&#8217;re configuring your application via <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Application.html">Application</a> subclass (or via proprietary mechanism provided by JAX-RS implementation you use, e.g. <a href="/2013/11/19/registering-resources-and-providers-in-jersey-2/">ResourceConfig</a>) be sure your providers are registered using

  * <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Application.html#getClasses()">getClasses()</a> – <a href="https://github.com/jersey/jersey/blob/master/examples/helloworld-pure-jax-rs/src/main/java/org/glassfish/jersey/examples/helloworld/jaxrs/JaxRsApplication.java#L64">example</a>,
  * <a href="https://jax-rs.github.io/apidocs/2.0.1/javax/ws/rs/core/Application.html#getSingletons()">getSingletons()</a> – <a href="https://github.com/jersey/jersey/blob/master/examples/server-async-standalone/webapp/src/main/java/org/glassfish/jersey/examples/server/async/AsyncJaxrsApplication.java#L69">example</a>, or
  * <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ResourceConfig.html#register(java.lang.Class)">register(&#8230;)</a> (Jersey) – <a href="https://github.com/jersey/jersey/blob/master/examples/json-processing-webapp/src/main/java/org/glassfish/jersey/examples/jsonp/MyApplication.java#L54">example</a>.

### Service Lookup – META-INF/services (JAX-RS)

Some 3<sup>rd</sup> party libraries (e.g. Jackson 2) may use META-INF/services lookup mechanism to register their providers in JAX-RS runtime. In this case make sure the Jars are really on the class-path. If they are and you still experiencing troubles look at the _Note_ below (for Jersey 2), or _My Case_ at the end.

_Note:_ Jersey 2 doesn&#8217;t automatically pick-up providers registered via META-INF/services for quite some time now. If you want to use those providers either register them manually via Application (see above) or add <a href="http://search.maven.org/#search%7Cga%7C1%7C%22jersey-metainf-services%22">org.glassfish.jersey.ext:jersey-metainf-services</a> extension module to your application. The module enables automatic registration of JAX-RS providers (MBW/MBR/EM) via META-INF/services mechanism.

### Uber JAR (JAX-RS)

When an (executable) über jar is your way of deploying/running applications you may encounter an exception like:

```text
SEVERE: com.sun.jersey.api.MessageException: A message body writer for Java class java.lang.String, and Java type class java.lang.String, and MIME media type text/plain was not found

SEVERE: The registered message body writers compatible with the MIME media type
are:
*/* ->
  com.sun.jersey.server.impl.template.ViewableMessageBodyWriter
```

When you see an exception like this (I mean a provider for String, required by JAX-RS itself, cannot be found) you know there&#8217;s something really wrong with your application.

The reason is you&#8217;re creating your über jar using mis-configured assembly plugin (e.g. from Maven) that, by default, handles META-INF/services incorrectly. Files in META-INF/services are not CONCATENATED as they should be, they&#8217;re overwritten by each other. This means that if you have same services (e.g. MessageBodyWriter) in multiple Jars then only services from one (the last one) Jar are registered.

Solution to this problem is to either use prepared &#8220;all-in-one&#8221; <a href="http://stackoverflow.com/a/12622037/290799">bundles</a> of JAX-RS implementation of choice or use plugin that concatenates META-INF/services files instead of overwriting them (e.g. <a href="http://search.maven.org/#search%7Cgav%7C1%7Cg%3A%22org.apache.maven.plugins%22%20AND%20a%3A%22maven-shade-plugin%22">maven-shade-plugin</a>).

Note: This problem is uncommon when using Jersey 2.

### LC\_ALL and LC\_CTYPE – My Case

Our performance <a href="https://github.com/jersey/jersey/tree/master/tests/performance/test-cases">test-cases</a> are simple applications designed to test features of JAX-RS 2.0 and Jersey 2. Some of them are JAX-RS 1.1 compliant and we decided to run them using Jersey 1 to be able to compare performance of major versions of Jersey. To do so we have a hudson job that starts one test-case per time on a &#8220;server&#8221; node (via ssh) and it also starts wrk clients on two &#8220;client&#8221; nodes (also via ssh). No mysteries here.

I selected test-cases to be measured and using this hudson job tried that the Jersey 1 runner was working as expected (on my machine). Then I tried to run the tests on our infrastructure expecting no problems. The first run created 750MB of logs that looked like:

```text
SEVERE: com.sun.jersey.api.MessageException: A message body writer for Java class java.lang.String, and Java type class java.lang.String, and MIME media type text/plain was not found

SEVERE: The registered message body writers compatible with the MIME media type
are:
*/* ->
  com.sun.jersey.server.impl.template.ViewableMessageBodyWriter
```

Exactly as in the &#8220;Uber JAR&#8221; case. Except the test-cases wasn&#8217;t deployed as one big fat Jar. It worked on my machine, it didn&#8217;t on the remote one. I tried to copy binaries built on my machine (they worked, right?) to the remote one and ran the case. It didn&#8217;t work. Then I tried to use different versions of JDK (maybe something is broken there). It didn&#8217;t work either.

&#8220;Awesome&#8221;.

I put few debug logs into ServiceFinder (class that loads services/provider from META-INF/services in Jersey) to find out what&#8217;s going on. ServiceFinder loads META-INF/services files using <a href="http://docs.oracle.com/javase/7/docs/api/java/lang/ClassLoader.html#getResources(java.lang.String)">Classloader#getResources(String)</a> method. When executed it looked like only one META-INF/service resource file for a particular service was loaded from all Jars present on my class-path. The first that was found. Others were ignored even though they were there as well. And I got no ideas how to solve that.

It worked on my machine. It didn&#8217;t work when hudson started the server via ssh on a separate node. And it didn&#8217;t work when I tried to start the server on the same node via ssh.

Out of desperation I asked <a href="https://twitter.com/pavel_bucek">Pavel</a> to try the test-case on his machine. It worked as it had worked on mine. Then he tried to connect to the remote machine and start the test-case there. It **worked**. _Wot?!_ I tried it again from my machine. And &#8230; it didn&#8217;t work.

This was interesting. We compared environment variables when connected to the remote machine and there was one difference. Value of **LC_CTYPE**. My terminal (iTerm2) doesn&#8217;t set this variable on remote machine but Pavel&#8217;s terminal (OS X Terminal) does. This was it. I spent few hours trying to figure out what the problem was and in the end the remedy to the problem was so stupidly simple. Add the following line to `.bashrc` on the hudson slave that invokes *nix scripts on remote machines:

```bash
export LC_ALL=UTF-8
```

**Conclusion:** When class loader loads resources stored in Jars present on your class-path the result depend on value of the **LC_CTYPE** environment variable (*nix boxes). So, when you ssh&#8217;d to a remote host make sure you have correctly set **LC_ALL** or **LC_CTYPE** because it does matter!

Note: The value of this variable on the remote host is set from the value that is set on the machine you&#8217;re trying to connect to the remote host from. Keep this is mind.