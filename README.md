# Vert.x 3 Application Deployment on Openshift using the DIY Cartbridge

This guide explains how a vert.x 3 application can be deployed on OpenShift using the 
[DIY cartbridge](https://developers.openshift.com/en/diy-overview.html).

Be aware that the DIY Cartbridge does not support 
[scalability](https://developers.openshift.com/en/managing-scaling.html#supported-cartridges). However if you just 
want to have you application running, it's very simple.

## Step 1 - Create your application on OpenShift

First login or sign up on [Openshift](https://openshift.redhat.com). Once done create a new application. If you don't 
have one already, you are invited to create one. You will have to upload a SSH Public Key to let you push to the Git 
repository hosting the application.

**1. Choose a type of application**
When selecting the _type of application_, select *Do-It-Yourself 0.1*. You can search for `diy` in the search box.
 
**2. Configure the application**
On the _configure the application_ form, enter the requested data. You don't have to enter a git repository url, 
Openshift is going to create one for you.
 
Once done, click on the `Create Application` button (on the bottom of the page). It takes a couple of seconds, so 
don't worry.

**3. Next steps**

Almost there. On this page, you can get the Git repository url. For example:

````
git clone ssh://55656c6050044659b50000ce@diy2-vertx3sample.rhcloud.com/~/git/diy2.git/
cd diy2/
````

Clone this repository on your system. It's where we are going to copy your application. If you have configured the 
public key to use, you may have to configure your `~/.ssh/config` file to pick up the right key. Open the `~/
.ssh/config` and add the following content (and don't forget to edit):

```
Host diy2-vertx3sample.rhcloud.com
    User 55656c6050044659b50000ce
    IdentityFile ~/.ssh/id_rsa_github  
```
    
Don't forget to edit the host, user and private key associated with the public key you have uploaded. The user and 
host can be found in the git url.     


**Note about deployment**

OpenShift is listening for Git pushes. On every push it _redeploys_ your application. In the context of this guide,
 we are going to push a Vert.x application packaged as a _fat_ jar. 
 
## Step 2 - Application preparation
 
Deploying on OpenShift imposes a couple of conventions. Your server must be bound to 
`OPENSHIFT_DIY_IP:OPENSHIFT_DIY_PORT`.

Let's assume these environment variables are passed as two Java system variables respectively `http.address` and `http
.port`. In that case your HTTP server should be initialized with:

````
@Override
public void start() throws Exception {
   vertx.createHttpServer().requestHandler(req -> { /* your code */ })
           // IMPORTANT : we need to use the port and address given by openshift.
           .listen(Integer.getInteger("http.port", 8080), System.getProperty("http.address"));
}   
````   

It's very important to use the OpenShift values. Your application is not allowed to bind any other socket.
  
Once you have updated your application, you need to package it as a fat jar. This is what will be deployed on 
OpenShift.  
 
  
## Step 3 - Preparing the environment and deployment
  
**Clone the repository**
  
You should have cloned the Git repository, and get such kind of content:
  
```
.
├── .openshift
│   ├── README.md
│   ├── action_hooks
│   │   ├── README.md
│   │   ├── start
│   │   └── stop
│   ├── cron
│   └── markers
├── README.md
├── diy
│   ├── index.html
│   └── testrubyserver.rb
└── misc
    └── .gitkeep
```     

(for sake of simplicity, only relevant files are shown)

Let's do some cleanup. Delete the `diy` and `misc` directory, and create an `application` directory.

```
rm -Rf diy misc
mkdir application
```

Your application is going to be copied into the `application` directory.
 
**Copy the scripts**

There are three scripts to copy to `.openshift/action_hooks`:
  
* `pre_start` - provision Java 8: [link](https://raw.githubusercontent.com/cescoffier/vertx-openshift/initial-work/diy/template/.openshift/action_hooks/pre_start)
* `start` - Start the application: [link](https://raw.githubusercontent.com/cescoffier/vertx-openshift/initial-work/diy/template/.openshift/action_hooks/start) 
* `stop` - Stop the application: [link](https://raw.githubusercontent.com/cescoffier/vertx-openshift/initial-work/diy/template/.openshift/action_hooks/stop)   

These scripts can be copied from this template. 

Don't forget to put the execution permission on these scripts:

```
cd .openshift/action_hooks
chmod +x pre_start start stop
```

The `pre_start` script is required because Openshift does not provides a Java 8 JVM, so it installs one. The `start` 
script uses the installed JVM and launches the application. It redirects application output to `logs/out.log` and 
configure the `http.port` and `http.address` system properties. The `stop` script just stops the application.

**Copy the application**

The application's fat jar needs to be copied to the `application` directory.

**Final review before deployment**

So, you should get the following structure:

```
.
├── .openshift
│   ├── README.md
│   ├── action_hooks
│   │   ├── README.md
│   │   ├── pre_start
│   │   ├── start
│   │   └── stop
│   ├── cron
│   └── markers
├── README.md
└── application
    └── vertx-sample-3.0.0-SNAPSHOT-fat.jar  <= your fat jar
```

**Deployment**

It's deployment time. Deploying on Openshift is just a `git push`.

```
git add -A
git commit -m "initial version of my application"
git push
```

The first _push_ take some time as the Java 8 JVM is installed. Don't worry, the script does this only once.

Here you are, you application should have started. If you go back to OpenShift and navigate to your application page,
 it gives you the root url. this url should be served by your application.
    