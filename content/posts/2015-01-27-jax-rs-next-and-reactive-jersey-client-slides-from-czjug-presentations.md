---
title: JAX-RS.next and Reactive Jersey Client – Slides from CZJUG presentations
author: Michal Gajdoš
type: post
date: 2015-01-27T12:41:41+00:00
url: /jax-rs-next-and-reactive-jersey-client-slides-from-czjug-presentations/
aliases: /2015/01/27/jax-rs-next-and-reactive-jersey-client-slides-from-czjug-presentations/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - client
  - mvc
  - reactive
  - security

---
Yesterday I was given the opportunity to speak at our local Java User Group meeting (CZJUG). I was covering two topics – _JAX-RS.next_ and _Reactive Jersey Client_. In this post I&#8217;d like to share the used slide-decks and some additional resources.

<!--more-->

## JAX-RS.next

> With the advent of HTML5 and thin-server architectures, JAX-RS is becoming an increasingly important component of the Java EE platform. This presentation covers a few proposed enhancements and extensions to JAX-RS as part of Java EE 8. These proposals include better support for JSON (JSON-B), improved injection, enhanced support for hypermedia as well as declarative security, server-sent events, and MVC.

(Download the presentation in PDF format – [JAX-RS.next – CZJUG 2015][1])

## Reactive Jersey Client

> Reactive Jersey Client API is quite a qeneric API allowing end users to utilize the popular reactive programming model when using JAX-RS (Jersey) Client. Together we&#8217;re going to take a look at an orchestration layer example where Jersey Client (sync, async, rx) would be used to obtain data from remote services and combine those data to construct a response for the calling party.

(Download the presentation in PDF format – [Reactive Jersey Client – CZJUG 2015][2])

### Examples

  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-webapp">Travel Agency (Orchestration Layer) using Reactive Jersey Client (Java 7)</a>
  * <a class="link" href="https://github.com/jersey/jersey/tree/master/examples/rx-client-java8-webapp">Travel Agency (Orchestration Layer) using Reactive Jersey Client (Java 8)</a>

### Other resources related to Reactive Jersey Client

Blog posts:

  * [Part 1 – Motivation][3]
  * [Part 2 – Usage and Supported Reactive Libraries][4]
  * [Part3 – Customization (SPI)][5]

Jersey User Guide:

  * <a href="https://jersey.github.io/documentation/latest/rx-client.html">Reactive Jersey Client API</a>

 [1]: /uploads/jaxrs.next_reduced.pdf
 [2]: /uploads/reactive_jersey_client_reduced.pdf
 [3]: /2015/01/07/reactive-jersey-client-part-1-motivation "Reactive Jersey Client, Part 1 – Motivation"
 [4]: /2015/01/07/reactive-jersey-client-part-2-usage-and-supported-reactive-libraries "Rx Client – Usage and Supported Reactive Libraries"
 [5]: /2015/01/07/reactive-jersey-client-part-3-customization "Rx Client – Customization (SPI)"