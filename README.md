# Vert.x Sample Application on CloudFoundry

This is a sample project that shows how to run a vert.x application on CloudFoundry. The running application can be seen at [http://vtoons.cloudfoundry.com/](http://vtoons.cloudfoundry.com/).

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

For simplicity, the configuration of the web server in the tutorial application is hard-coded to `localhost:8080`:

    def webServerConf = [
        port: 8080,
        host: 'localhost',
    
    ...
    
    ]

In order to run on CloudFoundry, the application must use the host name and port provided by CloudFoundry. This code reads the host name and port from the CloudEnvironment instance, defaulting to `localhost:8080` if the CloudFoundry environment is not detected:

    def webServerConf = [
        port: (cloudEnv.getValue('VCAP_APP_PORT') ?: '8080') as int,
        host: cloudEnv.getValue('VCAP_APP_HOST') ?: 'localhost',
    
    ...
    
    ]

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

### SSL

SSL and HTTPS on CloudFoundry is tricky. In order to simplify this example, the SSL support was removed from the example application. 

### vert.x Distribution

A full vert.x distribution (available on the [Downloads](http://vertx.io/downloads.html) page of the vert.x web site) must be pushed to CloudFoundry along with the application. This project has a `vert.x` directory where you will need to copy the following directories from your downloaded vert.x distribution:

* bin
* conf
* lib

We are also using the [Java cloudfoundry-runtime so we need to download this jar](https://repo.springsource.org/simple/libs-milestone-s3-cache/org/cloudfoundry/cloudfoundry-runtime/0.8.1/cloudfoundry-runtime-0.8.1.jar)  and put it in the vert.x/lib directory that we just created.

## Pushing the Application to CloudFoundry

Since this sample application uses Groovy, no compilation of the application files is required. The application files, along with the vert.x distribution can be pushed to CloudFoundry using the `vmc push` command: 

    > vmc push vtoons
    Would you like to deploy from the current directory? [Yn]: y
    Detected a Standalone Application, is this correct? [Yn]: y
    1: java
    2: java7
    3: node
    4: node06
    5: ruby18
    6: ruby19
    Select Runtime [ruby18]: 2
    Selected java7
    Start Command: vert.x/bin/vertx run App.groovy
    Application Deployed URL [None]: vtoons.cloudfoundry.com
    Memory reservation (128M, 256M, 512M, 1G, 2G) [64M]: 256M
    How many instances? [1]: 
    Bind existing services to 'vt2'? [yN]: n
    Create services to bind to 'vt2'? [yN]: y
    1: mongodb
    2: mysql
    3: postgresql
    4: rabbitmq
    5: redis
    What kind of service?: 1
    Specify the name of the service [mongodb-771fd]: mongodb-vtoons
    Create another? [yN]: n
    Would you like to save this configuration? [yN]: y
    Manifest written to manifest.yml.
    Creating Application: OK
    Binding Service [mongodb-vtoons]: OK
    Uploading Application:
        Checking for available resources: OK
        Processing resources: OK
        Packing application: OK
        Uploading (43K): OK   
    Push Status: OK
    Staging Application 'vtoons': OK                                                   
    Starting Application 'vtoons': OK

The important responses the `vmc push` prompts are these:

* Choose `Standalone Application`
* Choose `java7` runtime
* Set the `Start Command` as shown
* Set the `Memory reservation` to at least `256M`
* Create a new MongoDB service to bind the application to

Note that if you copy this project and try to push it, you will have to change the url to something other than `vtoons.cloudfoundry.com`.
