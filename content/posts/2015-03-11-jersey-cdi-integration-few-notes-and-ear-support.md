---
title: Jersey CDI Integration – Few Notes and EAR Support
author: Michal Gajdoš
type: post
date: 2015-03-11T13:35:43+00:00
url: /jersey-cdi-integration-few-notes-and-ear-support/
aliases: /2015/03/11/jersey-cdi-integration-few-notes-and-ear-support/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey

---
During our last sprint I was porting support for CDI injections for EARs that contain multiple JAX-RS web applications (WARs). This worked fine in Jersey 1 and it works on WebLogic but we didn&#8217;t have support for these kind of deployments directly in Jersey (which means that Glassfish was affected as well) until this very moment. I&#8217;d like to share some findings I&#8217;ve made while working on this task.

<!--more-->

> At the beginning, just let me mention that since Jersey 2.17 we do have a working CDI support for multiple JAX-RS webapps deployed inside an EAR. You can test it on the latest nightly <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.1/nightly/">Glassfish</a>.

Jersey internally uses <a href="https://hk2.java.net">HK2</a> injection framework to support injections (JAX-RS and custom) when running standalone. This is nice but makes injections between HK2 and CDI, JAX-RS components into CDI (HK2 -> CDI) and CDI beans into pure JAX-RS components (CDI -> HK2), slightly harder to implement.

CDI support in Jersey is implemented using two entry points:

  * CDI <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/Extension.html">Extension</a> that allows us to discover and process CDI beans and injection targets, and
  * Jersey <a href="https://jersey.github.io/apidocs/latest/jersey/org/glassfish/jersey/server/spi/ComponentProvider.html">ComponentProvider</a> which makes possible for 3rd party dependency injection frameworks to hook into Jersey and inject custom managed objects into classes used by Jersey application

Both of mentioned entry points are implemented in a single class called <a href="https://github.com/jersey/jersey/blob/master/ext/cdi/jersey-cdi1x/src/main/java/org/glassfish/jersey/ext/cdi1x/internal/CdiComponentProvider.java#L126">CdiCompomentProvider</a>.

As the life-cycle of CDI and JAX-RS apps are different we need to handle and keep in sync always at least two instances of CdiComponentProvider. CDI creates one instance of this class, does its thing and then, after a while, Jersey gets its turn. The instance created by Jersey receives an instance of <a href="https://hk2.java.net/hk2-api/apidocs/org/glassfish/hk2/api/ServiceLocator.html">ServiceLocator</a>, finds CdiComponetProvider created by CDI (via <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/BeanManager.html">BeanManager</a>) and provides the service locator to this instance to make sure HK2 is in play when CDI creates and injects beans used in the application. This is, very quickly, how the bi-directional injection works at the moment.

However, there are few interesting things, worth mentioning, that pops up with this approach.

## #1: One CDI extension per deployment

As I mentioned we always have to deal with at least two instances of our component provider. This is the case for standalone Jersey/CDI deployments as well as for single WARs deployed on a Servlet container.

The EAR deployments are different. If EAR contains multiple Jersey-based WARs you&#8217;d expect to still have two instances of CdiComponentProvider per WAR (one from CDI, one from Jersey).

Wrong!

CDI only creates one instance of the extension per EAR. Underneath CDI provides correct injections into its beans which is great. But this approach is not intuitive for someone that needs to do some advanced processing in the extension at the first sight.

The consequences for Jersey are pretty straight-forward: Since we are now dealing with multiple service locators (multiple WARs) we need to provide the correct locator for each <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/InjectionTarget.html">InjectionTarget</a> based on some context. We decided the context in this case would be the context-path of WAR. Problem solved.

**Conclusion**: Only one instance of CDI <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/Extension.html">Extension</a> is created for a deployment. Does&#8217;t matter whether your deployment is EAR or WAR.

## #2: CDI extensions are proxies

In the light of previous findings this statement is not surprising for CDI beans. Even not surprising for <a href="http://docs.oracle.com/javaee/7/api/javax/servlet/ServletContext.html">ServletContext</a> and <a href="http://docs.oracle.com/javaee/7/api/javax/servlet/ServletConfig.html">ServletConfig</a> obtained from a <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/BeanManager.html">BeanManager</a>. But it surprised me when I found that CDI <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/Extension.html">Extension</a>s are proxies too (remember there is only one instance of particular extension per deployment).

With this in mind you cannot do something like this:

{{< highlight java "hl_lines=11" >}}
@Override
public void initialize(final ServiceLocator locator) {
    this.locator = locator;
    this.beanManager = CdiUtil.getBeanManager();

    if (beanManager != null) {
        // Try to get CdiComponentProvider created by CDI.
        final CdiComponentProvider extension = beanManager.getExtension(CdiComponentProvider.class);

        if (extension != null) {
            extension.locator = locator;

            // ...
        }
    }
}
{{< / highlight >}}

**Conclusion**: Never access fields of a (CDI) proxy from another instance of that class directly.

This makes sense I guess. I think I would have provide some kind of setter (accessor) in this case as well. But what I was not expecting was &#8230;

## #3: Private setters/accessors on (CDI) proxies are never called and you get no error

When you create setter for the case describe above, for example the _setLocator_ method, and you make it private CDI wouldn&#8217;t invoke it.

When the method is private your application would deploy without any problems but you get errors in runtime when injection fails. And it really isn&#8217;t pleasant to fix these kind of problems. I would expect some kind of error during deployment time. But no.

**Conclusion:** (CDI) Proxies don&#8217;t see your private setters/getters/accessors. Make the method at least package private.

## #4: Different Application Servers behave differently

Not that surprising but it&#8217;s always good to remember that when you&#8217;re trying to do something that is expected to work on multiple application servers (e.g. Glassfish or WebLogic) or in multiple environments you need to find some middle ground where your code just works for everybody.

For example, <a href="http://docs.oracle.com/javaee/7/api/javax/enterprise/inject/spi/BeanManager.html">BeanManager</a> lookup. On application servers you can obtain an instance of bean manager via JNDI lookup:

{{< highlight java "hl_lines=3-4" >}}
InitialContext initialContext = null;
try {
    initialContext = new InitialContext();
    return (BeanManager) initialContext.lookup("java:comp/BeanManager");
} catch (final Exception ex) {
    // Log and return null.
    return null;
} finally {
    if (initialContext != null) {
        try {
            initialContext.close();
        } catch (final NamingException ignored) {
            // no-op
        }
    }
}
{{< / highlight >}}

But this doesn&#8217;t work for standalone deployments, for them you have to use:

{{< highlight java "hl_lines=2" >}}
try {
    return CDI.current().getBeanManager();
} catch (final Exception e) {
    // Log and return null.
    return null;
}
{{< / highlight >}}

OK, so you think your universal code would at first look for a BeanManager in JNDI and as a fallback it&#8217;d use _CDI.current()._

{{< highlight java "hl_lines=3-4 7" >}}
InitialContext initialContext = null;
try {
    initialContext = new InitialContext();
    return (BeanManager) initialContext.lookup("java:comp/BeanManager");
} catch (final Exception ex) {
    try {
        return CDI.current().getBeanManager();
    } catch (final Exception e) {
        // Log and return null.
        return null;
    }
} finally {
    if (initialContext != null) {
        try {
            initialContext.close();
        } catch (final NamingException ignored) {
            // no-op
        }
    }
}
{{< / highlight >}}

Good approach, you think. Unfortunately this doesn&#8217;t solve all the issues. On WebLogic, for example, when you try to run this code for a non-CDI application and then you try to deploy a CDI application the second deployment would fail. The first call of _CDI.current().getBeanManager()_ gets the server in some inconsistent state which is the cause of the second deployment failure.

**Conclusion:** Although this last statement is not specific to CDI, and it can be applied to any domain, it&#8217;s good to always have it in mind.