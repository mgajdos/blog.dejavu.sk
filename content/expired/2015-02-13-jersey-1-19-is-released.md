---
title: Jersey 1.19 is released
author: Michal Gajdoš
type: post
date: 2015-02-13T14:53:07+00:00
expirydate: 2018-09-01T00:00:00+00:00
url: /jersey-1-19-is-released/
aliases: /2015/02/13/jersey-1-19-is-released/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - release

---
We have released [Jersey][1] 1.19, the open source, production quality, reference implementation of JAX-RS 1.1. The JAX-RS 1.1 [specification][2] is available at the JCP web site and also available in non-normative HTML [here][3].

For an overview of JAX-RS features read the Jersey <a href="https://jersey.java.net/documentation/1.19/user-guide.html">user guide</a>. To get started with Jersey read the [getting started section][4] of that guide. To understand more about what Jersey depends on read the [dependencies section][5] of that guide. See change log <a href="https://github.com/jersey/jersey-1.x/releases/tag/1.19">here</a>.

<!--more-->

> This article describes changes in Jersey **1.19**, Reference Implementation of JAX-RS **1.1**. If you&#8217;re looking for JAX-RS **2.0** Reference Implementation refer to the [Jersey 2.15 is released][6] article or [Latest Jersey Versions][7].

## Jersey 1.x and JDK8

Probably the most anticipated change among all. <a href="https://github.com/cowwoc">Gili</a> provided a <a href="https://github.com/jersey/jersey-1.x/pull/13">Pull Request</a> which brings JDK 8 support for Jersey 1.x applications. The change updated (internally repackaged) ASM library to version 5 used on server for package scanning for JAX-RS resources and providers. Thanks!

## JAX-RS 1.1 classes no longer in Jersey Core

This is a possible breaking change when upgrading from older (<= 1.18.4) versions of Jersey to the latest one. We decided to remove bundled JAX-RS 1.1 classes from Jersey Core module.

If you&#8217;re not using a dependency manager (e.g. Maven) in your projects you need to manually add JAX-RS 1.1 APIs to your class-path. The library is available for download on Maven Central under the following coordinates:

<p style="padding-left: 30px;">
  <a href="http://search.maven.org/#artifactdetails%7Cjavax.ws.rs%7Cjsr311-api%7C1.1.1%7Cjar">javax.ws.rs:jsr311-api:1.1.1</a>
</p>

If you&#8217;re running your Jersey 1.x application in OSGi don&#8217;t forget to register the _jsr311-api_ module as well to satisfy dependency of Jersey Core (Client/Server).

## CDI Improvements

Jakub submitted some CDI improvements (available also in micro releases of Jersey 1.18):

  * Support for CDI when deploying an EAR that contains multiple JAX-RS based WARs in it
  * Custom <a href="https://jersey.github.io/apidocs/1.19/jersey/com/sun/jersey/core/util/Priority.html">Priority</a> annotation that allows to select a particular <a href="https://jersey.github.io/apidocs/1.19/jersey/com/sun/jersey/core/spi/component/ioc/IoCComponentProvider.html">IoCCompomentProvider</a> instance for each component
  * Better lookup for injecting CDI beans

## WADL JSON Schema Example

Adam added new <a href="https://github.com/jersey/jersey-1.x/tree/master/samples/wadl-json-schema-webapp">example</a> that demonstrates the <a href="https://github.com/jersey/jersey-1.x/tree/master/contribs/jersey-wadl-json-schema">wadl-json-schema</a> extension module. It generates JSON Schemas for produced/consumed types and exposes it under _application.wadl/[bean-name]_ path. It also changes the _<representation>_ element in WADL XML by adding references to the schemas.

## Bugs, Tasks and Improvements

  *  <a href="https://github.com/lcsanchez">Luis</a> updated the URL encoding algorithm so that it iterates over code points (ints) instead of code units (chars) and fixed <a href="https://java.net/jira/browse/JERSEY-2293">Unicode Astral characters included in Uri parts</a>. Thanks!
  * Also thanks to <a href="https://github.com/ingarabr">Ingar</a> who migrated an unfortunate mistake in Jersey Multipart which caused creation of temporary files (without deleting them) to check whether _mimepull_ library (used in multipart module) would work properly.
  * Also now it&#8217;s able to use Jersey Multipart module on Google AppEngine without encountering problems described <a href="https://java.net/jira/browse/JERSEY-2279">here</a>.
  * <a href="https://java.net/jira/browse/JERSEY-2028">GZIPContentEncodingFilter</a> leak has been fixed.

If you have any feedback send an email to <users@jersey.java.net> (archived [here][8]) or log bugs in our [JIRA][9].

 [1]: https://jersey.java.net/
 [2]: http://jcp.org/aboutJava/communityprocess/mrel/jsr311/index.html
 [3]: http://jsr311.java.net/nonav/releases/1.1/spec/spec.html
 [4]: https://jersey.java.net/documentation/1.19/user-guide.html#getting-started
 [5]: https://jersey.java.net/documentation/1.19/user-guide.html#chapter_deps
 [6]: /2015/01/16/jersey-2-15-is-released/ "Jersey 2.15 is released"
 [7]: /latest-jersey-version/ "Latest Jersey Versions"
 [8]: http://markmail.org/search/?q=list%3Anet.java.dev.jersey.users
 [9]: http://java.net/jira/browse/JERSEY/