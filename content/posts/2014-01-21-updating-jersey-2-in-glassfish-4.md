---
title: Updating Jersey 2 in GlassFish 4
author: Michal Gajdoš
type: post
date: 2014-01-21T13:35:26+00:00
url: /updating-jersey-2-in-glassfish-4/
aliases: /2014/01/21/updating-jersey-2-in-glassfish-4/
categories:
  - Jersey
tags:
  - jax-rs
  - jersey

---
Different life-cycles of Jersey 2 and GlassFish 4 arise a question how to make sure that ones GlassFish instance contains always the latest version of Jersey. This question is even more important in case you don&#8217;t want to download the nightly/promoted build every-time a new version of Jersey is released but you still want to use the latest and greatest Jersey.

> Note: The script below is not compatible with Jersey 2.6 at the moment. I&#8217;ll update it as soon as possible.

<!--more-->

In this article we&#8217;ll see if your GlassFish installation is ready for update and what steps you need to take to actually update it:

  * [Jersey in GlassFish nightly][1]
  * [What Jersey am I using?][2]
  * [Is it possible to update Jersey in my GlassFish?][3]
  * [Updating Jersey in GF 4.0.1][4]
  * [What if GlassFish with updated Jersey stinks?][5]

## <a name="gf-nightly"></a>Jersey in GlassFish nightly

New version of Jersey is released approximately once in 4-5 weeks (see <a href="https://java.net/jira/browse/JERSEY#selectedTab=com.atlassian.jira.plugin.system.project%3Aroadmap-panel">road-map</a>). Together with releasing a new version of Jersey (announced on our <a href="https://jersey.github.io/mailing.html">mailing list</a> or via <a href="https://twitter.com/gf_jersey">@gf_jersey</a> on Twitter) we&#8217;re also integrating Jersey with GlassFish trunk (currently version 4.0.1) to make sure nothing gets broken. This means that all major/minor Jersey versions are ready to be used in the nightly build of GlassFish within few days after the release (usually it takes 1-2 day to have a version of GlassFish with the latest Jersey). The nightly build of GlassFish can be downloaded from:

  * <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/nightly/latest-glassfish-ml.zip">http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/nightly/latest-glassfish-ml.zip</a>

## <a name="jersey-version"></a>What Jersey am I using?

People are often curious what Jersey version do they use in a GlassFish instance they&#8217;d downloaded. The simplest way to find out is by running the following command in your shell:

{{< highlight bash "hl_lines=5" >}}
# make sure we're in glassfish4/glassfish
$ pwd
/.../mgajdos/blog/update-jersey-in-gf/glassfish4/glassfish

$ unzip -p modules/jersey-common.jar META-INF/MANIFEST.MF | grep Bundle-Version
Bundle-Version: 2.0.0
{{< / highlight >}}

From the output we can see that the _Bundle-Version_ is _2.0.0_ which means that the GlassFish I have contains Jersey 2.0 (released in May 2013).

Other quick way to find out is by deploying an JAX-RS application and taking a look into the logs where you ought to find a line similar to:

{{< highlight bash "hl_lines=2" >}}
[2014-01-19T18:32:48.384+0100] [glassfish 4.0] [INFO] [] [org.glassfish.jersry.server.ApplicationHandler] [tid: _ThreadID=16 _ThreadName=RunLevelControllerThread-1390152765826] [timeMillis: 1390152768384] [levelValue: 800] [[
  Initiating Jersey application, version Jersey: 2.5.1 2014-01-02 13:43:00...]]
{{< / highlight >}}

As you can see my other instance of GlassFish contains Jersey 2.5.1.

## <a name="can-i-update"></a>Is it possible to update Jersey in my GlassFish?

Unfortunately it&#8217;s not possible to update Jersey 2.0 in release of GlassFish 4.0 (or Java EE 7 RI GlassFish) downloaded from <a href="https://glassfish.java.net/download.html">here</a>. HK2 (dependency injection framework Jersey uses) went through some big changes between versions 2.1.x and 2.2.0 that affected also the core GlassFish classes and because of this fact the simple change of jars does not work in this case.

To make sure you&#8217;re able to update Jersey in your GlassFish you need to check whether you&#8217;re using version 4.0.1.

{{< highlight bash "hl_lines=5" >}}
# make sure we're in glassfish4/glassfish
$ pwd
/.../mgajdos/blog/update-jersey-in-gf/glassfish4/glassfish

$ unzip -p modules/glassfish.jar META-INF/MANIFEST.MF | grep Bundle-Version
Bundle-Version: 4.0.1.b01
{{< / highlight >}}

Again see the _Bundle-Version_ which is _4.0.1.b01_ (first promoted build of 4.0.1).

For each version of Jersey that is in one of the promoted or nightly builds of 4.0.1 it is possible to update to the latest version of Jersey (currently 2.5.1). Here are the builds that can be updated:

  * <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/nightly/latest-glassfish-ml.zip">latest nightly build</a> (contains the latest version of Jersey)
  * <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/promoted/latest-glassfishml.zip">latest promoted build</a> (contains Jersey 2.4.1)
  * <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/promoted/glassfish-4.0.1-b03-ml.zip">4.0.1-b03</a> (contains Jersey 2.2)
  * <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/promoted/glassfish-4.0.1-b02-ml.zip">4.0.1-b02</a> (contains Jersey 2.2)
  * <a href="http://dlc.sun.com.edgesuite.net/glassfish/4.0.1/promoted/glassfish-4.0.1-b01-ml.zip">4.0.1-b01</a> (contains Jersey 2.0)

## <a name="gf-update"></a>Updating Jersey in GF 4.0.1

Basically, to update Jersey there are five things that need to be done:

  * stop GF domain
  * replace Jersey bits
  * replace HK2  bits (dependency injection framework) to the one Jersey uses
  * erase OSGi cache
  * start GF domain

> Erasing OSGi cache (glassfish4/glassfish/domains/domain/osgi-cache/felix) is a good practice in case you&#8217;re trying to replace one or more modules that are present in GF and you don&#8217;t want to see weird OSGi exceptions.

Since the number of Jersey and HK2 bits in GlassFish is pretty high (~18 for Jersey and ~16 for HK2 and it&#8217;s repackaged modules) I&#8217;ve prepared a simple script that takes care of downloading these libraries for you (incl. backing-up the ones you already got, just in case) and erasing the OSGi cache. Few comments on the script:

  * versions of Jersey and HK2 are hard-coded
  * it needs to be executed from `glassfish4/glassfish` directory
  * domain should be located at  `glassfish4/glassfish/domains/domain1`

The whole script (see this <a href="https://gist.github.com/mgajdos/8522248">Gist</a>):

{{< highlight bash >}}
#!/bin/bash

JERSEY_VERSION=2.5.1
HK2_VERSION=2.2.0-b26
JAVASSIST_VERSION=3.18.1-GA

MODULES_DIR=`pwd`/modules
OSGI_CACHE_DIR=`pwd`/domains/domain1/osgi-cache/felix

processArtifact() {
    # Backup atifact
    if [ ! -f ${MODULES_DIR}/$1.bak ]; then
        echo "Backuping $1 ..."
        cp ${MODULES_DIR}/$1 ${MODULES_DIR}/$1.bak
    fi

    # Download new version 
    echo "Downloading $1 ..."

    if curl -s -o ${MODULES_DIR}/$1 $2 ; then
        echo "Downloaded $1"
    else
        echo "Error downloading $1"
    fi

    return 0
}

# Backup and download new Jersey

processArtifact jersey-gf-cdi.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/containers/glassfish/jersey-gf-cdi/${JERSEY_VERSION}/jersey-gf-cdi-${JERSEY_VERSION}.jar
processArtifact jersey-gf-ejb.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/containers/glassfish/jersey-gf-ejb/${JERSEY_VERSION}/jersey-gf-ejb-${JERSEY_VERSION}.jar
processArtifact jersey-container-grizzly2-http.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/containers/jersey-container-grizzly2-http/${JERSEY_VERSION}/jersey-container-grizzly2-http-${JERSEY_VERSION}.jar
processArtifact jersey-container-servlet-core.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/containers/jersey-container-servlet-core/${JERSEY_VERSION}/jersey-container-servlet-core-${JERSEY_VERSION}.jar
processArtifact jersey-container-servlet.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/containers/jersey-container-servlet/${JERSEY_VERSION}/jersey-container-servlet-${JERSEY_VERSION}.jar
processArtifact jersey-client.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/core/jersey-client/${JERSEY_VERSION}/jersey-client-${JERSEY_VERSION}.jar
processArtifact jersey-common.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/core/jersey-common/${JERSEY_VERSION}/jersey-common-${JERSEY_VERSION}.jar
processArtifact jersey-server.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/core/jersey-server/${JERSEY_VERSION}/jersey-server-${JERSEY_VERSION}.jar
processArtifact jersey-bean-validation.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/ext/jersey-bean-validation/${JERSEY_VERSION}/jersey-bean-validation-${JERSEY_VERSION}.jar
processArtifact jersey-entity-filtering.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/ext/jersey-entity-filtering/${JERSEY_VERSION}/jersey-entity-filtering-${JERSEY_VERSION}.jar
processArtifact jersey-mvc-jsp.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/ext/jersey-mvc-jsp/${JERSEY_VERSION}/jersey-mvc-jsp-${JERSEY_VERSION}.jar
processArtifact jersey-mvc.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/ext/jersey-mvc/${JERSEY_VERSION}/jersey-mvc-${JERSEY_VERSION}.jar
processArtifact jersey-media-json-jackson.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/media/jersey-media-json-jackson/${JERSEY_VERSION}/jersey-media-json-jackson-${JERSEY_VERSION}.jar
processArtifact jersey-media-json-jettison.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/media/jersey-media-json-jettison/${JERSEY_VERSION}/jersey-media-json-jettison-${JERSEY_VERSION}.jar
processArtifact jersey-media-json-processing.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/media/jersey-media-json-processing/${JERSEY_VERSION}/jersey-media-json-processing-${JERSEY_VERSION}.jar
processArtifact jersey-media-moxy.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/media/jersey-media-moxy/${JERSEY_VERSION}/jersey-media-moxy-${JERSEY_VERSION}.jar
processArtifact jersey-media-multipart.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/media/jersey-media-multipart/${JERSEY_VERSION}/jersey-media-multipart-${JERSEY_VERSION}.jar
processArtifact jersey-media-sse.jar http://repo.maven.apache.org/maven2/org/glassfish/jersey/media/jersey-media-sse/${JERSEY_VERSION}/jersey-media-sse-${JERSEY_VERSION}.jar

# Backup and download new HK2

processArtifact hk2-api.jar http://central.maven.org/maven2/org/glassfish/hk2/hk2-api/${HK2_VERSION}/hk2-api-${HK2_VERSION}.jar
processArtifact class-model.jar http://central.maven.org/maven2/org/glassfish/hk2/class-model/${HK2_VERSION}/class-model-${HK2_VERSION}.jar
processArtifact core.jar http://central.maven.org/maven2/org/glassfish/hk2/core/${HK2_VERSION}/core-${HK2_VERSION}.jar
processArtifact hk2-locator.jar http://central.maven.org/maven2/org/glassfish/hk2/hk2-locator/${HK2_VERSION}/hk2-locator-${HK2_VERSION}.jar
processArtifact hk2-utils.jar http://central.maven.org/maven2/org/glassfish/hk2/hk2-utils/${HK2_VERSION}/hk2-utils-${HK2_VERSION}.jar
processArtifact hk2.jar http://central.maven.org/maven2/org/glassfish/hk2/hk2/${HK2_VERSION}/hk2-${HK2_VERSION}.jar
processArtifact hk2-runlevel.jar http://central.maven.org/maven2/org/glassfish/hk2/hk2-runlevel/${HK2_VERSION}/hk2-runlevel-${HK2_VERSION}.jar
processArtifact hk2-config.jar http://central.maven.org/maven2/org/glassfish/hk2/hk2-config/${HK2_VERSION}/hk2-config-${HK2_VERSION}.jar
processArtifact osgi-adapter.jar http://central.maven.org/maven2/org/glassfish/hk2/osgi-adapter/${HK2_VERSION}/osgi-adapter-${HK2_VERSION}.jar

processArtifact bean-validator-cdi.jar http://central.maven.org/maven2/org/glassfish/hk2/external/bean-validator-cdi/${HK2_VERSION}/bean-validator-cdi-${HK2_VERSION}.jar
processArtifact bean-validator.jar http://central.maven.org/maven2/org/glassfish/hk2/external/bean-validator/${HK2_VERSION}/bean-validator-${HK2_VERSION}.jar
processArtifact aopalliance-repackaged.jar http://central.maven.org/maven2/org/glassfish/hk2/external/aopalliance-repackaged/${HK2_VERSION}/aopalliance-repackaged-${HK2_VERSION}.jar
processArtifact asm-all-repackaged.jar http://central.maven.org/maven2/org/glassfish/hk2/external/asm-all-repackaged/${HK2_VERSION}/asm-all-repackaged-${HK2_VERSION}.jar
processArtifact javax.inject.jar http://central.maven.org/maven2/org/glassfish/hk2/external/javax.inject/${HK2_VERSION}/javax.inject-${HK2_VERSION}.jar

processArtifact javassist.jar http://repo.maven.apache.org/maven2/org/javassist/javassist/${JAVASSIST_VERSION}/javassist-${JAVASSIST_VERSION}.jar

# Clean up OSGi cache

if [ -d "$OSGI_CACHE_DIR" ]; then
  rm -rf $OSGI_CACHE_DIR
fi
{{< / highlight >}}

You can download it from <a href="https://gist.github.com/mgajdos/8522248/raw">here</a>, i.e.:

{{< highlight bash >}}
$ cd glasshfish4/glassfish
$ curl -s -k -o update.sh https://gist.github.com/mgajdos/8522248/raw
$ chmod +x update.sh
{{< / highlight >}}

> In case you&#8217;re experiencing problems after updating Jersey replace new modules with backed-up versions (_**.bak**_ suffix).

After running this script you&#8217;ll see an output similar to:

{{< highlight bash >}}
$ ./update.sh
Backuping jersey-gf-cdi.jar ...
Downloading jersey-gf-cdi.jar ...
Downloaded jersey-gf-cdi.jar
Backuping jersey-gf-ejb.jar ...
Downloading jersey-gf-ejb.jar ...
Downloaded jersey-gf-ejb.jar
Backuping jersey-container-grizzly2-http.jar ...
Downloading jersey-container-grizzly2-http.jar ...
Downloaded jersey-container-grizzly2-http.jar
Backuping jersey-container-servlet-core.jar ...
Downloading jersey-container-servlet-core.jar ...
Downloaded jersey-container-servlet-core.jar
Backuping jersey-container-servlet.jar ...
Downloading jersey-container-servlet.jar ...
Downloaded jersey-container-servlet.jar
Backuping jersey-client.jar ...
Downloading jersey-client.jar ...
Downloaded jersey-client.jar
Backuping jersey-common.jar ...
Downloading jersey-common.jar ...
Downloaded jersey-common.jar
Backuping jersey-server.jar ...
Downloading jersey-server.jar ...
Downloaded jersey-server.jar
Backuping jersey-bean-validation.jar ...
Downloading jersey-bean-validation.jar ...
Downloaded jersey-bean-validation.jar
Backuping jersey-entity-filtering.jar ...
cp: /.../mgajdos/blog/update-jersey-in-gf/promoted/glassfish4/glassfish/modules/jersey-entity-filtering.jar: No such file or directory
Downloading jersey-entity-filtering.jar ...
Downloaded jersey-entity-filtering.jar
Backuping jersey-mvc-jsp.jar ...
Downloading jersey-mvc-jsp.jar ...
Downloaded jersey-mvc-jsp.jar
Backuping jersey-mvc.jar ...
Downloading jersey-mvc.jar ...
Downloaded jersey-mvc.jar
Backuping jersey-media-json-jackson.jar ...
Downloading jersey-media-json-jackson.jar ...
Downloaded jersey-media-json-jackson.jar
Backuping jersey-media-json-jettison.jar ...
Downloading jersey-media-json-jettison.jar ...
Downloaded jersey-media-json-jettison.jar
Backuping jersey-media-json-processing.jar ...
Downloading jersey-media-json-processing.jar ...
Downloaded jersey-media-json-processing.jar
Backuping jersey-media-moxy.jar ...
Downloading jersey-media-moxy.jar ...
Downloaded jersey-media-moxy.jar
Backuping jersey-media-multipart.jar ...
Downloading jersey-media-multipart.jar ...
Downloaded jersey-media-multipart.jar
Backuping jersey-media-sse.jar ...
Downloading jersey-media-sse.jar ...
Downloaded jersey-media-sse.jar
Backuping hk2-api.jar ...
Downloading hk2-api.jar ...
Downloaded hk2-api.jar
Backuping class-model.jar ...
Downloading class-model.jar ...
Downloaded class-model.jar
Backuping core.jar ...
Downloading core.jar ...
Downloaded core.jar
Backuping hk2-locator.jar ...
Downloading hk2-locator.jar ...
Downloaded hk2-locator.jar
Backuping hk2-utils.jar ...
Downloading hk2-utils.jar ...
Downloaded hk2-utils.jar
Backuping hk2.jar ...
Downloading hk2.jar ...
Downloaded hk2.jar
Backuping hk2-runlevel.jar ...
Downloading hk2-runlevel.jar ...
Downloaded hk2-runlevel.jar
Backuping hk2-config.jar ...
Downloading hk2-config.jar ...
Downloaded hk2-config.jar
Backuping osgi-adapter.jar ...
Downloading osgi-adapter.jar ...
Downloaded osgi-adapter.jar
Backuping bean-validator-cdi.jar ...
Downloading bean-validator-cdi.jar ...
Downloaded bean-validator-cdi.jar
Backuping bean-validator.jar ...
Downloading bean-validator.jar ...
Downloaded bean-validator.jar
Backuping aopalliance-repackaged.jar ...
cp: /.../mgajdos/blog/update-jersey-in-gf/promoted/glassfish4/glassfish/modules/aopalliance-repackaged.jar: No such file or directory
Downloading aopalliance-repackaged.jar ...
Downloaded aopalliance-repackaged.jar
Backuping asm-all-repackaged.jar ...
Downloading asm-all-repackaged.jar ...
Downloaded asm-all-repackaged.jar
Backuping javax.inject.jar ...
Downloading javax.inject.jar ...
Downloaded javax.inject.jar
Backuping javassist.jar ...
cp: /.../mgajdos/blog/update-jersey-in-gf/promoted/glassfish4/glassfish/modules/javassist.jar: No such file or directory
Downloading javassist.jar ...
Downloaded javassist.jar
{{< / highlight >}}

Now, let&#8217;s try to deploy <a href="http://central.maven.org/maven2/org/glassfish/jersey/examples/bean-validation-webapp/2.5.1/bean-validation-webapp-2.5.1-gf-project-src.zip">bean-validation-webapp</a> sample to see if our GlassFish really uses the latest (2.5.1) version of Jersey:

{{< highlight bash "hl_lines=2" >}}
[2014-01-20T18:52:41.110+0100] [glassfish 4.0] [INFO] [] [org.glassfish.jersey.server.ApplicationHandler] [tid: _ThreadID=69 _ThreadName=AutoDeployer] [timeMillis: 1390236761110] [levelValue: 800] [[
  Initiating Jersey application, version Jersey: 2.5.1 2014-01-02 13:43:00...]]
{{< / highlight >}}

Enjoy!

## <a name="gf-stinks"></a>What if GlassFish with updated Jersey stinks?

Revert the process. The previous script also makes a backup copy of all jars it&#8217;s replacing. The script in this <a href="https://gist.github.com/mgajdos/8525480">Gist</a> restores the jars from backup if needed.

{{< highlight bash >}}
$ cd glasshfish4/glassfish
$ curl -s -k -o restore.sh https://gist.github.com/mgajdos/8525480/raw
$ chmod +x restore.sh
{{< / highlight >}}

Run it.

{{< highlight bash >}}
$ ./restore.sh
Restoring jersey-gf-cdi.jar ...
Restoring jersey-gf-ejb.jar ...
Restoring jersey-container-grizzly2-http.jar ...
Restoring jersey-container-servlet-core.jar ...
Restoring jersey-container-servlet.jar ...
Restoring jersey-client.jar ...
Restoring jersey-common.jar ...
Restoring jersey-server.jar ...
Restoring jersey-bean-validation.jar ...
Restoring jersey-mvc-jsp.jar ...
Restoring jersey-mvc.jar ...
Restoring jersey-media-json-jackson.jar ...
Restoring jersey-media-json-jettison.jar ...
Restoring jersey-media-json-processing.jar ...
Restoring jersey-media-moxy.jar ...
Restoring jersey-media-multipart.jar ...
Restoring jersey-media-sse.jar ...
Restoring hk2-api.jar ...
Restoring class-model.jar ...
Restoring core.jar ...
Restoring hk2-locator.jar ...
Restoring hk2-utils.jar ...
Restoring hk2.jar ...
Restoring hk2-runlevel.jar ...
Restoring hk2-config.jar ...
Restoring osgi-adapter.jar ...
Restoring bean-validator-cdi.jar ...
Restoring bean-validator.jar ...
Restoring asm-all-repackaged.jar ...
Restoring javax.inject.jar ...
{{< / highlight >}}

And make sure you have old Jersey jars restored.

{{< highlight bash "hl_lines=2" >}}
[2014-01-20T19:03:43.478+0100] [glassfish 4.0] [INFO] [] [org.glassfish.jersey.server.ApplicationHandler] [tid: _ThreadID=39 _ThreadName=admin-listener(1)] [timeMillis: 1390241023478] [levelValue: 800] [[
  Initiating Jersey application, version Jersey: 2.2 2013-08-14 08:51:58...]]
{{< / highlight >}}

 [1]: #gf-nightly
 [2]: #jersey-version
 [3]: #can-i-update
 [4]: #gf-update
 [5]: #gf-stinks