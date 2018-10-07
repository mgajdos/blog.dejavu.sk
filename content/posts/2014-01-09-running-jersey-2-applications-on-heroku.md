---
title: Running Jersey 2 Applications on Heroku
author: Michal Gajdoš
type: post
date: 2014-01-09T13:50:32+00:00
url: /running-jersey-2-applications-on-heroku/
aliases: /2014/01/09/running-jersey-2-applications-on-heroku/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey
  - heroku

---
In this article we&#8217;ll create a simple [Jersey][1] application deploy it in [Jetty][2] servlet container and everything will be hosted on [Heroku][3].

## Create an Application

To create a skeleton of Jersey 2 web-app that can be run on Heroku from the [jersey-heroku-webapp][4] archetype (available since Jersey 2.5), execute the following Maven command in the directory where you want the new project should be located:

```bash
mvn archetype:generate -DarchetypeArtifactId=jersey-heroku-webapp \
    -DarchetypeGroupId=org.glassfish.jersey.archetypes -DinteractiveMode=false \
    -DgroupId=com.example -DartifactId=simple-heroku-webapp -Dpackage=com.example \
    -DarchetypeVersion=2.5.1
```

Adjust the `groupId`, `package` and/or `artifactId` of your new web application project to your needs or, alternatively, you can change it by updating the new project `pom.xml` once it gets generated.

<!--more-->

Once the project generation from a Jersey maven archetype is successfully finished, you should see the new `simple-heroku-webapp` project directory created in your current location. The directory contains a standard Maven project structure:

  * `src/main/java` &#8211; project sources
  * `src/main/resources` &#8211; project resources
  * `src/main/webapp` &#8211; project web application files
  * `src/test/java` &#8211; project test-sources (based on <a href="https://jersey.github.io/apidocs/2.5.1/jersey/org/glassfish/jersey/test/JerseyTest.html">JerseyTest</a>)
  * `pom.xml` &#8211; project build and management configuration
  * `system.properties` &#8211; Heroku system properties (OpenJDK version)
  * `Procfile` &#8211; lists of the process types in an application for Heroku

The project ([simple-heroku-webapp][5]) contains one JAX-RS resource class, [MyResouce][6], and one resource method which returns simple text message. To make sure the resource is properly tested there is also an end-to-end test-case in [MyResourceTest][7] (the test is based on <a href="https://jersey.github.io/apidocs/2.5.1/jersey/org/glassfish/jersey/test/JerseyTest.html">JerseyTest</a> from our [Jersey Test Framework][8]). The project contains the standard Java EE web application `web.xml` deployment descriptor under `src/main/webapp/WEB-INF` since the goal is to deploy the application in a Servlet container (in our case the application will run in [Jetty][2] on Heroku).

To compile and package the application into a WAR, invoke the following maven command in your console:

{{< highlight bash >}}
mvn clean package
{{< / highlight >}}

A successful build output will produce an output similar to the one below:

{{< highlight bash >}}
Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO]
[INFO] --- maven-war-plugin:2.2:war (default-war) @ simple-heroku-webapp ---
[INFO] Packaging webapp
[INFO] Assembling webapp [simple-heroku-webapp] in [.../simple-heroku-webapp/target/simple-heroku-webapp]
[INFO] Processing war project
[INFO] Copying webapp resources [.../simple-heroku-webapp/src/main/webapp]
[INFO] Webapp assembled in [85 msecs]
[INFO] Building war: .../simple-heroku-webapp/target/simple-heroku-webapp.war
[INFO] WEB-INF/web.xml already added, skipping
[INFO]
[INFO] --- maven-dependency-plugin:2.8:copy-dependencies (copy-dependencies) @ simple-heroku-webapp ---
[INFO] Copying hk2-locator-2.2.0-b21.jar to .../simple-heroku-webapp/target/dependency/hk2-locator-2.2.0-b21.jar
[INFO] Copying jersey-container-servlet-core-2.5.1.jar to .../simple-heroku-webapp/target/dependency/jersey-container-servlet-core-2.5.1.jar
[INFO] Copying jetty-security-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-security-9.0.6.v20130930.jar
[INFO] Copying asm-all-repackaged-2.2.0-b21.jar to .../simple-heroku-webapp/target/dependency/asm-all-repackaged-2.2.0-b21.jar
[INFO] Copying jersey-client-2.5.1.jar to .../simple-heroku-webapp/target/dependency/jersey-client-2.5.1.jar
[INFO] Copying validation-api-1.1.0.Final.jar to .../simple-heroku-webapp/target/dependency/validation-api-1.1.0.Final.jar
[INFO] Copying jetty-webapp-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-webapp-9.0.6.v20130930.jar
[INFO] Copying cglib-2.2.0-b21.jar to .../simple-heroku-webapp/target/dependency/cglib-2.2.0-b21.jar
[INFO] Copying osgi-resource-locator-1.0.1.jar to .../simple-heroku-webapp/target/dependency/osgi-resource-locator-1.0.1.jar
[INFO] Copying hk2-utils-2.2.0-b21.jar to .../simple-heroku-webapp/target/dependency/hk2-utils-2.2.0-b21.jar
[INFO] Copying hk2-api-2.2.0-b21.jar to .../simple-heroku-webapp/target/dependency/hk2-api-2.2.0-b21.jar
[INFO] Copying jetty-io-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-io-9.0.6.v20130930.jar
[INFO] Copying jetty-server-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-server-9.0.6.v20130930.jar
[INFO] Copying jetty-util-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-util-9.0.6.v20130930.jar
[INFO] Copying jersey-server-2.5.1.jar to .../simple-heroku-webapp/target/dependency/jersey-server-2.5.1.jar
[INFO] Copying jetty-http-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-http-9.0.6.v20130930.jar
[INFO] Copying guava-14.0.1.jar to .../simple-heroku-webapp/target/dependency/guava-14.0.1.jar
[INFO] Copying jetty-xml-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-xml-9.0.6.v20130930.jar
[INFO] Copying jersey-common-2.5.1.jar to .../simple-heroku-webapp/target/dependency/jersey-common-2.5.1.jar
[INFO] Copying javax.ws.rs-api-2.0.jar to .../simple-heroku-webapp/target/dependency/javax.ws.rs-api-2.0.jar
[INFO] Copying jetty-servlet-9.0.6.v20130930.jar to .../simple-heroku-webapp/target/dependency/jetty-servlet-9.0.6.v20130930.jar
[INFO] Copying javax.inject-2.2.0-b21.jar to .../simple-heroku-webapp/target/dependency/javax.inject-2.2.0-b21.jar
[INFO] Copying javax.servlet-3.0.0.v201112011016.jar to .../simple-heroku-webapp/target/dependency/javax.servlet-3.0.0.v201112011016.jar
[INFO] Copying javax.annotation-api-1.2.jar to .../simple-heroku-webapp/target/dependency/javax.annotation-api-1.2.jar
[INFO] Copying jersey-container-servlet-2.5.1.jar to .../simple-heroku-webapp/target/dependency/jersey-container-servlet-2.5.1.jar
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 6.637s
[INFO] Finished at: Thu Jan 09 11:24:54 CET 2014
[INFO] Final Memory: 19M/253M
[INFO] ------------------------------------------------------------------------
{{< / highlight >}}

Now when you know that everything went as expected you are ready to:

  * make some changes in your project,
  * take the packaged WAR (located under `./target/simple-service-webapp.war`) and deploy it to a Servlet container of your choice, or
  * deploy it on Heroku

> If you want to make some changes to your application you can run the application locally by simply running `mvn clean package jetty:run` (which starts the embedded Jetty server) or by `java -cp target/classes:target/dependency/* com.example.heroku.Main` (this is how Jetty is started on Heroku).

## Deploy it on Heroku

I don&#8217;t want to go into details how to create an account on <a href="https://www.heroku.com/">Heroku</a> and setup the tools on your machine. You can find a lot of useful information in this article: <a href="https://devcenter.heroku.com/articles/getting-started-with-java">Getting Started with Java on Heroku</a>. Instead, let&#8217;s take a look at the steps needed after your environment is ready.

The first step is to create a Git repository from your project:

{{< highlight bash "hl_lines=1" >}}
$ git init
Initialized empty Git repository in /.../simple-heroku-webapp/.git/
{{< / highlight >}}

Then, create Heroku application from the project and add a remote reference to your Git repository:

{{< highlight bash "hl_lines=1" >}}
$ heroku create
Creating simple-heroku-webapp... done, stack is cedar
http://simple-heroku-webapp.herokuapp.com/ | git@heroku.com:simple-heroku-webapp.git
Git remote heroku added
{{< / highlight >}}

> I&#8217;ve changed the name of the instance in the output to simple-heroku-webapp. Yours will be named more like tranquil-basin-4744.

Add and commit files to your Git repository:

{{< highlight bash "hl_lines=1 3" >}}
$ git add src/ pom.xml Procfile system.properties

$ git commit -a -m "initial commit"
[master (root-commit) e2b58e3] initial commit
 7 files changed, 221 insertions(+)
 create mode 100644 Procfile
 create mode 100644 pom.xml
 create mode 100644 src/main/java/com/example/MyResource.java
 create mode 100644 src/main/java/com/example/heroku/Main.java
 create mode 100644 src/main/webapp/WEB-INF/web.xml
 create mode 100644 src/test/java/com/example/MyResourceTest.java
 create mode 100644 system.properties
{{< / highlight >}}

Push changes to Heroku:

{{< highlight bash "hl_lines=1" >}}
$ git push heroku master
Fetching repository, done.
Counting objects: 19, done.
Delta compression using up to 8 threads.
Compressing objects: 100% (7/7), done.
Writing objects: 100% (11/11), 981 bytes | 0 bytes/s, done.
Total 11 (delta 3), reused 0 (delta 0)

-----> Java app detected
-----> Installing OpenJDK 1.7... done
-----> Installing settings.xml... done
-----> executing /app/tmp/cache/.maven/bin/mvn -B -Duser.home=/tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62 -Dmaven.repo.local=/app/tmp/cache/.m2/repository -s /app/tmp/cache/.m2/settings.xml -DskipTests=true clean install
       [INFO] Scanning for projects...
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/jersey-bom/2.5.1/jersey-bom-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/jersey-bom/2.5.1/jersey-bom-2.5.1.pom (15 KB at 97.0 KB/sec)
       [INFO]
       [INFO] ------------------------------------------------------------------------
       [INFO] Building simple-heroku-webapp 1.0-SNAPSHOT
       [INFO] ------------------------------------------------------------------------
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet/2.5.1/jersey-container-servlet-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet/2.5.1/jersey-container-servlet-2.5.1.pom (5 KB at 24.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/project/2.5.1/project-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/project/2.5.1/project-2.5.1.pom (4 KB at 11.0 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/project/2.5.1/project-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/project/2.5.1/project-2.5.1.pom (57 KB at 130.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet-core/2.5.1/jersey-container-servlet-core-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet-core/2.5.1/jersey-container-servlet-core-2.5.1.pom (5 KB at 18.6 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-server/2.5.1/jersey-server-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-server/2.5.1/jersey-server-2.5.1.pom (7 KB at 48.2 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-common/2.5.1/jersey-common-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-common/2.5.1/jersey-common-2.5.1.pom (8 KB at 73.4 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-client/2.5.1/jersey-client-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-client/2.5.1/jersey-client-2.5.1.pom (5 KB at 37.0 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-bundle/2.5.1/jersey-test-framework-provider-bundle-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-bundle/2.5.1/jersey-test-framework-provider-bundle-2.5.1.pom (5 KB at 22.0 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/project/2.5.1/project-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/project/2.5.1/project-2.5.1.pom (3 KB at 30.2 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/project/2.5.1/project-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/project/2.5.1/project-2.5.1.pom (3 KB at 3.3 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-grizzly2/2.5.1/jersey-test-framework-provider-grizzly2-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-grizzly2/2.5.1/jersey-test-framework-provider-grizzly2-2.5.1.pom (4 KB at 24.1 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/jersey-test-framework-core/2.5.1/jersey-test-framework-core-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/jersey-test-framework-core/2.5.1/jersey-test-framework-core-2.5.1.pom (4 KB at 13.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-grizzly2-http/2.5.1/jersey-container-grizzly2-http-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-grizzly2-http/2.5.1/jersey-container-grizzly2-http-2.5.1.pom (4 KB at 5.7 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http-server/2.3.8/grizzly-http-server-2.3.8.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http-server/2.3.8/grizzly-http-server-2.3.8.pom (5 KB at 43.3 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-project/2.3.8/grizzly-project-2.3.8.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-project/2.3.8/grizzly-project-2.3.8.pom (20 KB at 173.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-bom/2.3.8/grizzly-bom-2.3.8.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-bom/2.3.8/grizzly-bom-2.3.8.pom (11 KB at 51.3 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http/2.3.8/grizzly-http-2.3.8.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http/2.3.8/grizzly-http-2.3.8.pom (5 KB at 50.7 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-framework/2.3.8/grizzly-framework-2.3.8.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-framework/2.3.8/grizzly-framework-2.3.8.pom (7 KB at 27.7 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-inmemory/2.5.1/jersey-test-framework-provider-inmemory-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-inmemory/2.5.1/jersey-test-framework-provider-inmemory-2.5.1.pom (4 KB at 32.6 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-external/2.5.1/jersey-test-framework-provider-external-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-external/2.5.1/jersey-test-framework-provider-external-2.5.1.pom (4 KB at 29.9 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-default-client/2.5.1/jersey-test-framework-provider-default-client-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-default-client/2.5.1/jersey-test-framework-provider-default-client-2.5.1.pom (3 KB at 20.8 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jdk-http/2.5.1/jersey-test-framework-provider-jdk-http-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jdk-http/2.5.1/jersey-test-framework-provider-jdk-http-2.5.1.pom (4 KB at 4.1 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jdk-http/2.5.1/jersey-container-jdk-http-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jdk-http/2.5.1/jersey-container-jdk-http-2.5.1.pom (4 KB at 38.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-simple/2.5.1/jersey-test-framework-provider-simple-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-simple/2.5.1/jersey-test-framework-provider-simple-2.5.1.pom (4 KB at 33.4 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-simple-http/2.5.1/jersey-container-simple-http-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-simple-http/2.5.1/jersey-container-simple-http-2.5.1.pom (4 KB at 21.4 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jetty/2.5.1/jersey-test-framework-provider-jetty-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jetty/2.5.1/jersey-test-framework-provider-jetty-2.5.1.pom (4 KB at 39.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jetty-http/2.5.1/jersey-container-jetty-http-2.5.1.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jetty-http/2.5.1/jersey-container-jetty-http-2.5.1.pom (5 KB at 17.1 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/eclipse/jetty/jetty-continuation/9.0.6.v20130930/jetty-continuation-9.0.6.v20130930.pom
       Downloaded: http://s3pository.heroku.com/jvm/org/eclipse/jetty/jetty-continuation/9.0.6.v20130930/jetty-continuation-9.0.6.v20130930.pom (2 KB at 18.6 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet/2.5.1/jersey-container-servlet-2.5.1.jar
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-common/2.5.1/jersey-common-2.5.1.jar
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-server/2.5.1/jersey-server-2.5.1.jar
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-client/2.5.1/jersey-client-2.5.1.jar
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet-core/2.5.1/jersey-container-servlet-core-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet/2.5.1/jersey-container-servlet-2.5.1.jar (16 KB at 111.8 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-grizzly2/2.5.1/jersey-test-framework-provider-grizzly2-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-client/2.5.1/jersey-client-2.5.1.jar (156 KB at 812.5 KB/sec)
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-servlet-core/2.5.1/jersey-container-servlet-core-2.5.1.jar (53 KB at 276.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-grizzly2-http/2.5.1/jersey-container-grizzly2-http-2.5.1.jar
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/jersey-test-framework-core/2.5.1/jersey-test-framework-core-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-common/2.5.1/jersey-common-2.5.1.jar (684 KB at 2442.8 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http-server/2.3.8/grizzly-http-server-2.3.8.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/core/jersey-server/2.5.1/jersey-server-2.5.1.jar (809 KB at 2472.8 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http/2.3.8/grizzly-http-2.3.8.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/jersey-test-framework-core/2.5.1/jersey-test-framework-core-2.5.1.jar (15 KB at 106.3 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-framework/2.3.8/grizzly-framework-2.3.8.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-grizzly2-http/2.5.1/jersey-container-grizzly2-http-2.5.1.jar (28 KB at 158.8 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-inmemory/2.5.1/jersey-test-framework-provider-inmemory-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-grizzly2/2.5.1/jersey-test-framework-provider-grizzly2-2.5.1.jar (7 KB at 25.4 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-external/2.5.1/jersey-test-framework-provider-external-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http-server/2.3.8/grizzly-http-server-2.3.8.jar (220 KB at 1211.1 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-default-client/2.5.1/jersey-test-framework-provider-default-client-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-inmemory/2.5.1/jersey-test-framework-provider-inmemory-2.5.1.jar (16 KB at 125.3 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jdk-http/2.5.1/jersey-test-framework-provider-jdk-http-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-framework/2.3.8/grizzly-framework-2.3.8.jar (816 KB at 4689.4 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jdk-http/2.5.1/jersey-container-jdk-http-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-external/2.5.1/jersey-test-framework-provider-external-2.5.1.jar (6 KB at 47.2 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-simple/2.5.1/jersey-test-framework-provider-simple-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/grizzly/grizzly-http/2.3.8/grizzly-http-2.3.8.jar (326 KB at 1439.4 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-simple-http/2.5.1/jersey-container-simple-http-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-simple/2.5.1/jersey-test-framework-provider-simple-2.5.1.jar (7 KB at 48.6 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jetty/2.5.1/jersey-test-framework-provider-jetty-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-simple-http/2.5.1/jersey-container-simple-http-2.5.1.jar (23 KB at 212.8 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jetty-http/2.5.1/jersey-container-jetty-http-2.5.1.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-default-client/2.5.1/jersey-test-framework-provider-default-client-2.5.1.jar (4 KB at 16.5 KB/sec)
       Downloading: http://s3pository.heroku.com/jvm/org/eclipse/jetty/jetty-continuation/9.0.6.v20130930/jetty-continuation-9.0.6.v20130930.jar
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jdk-http/2.5.1/jersey-test-framework-provider-jdk-http-2.5.1.jar (7 KB at 32.7 KB/sec)
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jdk-http/2.5.1/jersey-container-jdk-http-2.5.1.jar (18 KB at 90.5 KB/sec)
       Downloaded: http://s3pository.heroku.com/jvm/org/eclipse/jetty/jetty-continuation/9.0.6.v20130930/jetty-continuation-9.0.6.v20130930.jar (16 KB at 213.9 KB/sec)
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/test-framework/providers/jersey-test-framework-provider-jetty/2.5.1/jersey-test-framework-provider-jetty-2.5.1.jar (7 KB at 48.8 KB/sec)
       Downloaded: http://s3pository.heroku.com/jvm/org/glassfish/jersey/containers/jersey-container-jetty-http/2.5.1/jersey-container-jetty-http-2.5.1.jar (25 KB at 172.7 KB/sec)
       [INFO]
       [INFO] --- maven-clean-plugin:2.4.1:clean (default-clean) @ simple-heroku-webapp ---
       [INFO]
       [INFO] --- maven-resources-plugin:2.4.3:resources (default-resources) @ simple-heroku-webapp ---
       [INFO] Using 'UTF-8' encoding to copy filtered resources.
       [INFO] skip non existing resourceDirectory /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/src/main/resources
       [INFO]
       [INFO] --- maven-compiler-plugin:2.5.1:compile (default-compile) @ simple-heroku-webapp ---
       [INFO] Compiling 2 source files to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/classes
       [INFO]
       [INFO] --- maven-resources-plugin:2.4.3:testResources (default-testResources) @ simple-heroku-webapp ---
       [INFO] Using 'UTF-8' encoding to copy filtered resources.
       [INFO] skip non existing resourceDirectory /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/src/test/resources
       [INFO]
       [INFO] --- maven-compiler-plugin:2.5.1:testCompile (default-testCompile) @ simple-heroku-webapp ---
       [INFO] Compiling 1 source file to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/test-classes
       [INFO]
       [INFO] --- maven-surefire-plugin:2.7.2:test (default-test) @ simple-heroku-webapp ---
       [INFO] Tests are skipped.
       [INFO]
       [INFO] --- maven-war-plugin:2.1.1:war (default-war) @ simple-heroku-webapp ---
       [INFO] Packaging webapp
       [INFO] Assembling webapp [simple-heroku-webapp] in [/tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/simple-heroku-webapp]
       [INFO] Processing war project
       [INFO] Copying webapp resources [/tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/src/main/webapp]
       [INFO] Webapp assembled in [72 msecs]
       [INFO] Building war: /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/simple-heroku-webapp.war
       [INFO] WEB-INF/web.xml already added, skipping
       [INFO]
       [INFO] --- maven-dependency-plugin:2.1:copy-dependencies (copy-dependencies) @ simple-heroku-webapp ---
       [INFO] Copying guava-14.0.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/guava-14.0.1.jar
       [INFO] Copying javax.annotation-api-1.2.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/javax.annotation-api-1.2.jar
       [INFO] Copying validation-api-1.1.0.Final.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/validation-api-1.1.0.Final.jar
       [INFO] Copying javax.ws.rs-api-2.0.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/javax.ws.rs-api-2.0.jar
       [INFO] Copying jetty-http-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-http-9.0.6.v20130930.jar
       [INFO] Copying jetty-io-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-io-9.0.6.v20130930.jar
       [INFO] Copying jetty-security-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-security-9.0.6.v20130930.jar
       [INFO] Copying jetty-server-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-server-9.0.6.v20130930.jar
       [INFO] Copying jetty-servlet-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-servlet-9.0.6.v20130930.jar
       [INFO] Copying jetty-util-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-util-9.0.6.v20130930.jar
       [INFO] Copying jetty-webapp-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-webapp-9.0.6.v20130930.jar
       [INFO] Copying jetty-xml-9.0.6.v20130930.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jetty-xml-9.0.6.v20130930.jar
       [INFO] Copying javax.servlet-3.0.0.v201112011016.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/javax.servlet-3.0.0.v201112011016.jar
       [INFO] Copying hk2-api-2.2.0-b21.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/hk2-api-2.2.0-b21.jar
       [INFO] Copying hk2-locator-2.2.0-b21.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/hk2-locator-2.2.0-b21.jar
       [INFO] Copying hk2-utils-2.2.0-b21.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/hk2-utils-2.2.0-b21.jar
       [INFO] Copying osgi-resource-locator-1.0.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/osgi-resource-locator-1.0.1.jar
       [INFO] Copying asm-all-repackaged-2.2.0-b21.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/asm-all-repackaged-2.2.0-b21.jar
       [INFO] Copying cglib-2.2.0-b21.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/cglib-2.2.0-b21.jar
       [INFO] Copying javax.inject-2.2.0-b21.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/javax.inject-2.2.0-b21.jar
       [INFO] Copying jersey-container-servlet-2.5.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jersey-container-servlet-2.5.1.jar
       [INFO] Copying jersey-container-servlet-core-2.5.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jersey-container-servlet-core-2.5.1.jar
       [INFO] Copying jersey-client-2.5.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jersey-client-2.5.1.jar
       [INFO] Copying jersey-common-2.5.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jersey-common-2.5.1.jar
       [INFO] Copying jersey-server-2.5.1.jar to /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/dependency/jersey-server-2.5.1.jar
       [INFO]
       [INFO] --- maven-install-plugin:2.3.1:install (default-install) @ simple-heroku-webapp ---
       [INFO] Installing /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/target/simple-heroku-webapp.war to /app/tmp/cache/.m2/repository/com/example/simple-heroku-webapp/1.0-SNAPSHOT/simple-heroku-webapp-1.0-SNAPSHOT.war
       [INFO] Installing /tmp/build_c46e8b05-3b71-4f86-8c01-d6824fbe7a62/pom.xml to /app/tmp/cache/.m2/repository/com/example/simple-heroku-webapp/1.0-SNAPSHOT/simple-heroku-webapp-1.0-SNAPSHOT.pom
       [INFO] ------------------------------------------------------------------------
       [INFO] BUILD SUCCESS
       [INFO] ------------------------------------------------------------------------
       [INFO] Total time: 13.762s
       [INFO] Finished at: Thu Jan 09 12:34:57 UTC 2014
       [INFO] Final Memory: 19M/514M
       [INFO] ------------------------------------------------------------------------
-----> Discovering process types
       Procfile declares types -> web

-----> Compressing... done, 76.0MB
-----> Launching... done, v7
       http://simple-heroku-webapp.herokuapp.com deployed to Heroku

To git@heroku.com:simple-heroku-webapp.git
   1fda15a..d76bb28  master -> master
{{< / highlight >}}

Now you can access your application at, for example: <a href="http://simple-heroku-webapp.herokuapp.com/myresource">http://simple-heroku-webapp.herokuapp.com/myresource</a>

{{< highlight bash "hl_lines=1" >}}
$ curl http://simple-heroku-webapp.herokuapp.com/myresource
Hello, Heroku!
{{< / highlight >}}

Or take a look at the [WADL][9] of the app, at <http://simple-heroku-webapp.herokuapp.com/application.wadl>:

{{< highlight xml >}}
<application xmlns="http://wadl.dev.java.net/2009/02">
    <doc xmlns:jersey="https://jersey.java.net/" jersey:generatedBy="Jersey: 2.5.1 2014-01-02 13:43:00"/>
    <doc xmlns:jersey="https://jersey.java.net/" jersey:hint="This is simplified WADL with user and core resources only. To get full WADL with extended resources use the query parameter detail. Link: http://simple-heroku-webapp.herokuapp.com/application.wadl?detail=true"/>
    <grammars/>
    <resources base="http://simple-heroku-webapp.herokuapp.com/">
        <resource path="myresource">
            <method id="getIt" name="GET">
                <response>
                    <representation mediaType="text/plain"/>
                </response>
            </method>
        </resource>
    </resources>
</application>
{{< / highlight >}}

## Resources

  * GitHub project: [mgajdos/jersey-simple-heroku-webapp][5] ([zip][10])
  * Deployed app: <a href="http://simple-heroku-webapp.herokuapp.com/myresource">http://simple-heroku-webapp.herokuapp.com/myresource</a>
  * Archetype on GitHub: [jersey/archetypes/jersey-heroku-webapp][11]

## Further reading

  * [Jersey User Guide][12]

 [1]: https://jersey.github.io/
 [2]: http://www.eclipse.org/jetty/
 [3]: http://heroku.com/
 [4]: https://github.com/jersey/jersey/tree/2.5.1/archetypes/jersey-heroku-webapp
 [5]: https://github.com/mgajdos/jersey-simple-heroku-webapp
 [6]: https://github.com/mgajdos/jersey-simple-heroku-webapp/blob/master/src/main/java/com/example/MyResource.java
 [7]: https://github.com/mgajdos/jersey-simple-heroku-webapp/blob/master/src/test/java/com/example/MyResourceTest.java
 [8]: https://jersey.github.io/documentation/latest/test-framework.html "Chapter 22. Jersey Test Framework"
 [9]: http://en.wikipedia.org/wiki/Web_Application_Description_Language
 [10]: https://github.com/mgajdos/jersey-simple-heroku-webapp/archive/master.zip
 [11]: https://github.com/jersey/jersey/tree/master/archetypes/jersey-heroku-webapp
 [12]: https://jersey.github.io/documentation/latest/user-guide.html