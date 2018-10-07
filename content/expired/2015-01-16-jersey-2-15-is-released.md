---
title: Jersey 2.15 is released
author: Michal Gajdoš
type: post
date: 2015-01-16T13:01:43+00:00
expirydate: 2018-09-01T00:00:00+00:00
url: /jersey-2-15-is-released/
aliases: /2015/01/16/jersey-2-15-is-released/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - release

---
We have released <a href="https://jersey.java.net/">Jersey</a> 2.15 – Reference Implementation of JAX-RS 2.0 (Java API for RESTful Web Services). The JAX-RS 2.0 <a href="https://jcp.org/aboutJava/communityprocess/mrel/jsr339/index.html">specification</a> is available at the JCP web site.

In this article I’d like to introduce some of the new features, important bug fixes and other noteworthy changes.

<!--more-->

> This article describe notable changes in Jersey 2.15. Refer to [Latest Jersey Versions][1] to find out information and links to the latest version released.

For an overview of all JAX-RS features read the Jersey <a href="https://jersey.java.net/documentation/2.15/index.html">user guide</a>. To get started with Jersey read the <a href="https://jersey.java.net/documentation/2.15/getting-started.html">getting started section</a> of that guide. To understand more about what Jersey depends on read the <a href="https://jersey.java.net/documentation/2.15/modules-and-dependencies.html">dependencies section</a>. To see the full change log, take a look <a href="https://github.com/jersey/jersey/releases/tag/2.15">here</a>.

## Container Agnostic CDI Support

<a href="https://twitter.com/japod">Jakub</a> submitted a new feature that allows you to run a Jersey/CDI application outside of a Java EE container (e.g. GlassFish). Now it&#8217;s possible to run your application on a lightweight HTTP container (e.g. Grizzly or Netty) or a web-container (e.g. Jetty, Tomcat) while leveraging the benefits of CDI.

There are two new examples available in Jersey code-base:

  * <a href="https://github.com/jersey/jersey/tree/2.15/examples/helloworld-weld">helloworld-weld</a> – uses Grizzly HTTP server, and
  * <a href="https://github.com/jersey/jersey/tree/2.15/examples/cdi-webapp">cdi-webapp</a> – illustrates Apache Tomcat Server deployments

This feature, however, is a breaking change (so be aware when upgrading to 2.15 from an older version). Some old modules providing CDI support have been removed and replaced with new, more general ones.

If you have a direct reference to the following Jersey CDI support module in your application, for example:

{{< highlight xml "hl_lines=2-4" >}}
<dependency>
  <groupId>org.glassfish.jersey.containers.glassfish</groupId>
  <artifactId>jersey-gf-cdi</artifactId>
  <version>${pre-2.15-version}</version>
</dependency>
{{< / highlight >}}

Replace it with this module:

{{< highlight xml "hl_lines=2-3" >}}
<dependency>
  <groupId>org.glassfish.jersey.ext.cdi</groupId>
  <artifactId>jersey-cdi1x</artifactId>
  <version>2.15</version>
</dependency>
{{< / highlight >}}

And, if you want to leverage CDI JTA support the following needs to be included in addition to the module above:

{{< highlight xml "hl_lines=2-3" >}}
<dependency>
  <groupId>org.glassfish.jersey.ext.cdi</groupId>
  <artifactId>jersey-cdi1x-transaction</artifactId>
  <version>2.15</version>
</dependency>
{{< / highlight >}}

## Reactive Jersey Client Documentation

I was finally able to finish documentation for _Reactive Jersey Client_, <a href="https://java.net/jira/browse/JERSEY-2639">feature</a> introduced in Jersey 2.13. The documentation can be found in the Jersey User Guide, <a href="https://jersey.java.net/documentation/latest/rx-client.html">Reactive Jersey Client API</a> chapter, or you can find out more about the reactive client in the following blog posts:

  * [Reactive Jersey Client, Part 1 – Motivation][2]
  * [Reactive Jersey Client, Part 2 – Usage and Supported Libraries][3]
  * [Reactive Jersey Client, Part 3 – Customization][4]

In addition to the documentation I have submitted a new sample <a href="https://github.com/jersey/jersey/tree/2.15/examples/rx-client-java8-webapp">rx-client-java8-webapp</a> that runs on Java 8 and compares JAX-RS synchronous and asynchronous approaches with reactive approaches using <a href="http://reactivex.io/RxJava/javadoc//rx/Observable.html">Observable</a>, <a href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletionStage.html">CompletionStage</a> (<a href="http://docs.oracle.com/javase/8/docs/api/java/util/concurrent/CompletableFuture.html">CompletableFuture</a>) and <a href="http://docs.guava-libraries.googlecode.com/git-history/v14.0.1/javadoc/com/google/common/util/concurrent/ListenableFuture.html">ListenableFuture</a>.

## Bugs

  * Thanks to <a href="https://github.com/vetler">Vetle</a> the JAX-RS providers (e.g. [ContainerRequestFilter][5]) managed by Spring are now <a href="https://java.net/jira/browse/JERSEY-2651">correctly</a> registered and invoked in a JAX-RS application. (<a href="https://github.com/jersey/jersey/pull/108">Pull Request #108</a>)
  * <a href="https://github.com/oscarguindzberg">Oscar</a> fixed a problem with Bean Validation in Jersey. Now it&#8217;s possible to put BV constraints on primitive data arrays (e.g. <a href="http://docs.oracle.com/javaee/7/api/javax/validation/constraints/NotNull.html">@NotNull</a>) in your beans. (<a href="https://github.com/jersey/jersey/pull/119">Pull Request #119</a>)
  * Closing <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ChunkedOutput.html">ChunkedOutput</a> on the server now correctly closes also the underlying connection even if the <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/ChunkedOutput.html#close()">close</a> method is invoked before returning a chunked output instance from a resource method. Problem could be encountered only when an application was deployed on a servlet container.

{{< highlight java "hl_lines=21 34" >}}
@GET
public ChunkedOutput closeBeforeReturn() {
    final ChunkedOutput output = new ChunkedOutput<>(Message.class, "\r\n");
    final CountDownLatch latch = new CountDownLatch(1);

    new Thread() {
        @Override
        public void run() {
            try {
                for (int i = 0; i < 3; i++) {
                    output.write(new Message(i, "test"));
                    Thread.sleep(200);
                }
            } catch (final IOException e) {
                LOGGER.log(Level.SEVERE, "Error writing chunk.", e);
            } catch (final InterruptedException e) {
                LOGGER.log(Level.SEVERE, "Sleep interrupted.", e);
                Thread.currentThread().interrupt();
            } finally {
                try {
                    output.close();
                    // Worker thread can continue.
                    latch.countDown();
                } catch (final IOException e) {
                    LOGGER.log(Level.INFO, "Error closing chunked output.", e);
                }
            }
        }
    }.start();

    try {
        // Wait till new thread closes the chunked output.
        latch.await();
        return output;
    } catch (final InterruptedException e) {
        throw new InternalServerErrorException(e);
    }
}
{{< / highlight >}}

  * A <a href="https://java.net/jira/browse/JERSEY-2721">problem</a> in Jersey Proxy Client was preventing to send a request with an entity if a parameter of a proxied resource method was annotated (e.g. with <a href="http://docs.oracle.com/javaee/7/api/javax/validation/constraints/NotNull.html">@NotNull</a>). The bug has been fixed thanks to <a href="https://github.com/walec51">Adam</a>. (<a href="https://github.com/jersey/jersey/pull/129">Pull Request #129</a>)

## Feedback

For feedback send an email to:

> <users@jersey.java.net> (archived [here][6])

or log bugs/features [here][7].

### _Request_

Since there are no articles like this one for Jersey 2.7–2.14 let me know (either in discussion below or via email that can be found on my <a href="https://github.com/mgajdos">GitHub profile</a>) if you&#8217;d like me to summarize notable changes in those versions as well.

 [1]: /latest-jersey-version/ "Latest Jersey Versions"
 [2]: /2015/01/07/reactive-jersey-client-part-1-motivation/ "Reactive Jersey Client, Part 1 – Motivation"
 [3]: /2015/01/07/reactive-jersey-client-part-2-usage-and-supported-reactive-libraries/ "Reactive Jersey Client, Part 2 – Usage and Supported Libraries"
 [4]: /2015/01/07/reactive-jersey-client-part-3-customization/ "Reactive Jersey Client, Part 3 – Customization"
 [5]: https://jersey.github.io/apidocs/latest/jersey/javax/ws/rs/container/ContainerRequestFilter.html
 [6]: http://markmail.org/search/?q=list%3Anet.java.dev.jersey.users
 [7]: http://java.net/jira/browse/JERSEY/