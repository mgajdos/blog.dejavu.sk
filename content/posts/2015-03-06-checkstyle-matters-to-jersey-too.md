---
title: Checkstyle matters to Jersey too …
author: Michal Gajdoš
type: post
date: 2015-03-06T12:09:38+00:00
url: /checkstyle-matters-to-jersey-too/
aliases: /2015/03/06/checkstyle-matters-to-jersey-too/
categories:
  - Jersey
tags:
  - jersey
  - checkstyle

---
Last week <a href="https://twitter.com/pavel_bucek">Pavel</a> wrote and published an interesting blog post about <a href="https://blogs.oracle.com/PavelBucek/entry/why_checkstyle_matters">Why Checkstyle matters &#8230;</a> I remember how he was describing his encounter with <a href="https://github.com/zeromq/jeromq">JeroMQ</a> to me. Especially the part where he wasn&#8217;t able to build the project for a few times because of the <a href="http://checkstyle.sourceforge.net">Checkstyle</a> issues that pop-up right after he executed the build. I found the whole idea behind running code style checks during build appealing as well and decided to do something similar for Jersey.

<!--more-->

My main motivation, besides better readability and not mixing various styles, was to make code reviews easier and a little bit more automated. Reviewer doesn&#8217;t need to focus and spend his energy on things that are still important but take time and can be done by machines.

In Jersey we have a document called _Coding Conventions_ that summarizes rules that we (and our contributors) ought to be following. For example,

  * The HARD LIMIT for the line length is set to **130 characters**. This hard limit is enforced by the checkstyle rules.
  * Never leave out braces in single-line compound statemets, e.g.: {{< highlight java >}}
// Should be:
    if (i > 0)    //     if (i > 0) {
      i--;        //       i--;
                  //     }
{{< / highlight >}}

  * Don&#8217;t use star (.*) imports.
  * Use spaces instead of tabs.

As mentioned in the first point these rules are checked by Checkstyle but they were never really enforced. And since they were never enforced some of them were not taken seriously. Even we weren&#8217;t following them all (e.g. the line length).

After discussion we have decided that a subset of the existing rules (<a href="https://github.com/jersey/jersey/blob/2.16/etc/config/checkstyle.xml">checkstyle.xml</a> from Jersey 2.16) should be really enforced during build. At first I ruled out all rules that check JavaDocs because they simply take too much time. Then I was looking for other common code style definitions which might be good for us too and I found <a href="http://checkstyle.sourceforge.net/google_style.html">Google&#8217;s style</a> page (see the <a href="https://github.com/checkstyle/checkstyle/blob/master/src/main/resources/google_checks.xml">google_checks.xml</a>). The whole effort resulted in our own <a href="https://github.com/jersey/jersey/blob/master/etc/config/checkstyle-verify.xml#L45">checkstyle-verify.xml</a> set of rules that are executed in (Maven) validation phase of Jersey build. The rest of the rules (based on Sun&#8217;s recommendations) are still present in the workspace (<a href="https://github.com/jersey/jersey/blob/master/etc/config/checkstyle.xml#L69">checkstyle.xml</a>), they are run but not enforced.

So I had my set of rules (which was the easy part). All I had to do next was to apply them to the code. Sounds easy, it was not. As Pavel said, _&#8220;It was pain&#8221;_. No real automated tool to fix those issues. Perfect! So I updated formatter in IntelliJ, created some regular expression to fix the problems and applied them. Job for a monkey (or an useful idiot like me). However, I was able to do it – you can see it in this <a href="https://github.com/jersey/jersey/commit/4c99cdefe97766c29efd77afdafa2271d1871613">commit</a> – 751 changed files (ouch!).

I am not saying that all the rules will be there in a year. Some of them are little controversial and we may end up throwing them away or update their scope. However, in the end I am glad that we took the opportunity to do this and (hopefully) improved our code.

## Contributions

I am aware that the above might sound scary. You may think you&#8217;re going to spend too much time fixing checkstyle issues when developing/prototyping. Don&#8217;t worry, you don&#8217;t have to comply with check style rules during the process of developing new features (or fixing a bug). You can turn the check off by either using `checkstyleSkip` maven profile or by setting maven `checkstyle.skip `property to `true`.

{{< highlight java >}}
mvn clean install -PcheckstyleSkip
{{< / highlight >}}

However, before creating a pull request on GitHub you should run the build with checkstyle checks otherwise the verification will fail.

### IntelliJ IDEA

I&#8217;ve prepared a code style XML file for Intellij IDEA (see [Jersey.xml][1], _<codeStyleSettings language=&#8221;JAVA&#8221;>_ part). You can take it and copy it to `~/Library/Preferences/IntelliJIdea14/codestyles` (or similar directory on your OS). After the file is in place you can review the styles in _Preferences -> Editor -> Code Style -> Java _(Scheme: Jersey). Changes I&#8217;ve made are entirely on _Wrapping and Braces_ tab. Two notes:

  * Make sure to check _**Keep when reformatting** -> Line Breaks_ on the tab – otherwise even manually aligned code gets formatted (which may be unwanted)
  * Consider turning off JavaDoc formatting as it can also produce unwanted changes

 [1]: /uploads/Jersey.xml