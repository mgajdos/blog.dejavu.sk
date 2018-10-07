---
title: Jersey 1.17 is released
author: Michal Gajdoš
type: post
date: 2013-01-24T12:54:43+00:00
expirydate: 2018-09-01T00:00:00+00:00
url: /jersey-1-17-is-released/
aliases: /2013/01/24/jersey-1-17-is-released/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - release

---
We have released the 1.17 version of [Jersey][1], the open source, production quality, reference implementation of JAX-RS. The JAX-RS 1.1 [specification][2] is available at the JCP web site and also available in non-normative HTML [here][3].

For an overview of JAX-RS features read the Jersey [user guide][4]. To get started with Jersey read the [getting started section][5] of that guide. To understand more about what Jersey depends on read the [dependencies section][6] of that guide. See change log [here][7].

<!--more-->

Here is the list of notable changes (versions 1.13-1.17):

**1.17**

  * added support for notifying <a href="https://jersey.java.net/nonav/apidocs/1.17/jersey/com/sun/jersey/spi/monitoring/ResponseListener.html">ResponseListener</a> implementations about thrown unmapped application exceptions (see <a href="https://jersey.java.net/nonav/apidocs/1.17/jersey/com/sun/jersey/spi/monitoring/ResponseListener.html#onError(long,%20java.lang.Throwable)">ResponseListener#onError</a>) (<a href="http://java.net/jira/browse/JERSEY-1642">JERSEY-1642</a>)
  * fixed the hardcoded package in project created using the `jersey-quickstart-grizzly2` archetype (<a href="http://java.net/jira/browse/JERSEY-1618">JERSEY-1618</a>)
  * accepted contribution by Arul Dhesiaseelan &#8211; Simple Server version update (4.1.20 -> 5.0.4) (<a href="http://java.net/jira/browse/JERSEY-1641">JERSEY-1641</a>)

**1.16**
    
  * <a href="https://jersey.java.net/nonav/apidocs/1.17/jersey/com/sun/jersey/api/client/filter/HTTPBasicAuthFilter.html">HTTPBasicAuthFilter</a>/<a href="https://jersey.java.net/nonav/apidocs/1.17/jersey/com/sun/jersey/api/client/filter/HTTPDigestAuthFilter.html">HTTPDigestAuthFilter</a> constructors allowing you to avoid storing plain password value in a String variable (support for passwords represented as `byte[]`) (<a href="http://java.net/jira/browse/JERSEY-1601">JERSEY-1601</a>)
  * fix for OPTIONS call to a webservice method with a leading &#8220;/&#8221; in `@Path` annotation
  * accepted contribution by Gerard Davison &#8211; Automatic JSON-Schema Generation (for more information on this topic take a look at <a href="http://kingsfleet.blogspot.co.uk/2012/11/json-schema-generation-in-jersey.html">this</a> blog post)

**1.15**

  * enabled `InputStream` parameter to be set on a WADL generator (<a href="http://java.net/jira/browse/JERSEY-1409">JERSEY-1409</a>)
  * `Multipart.BodyPart` now allows setting headers other than those starting with `Content-*` (<a href="http://java.net/jira/browse/JERSEY-1448">JERSEY-1448</a>)
  * applied the patch from Charlie Groves to unwrap <a href="https://jersey.java.net/nonav/apidocs/1.17/jersey/javax/ws/rs/WebApplicationException.html">WebApplicationException</a> from the Guice `ProvisionException` (<a href="http://java.net/jira/browse/JERSEY-1386">JERSEY-1386</a>)

**1.14**

  * fix for getting file name from the content disposition header of multi-part messages sent by Internet Explorer (<a href="http://java.net/jira/browse/JERSEY-759">JERSEY-759</a>)
  * support for package scanning (resource & provider discovery in WAR archives) in OSGi environment (<a href="http://java.net/jira/browse/JERSEY-881">JERSEY-881</a>)
  * WADL: Matrix params generation moved to "resource" element to not violate WADL spec (<a href="http://java.net/jira/browse/JERSEY-1336">JERSEY-1336</a>)
  * added property to WadlGeneratorGrammarsSupport which controls whether Jersey generated grammars should be used when user explicitly sets its own grammar (<a href="http://java.net/jira/browse/JERSEY-1230">JERSEY-1230</a>)

**1.13**

  * support for retrieving services defined in `META-INF/services` for Jersey extensions (oauth, multipart, &#8230;) that has other version as used Jersey Core module. ([JERSEY-1124][8])
  * added Vary header in [GZIPContentEncodingFilter][9] ([JERSEY-1228][10])
  * added [ResourceContext][11] binding to Guice ([JERSEY-1219][12])
  * extracting Form parameters when consumed by filters enabled for POST, PUT and other methods as well ([JERSEY-1200][13])
  * added SubjectSecurityContext to enable subject-based security
  * JSON support refactored + support for MOXy as JAXB provider

For feedback send email to:

> <users@jersey.java.net> (archived [here][14])

or log bugs/features [here][15].

 [1]: https://jersey.java.net/
 [2]: http://jcp.org/aboutJava/communityprocess/mrel/jsr311/index.html
 [3]: http://jsr311.java.net/nonav/releases/1.1/spec/spec.html
 [4]: https://jersey.java.net/nonav/documentation/1.17/user-guide.html
 [5]: https://jersey.java.net/nonav/documentation/1.17/user-guide.html#getting-started
 [6]: https://jersey.java.net/nonav/documentation/1.17/user-guide.html#chapter_deps
 [7]: http://java.net/projects/jersey/sources/svn/content/tags/jersey-1.17/jersey/changes.txt
 [8]: http://java.net/jira/browse/JERSEY-1124
 [9]: https://jersey.java.net/nonav/apidocs/1.17/jersey/com/sun/jersey/api/client/filter/GZIPContentEncodingFilter.html
 [10]: http://java.net/jira/browse/JERSEY-1228
 [11]: https://jersey.java.net/nonav/apidocs/1.17/jersey/com/sun/jersey/api/core/ResourceContext.html
 [12]: http://java.net/jira/browse/JERSEY-1219
 [13]: http://java.net/jira/browse/JERSEY-1200
 [14]: http://markmail.org/search/?q=list%3Anet.java.dev.jersey.users
 [15]: http://java.net/jira/browse/JERSEY/