---
title: Jersey Monthly Newsletter – February 2015
author: Michal Gajdoš
type: post
date: 2015-02-27T09:33:17+00:00
expirydate: 2018-09-01T00:00:00+00:00
url: /jersey-monthly-newsletter-february-2015/
aliases: /2015/02/27/jersey-monthly-newsletter-february-2015/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - newsletter

---
Jersey team has become more active in writing blog posts so I&#8217;ve decided to give it a try and publish a short summary of these posts and maybe something more at the end of each month. This first post focuses heavily on Jersey but I&#8217;d like to include interesting articles from more sources about the world of JAX-RS and REST services in the next months.

<!--more-->

At the beginning I&#8217;d like to mention a page where you can always find information about the latest released Jersey 1.x and 2.x. This includes links to the release notes, API docs, user guide and a blog post related to that release. It&#8217;s called <a href="/latest-jersey-version/">Latest Jersey Versions</a> and it is available on this very site.

But lets get into more interesting stuff. One of the goals we had in our Jersey sprints in February was _**performance**_. Articles <a href="https://blogs.oracle.com/japod/entry/jersey_2_performance">Jersey 2 Performance</a> and <a href="/2015/02/12/performance-improvements-of-sub-resource-locators-in-jersey/">Performance Improvements of Sub-Resource Locators in Jersey</a> dive into the details what we did to improve performance in version 2.16 and provide comparison with older versions as well. We also prepared a set of tools to run benchmarks with your own JAX-RS (server/client) applications – they are described in <a href="/2015/02/19/micro-benchmarking-jax-rs-applications/">Micro-Benchmarking JAX-RS Applications</a>. Nobody&#8217;s perfect and we (our injection framework) had some memory leaks when an application was deployed on a servlet container like Tomcat/Jetty. Adam has been investigating this and he wrote a short article about finding memory leaks in Jersey applications on Tomcat. You can find it here – <a href="http://hnusfialovej.cz/2015/02/19/1-funny-thing-not-to-forget-profiling-on-tomcat/">One Funny Thing Not to Forget While Profiling on Tomcat</a>.

When it comes to improving **_features_** there are few things to mention. If you use, or thinking of using, CDI in your application then make sure to take a look at <a href="https://blogs.oracle.com/japod/entry/container_agnostic_cdi_support_in">Container Agnostic CDI Support In Jersey</a> where you learn how to integrate Jersey with CDI outside of Java EE. We got even further and enabled injection of JAX-RS components into pure CDI beans. Take a look at <a href="http://hnusfialovej.cz/2015/02/25/jersey-further-improves-cdi-integration/">Jersey further improves CDI integration</a> for more details. Speaking of CDI, I found a nice article about simplifying JAX-RS caching using CDI by Abhishek Gupta, <a href="https://abhirockzz.wordpress.com/2015/02/20/simplifying-jax-rs-caching-with-cdi">Simplifying JAX-RS caching with CDI</a>.

Our Entity Filtering feature has been improved as well and now it supports JSON media type handled via <a href="https://github.com/FasterXML/jackson">Jackson</a> library. Overview of how you can use it and what to expect from this support is covered in <a href="/2015/02/04/jerseys-entity-filtering-meets-jackson/">Jersey’s Entity Filtering meets Jackson</a>.

In the last month we released two Jersey versions – JAX-RS 2.0 reference implementation <a href="https://github.com/jersey/jersey/releases/tag/2.16">Jersey 2.16</a> and also <a href="https://github.com/jersey/jersey-1.x/releases/tag/1.19">Jersey 1.19</a> which is reference implementation of JAX-RS 1.1. The overview of notable changes, merged pull requests and important bug fixes are in the following posts – <a href="http://yatel.kramolis.cz/2015/02/jersey-216-has-been-released.html">Jersey 2.16 has been released</a> and <a href="/2015/02/13/jersey-1-19-is-released/">Jersey 1.19 is released</a> respectively.

Oh, I almost forgot about two other articles. Want to use MongoDB with Jersey? <a style="line-height: 1.6471;" href="http://hnusfialovej.cz/2015/01/30/upload-file-to-mongodb-using-jersey-2/">Upload file to MongoDB using Jersey 2</a> is there for you. And what about Docker and Jersey? Yes, [dockerize your app][1]!

If you feel like some other blog post or article mentioning Jersey or JAX-RS should be here please let me know.

 [1]: http://okosatka.github.io/docker/2015/02/12/dockerize-your-app/