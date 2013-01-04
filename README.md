# Vert.x Sample Application on CloudFoundry

This is a sample project that shows how to run a vert.x application on CloudFoundry. The running application can be seen at [http://vtoons.cloudfoundry.com/](http://vtoons.cloudfoundry.com/).

> NOTE: This sample has been tested using cloudfoundry-runtime 0.8.2 and vert.x 1.2.3.final (Jan 2, 2013)

## vert.x

[vert.x](http://vertx.io/) is a framework for for developing asynchronous event-driven applications in a variety of languages, including JavaScript, Ruby, Groovy, Java, Python and Coffeescript.

## CloudFoundry

CloudFoundry recently added support for [stand-alone applications](http://blog.cloudfoundry.com/2012/05/11/running-standalone-web-applications-on-cloud-foundry/), which includes the ability to run applications in containers and frameworks that are not yet fully supported by CloudFoundry. 

Even more recently, CloudFoundry added support for running Java 7 applications. vert.x requires a Java 7 runtime environment, so the addition of Java 7 support now makes it possible to run vert.x applications on CloudFoundry. 

## Sample Application

The vert.x web site include a set of [tutorials](http://vertx.io/tutorials.html) in several languages. This sample app is based on the [Groovy tutorial application](http://vertx.io/groovy_web_tutorial.html). For more details on this application and how to use it, refer to the tutorial pages. The rest of this README will only refer to changes made to the tutorial app to support CloudFoundry. 

### Configuration for CloudFoundry

#### CloudFoundry runtime support

We can add support to detect runtime environment settings using the [Java cloudfoundry-runtime ](https://github.com/cloudfoundry/vcap-java/tree/master/cloudfoundry-runtime) library.

    import org.cloudfoundry.runtime.env.CloudEnvironment
    import org.cloudfoundry.runtime.env.MongoServiceInfo

    def cloudEnv = new CloudEnvironment()

#### Web Server

For simplicity, the configuration of the web server in the tutorial application is hard-coded to 'localhost:8080':

    def webServerConf = [
        port: 8080,
        host: 'localhost',
        ssl: true,
    ...
    
    ]

In order to run on CloudFoundry, the application must use the host name and port provided by CloudFoundry. This code reads the host name and port from the CloudEnvironment instance, defaulting to 'localhost:8080' if the CloudFoundry environment is not detected:

    def webServerConf = [
        port: (cloudEnv.getValue('VCAP_APP_PORT') ?: '8080') as int,
        host: cloudEnv.getValue('VCAP_APP_HOST') ?: 'localhost',
        /* ssl: true, */
    ...
    
    ]

Note: In order to simplify this example, the SSL support was removed from the application.

#### MongoDB

The tutorial app uses MongoDB as a back-end data store. No configuration is given for the MongoDB connection, so the default host and port are used by the MongoDB vert.x module. 

When the application is pushed to CloudFoundry, it is bound to a MongoDB service. The connection parameters for the MongoDB service must also be read from the environment using the MongoServiceInfo class, defaulting if the CloudFoundry environment is not detected:

    // Configuration for MongoDb 
    def mongoConf = [:]

    if (cloudEnv.isCloudFoundry()) {
      mongoSvcInfo = cloudEnv.getServiceInfo("mongodb-vtoons", MongoServiceInfo.class)
      mongoConf.host = mongoSvcInfo.getHost()
      mongoConf.port = mongoSvcInfo.getPort() as int
      mongoConf.db_name = mongoSvcInfo.getDatabase()
      mongoConf.username = mongoSvcInfo.getUserName()
      mongoConf.password = mongoSvcInfo.getPassword()
    }

### vert.x Distribution

A full vert.x distribution (available on the [Downloads](http://vertx.io/downloads.html) page of the vert.x web site) must be pushed to CloudFoundry along with the application (we used the 1.2.3.final version of vert.x for this app). This project uses the following directories from your downloaded vert.x distribution:

* bin
* conf
* lib

We are also using the [Java cloudfoundry-runtime so we need to include this jar](https://repo.springsource.org/simple/libs-milestone-s3-cache/org/cloudfoundry/cloudfoundry-runtime/0.8.1/cloudfoundry-runtime-0.8.2.jar).

### vert.x module

To simplify the packaging we will move the application code to a vert.x module in the 'src' directory. This includes adding a mod.json file listing the main application file. You can see this change in this [commit](https://github.com/cloudfoundry-samples/vertx-vtoons/commit/54e927f4e6191453d4607fedcc933e205eaf606a) in the repository.

### Gradle build script

While we could package the application by copying a few directories from vert.x distribution and then using ‘vmc push’ as usual, let’s simplify by creating a Gradle build script as seen [here](https://github.com/cloudfoundry-samples/vertx-vtoons/blob/master/build.gradle).

This build script provides the following:

* Defines repository locations for Maven Central, Spring milestone releases and vert.x release distributions
* Defines two configurations, one for runtime and one for dependent jars, as well as their dependencies
* Provides a “runtime” task that downloads and packages the vert/x runtime
* Provides a “build” task that packages the app source into a module directory
* Provides an “assemble” task packages the application as a zip file to be deployed
* Provides a “clean” task to clean up the build directory
* Configures the Gradle CloudFoundry plugin so we can use the build script to deploy the app

## Pushing the Application to CloudFoundry

With everything in place, let’s just build and deploy the app:

    $ ./gradlew assemble
    :runtime
    Download http://vertx.io/downloads/vert.x-1.2.3.final.zip
    :build
    Download http://repo.springsource.org/libs-snapshot/org/cloudfoundry/cloudfoundry-runtime/0.8.2/cloudfoundry-runtime-0.8.2.pom
    :assemble

    BUILD SUCCESSFUL

    Total time: 18.586 secs


    $ ./gradlew cf-add-service -PcfUser=user@email.com -PcfPasswd=secret -PcfTargetBase=cloudfoundry.com
    :cf-add-service
    CloudFoundry - Connecting to 'http://api.cloudfoundry.com' with user 'user@email.com'
    CloudFoundry - Provisioning mongodb service 'mongodb-vtoons'

    BUILD SUCCESSFUL

    Total time: 7.435 secs

    $ ./gradlew cf-push -PcfUser=user@email.com -PcfPasswd=secret -PcfTargetBase=cloudfoundry.com
    :cf-push
    CloudFoundry - Connecting to 'http://api.cloudfoundry.com' with user 'user@email.com'
    GET request for "http://api.cloudfoundry.com/apps/vtoons" resulted in 404 (Not Found); invoking error handler
    CloudFoundry - Creating standalone application 'vtoons' with runtime java7
    CloudFoundry - Deploying '/Users/trisberg/Projects/github/cloudfoundry-samples/vertx-vtoons/build/vertx-vtoons.zip'
    CloudFoundry - Starting 'vtoons'

    BUILD SUCCESSFUL

    Total time: 21.358 secs


Of course, change cloudfoundry.com to reflect the base of your domain, for example mydm.cloudfoundry.me
It will use vcap.me if cfTargetBase argument value is not provided on the command line


If you prefer to create your own zip file and use vmc for deployments here are some hints:

The zip file layout used is the following (only showing directories and some application files):

    +- mods
       +- vtoons
          +- lib
             cloudfoundry-runtime-0.8.2.jar
          +- web
             +- css
             +- js
             +- js3rdparty
             index.html
          App.groovy
          mod.json
          server-keystore.jks
          StaticData.groovy
    +- vert.x-1.2.3.final
       (the vert.x distribution)

The 'vmc push' choices would be these:

* Choose 'Standalone Application'
* Choose 'java7' runtime
* Set the 'Start Command' to 'vert.x-1.2.3.final/bin/vertx runmod vtoons'
* Set the 'Memory reservation' to at least '256M'
* Set the url to something other than 'vtoons.cloudfoundry.com' since that is taken
* Create a new MongoDB service named 'mongodb-vtoons' to bind to the application

