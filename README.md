# Blue/Green binary deployment to Tomcat on OpenShift v3.3

## Deploy from testing to production.

### Configuration

#### Set the following variables to match your enviroment.

```
OPENSHIFT_API_SERVER=10.1.2.2:8443
SUBDOMAIN=rhel-cdk.10.1.2.2.xip.io
```
Clone this git repo locally.

```
git clone https://github.com/bkoz/tomcatbinbg.git
cd tomcatbinbg
cp wars/blue.war source/deployments/ROOT.war
```
Login to OpenShift, create a project, a new build and start the build.

```
oc login $OPENSHIFT_API_SERVER
oc new-project bgwar
```

#### Create the test enviroment.
```
oc new-build --image-stream=jboss-webserver30-tomcat7-openshift --name=myapp --binary=true
oc start-build myapp --from-dir=source --follow --wait
```

Wait for the build and registry push to suceed.

```
Pushing image <registry-service-ip>:5000/bgwar/myapp:latest ...
...
Pushed 7/7 layers, 100% complete
Push successful
```
#### Deploy Blue into testing.

Create a deployment config. Replace `<registry-service-ip>`
with what IP is returned from the logs output above.

```
oc get is

oc create deploymentconfig testing --image=<registry-service-ip>:5000/bgwar/myapp:latest
```

Wait for the Catalina Server to finish starting up.

```
oc logs dc/testing --follow
...
$ oc logs dc/testing --follow
...
...
2016-12-02 18:02:28,583 [main] INFO  org.apache.catalina.startup.Catalina- Server startup in 6322 ms
```

Expose the testing application and create a route for it.

```
oc expose dc testing --port=8080
oc expose svc testing --name=testing 
```
Get the hostname of the route and visit your application using `curl` or a web browser.

```
oc get route
```
```
NAME      HOST/PORT                      PATH      SERVICES   PORT      TERMINATION
testing   testing.rhel-cdk.10.1.2.2.xip.io         testing    8080      
```

```
$ curl --silent testing.$SUBDOMAIN | grep jpg
<img src="images/bluerose.jpg">
$
```

#### Create the production environment.

Create the production deployment config. This will deploy using the `production` image stream tag. 

```
oc create deploymentconfig production --image=<registry-service-ip>:5000/bgwar/myapp:production
```
The deployment will initially fail because the image stream tag does not exist. Cancel the deployment 
to make sure.
```
oc deploy production --cancel
```

Edit the production deployment configuration and change the `imagePullPolicy` to `Always`.
```
oc edit dc/production
```
Example:
```
spec:
      containers:
      - image: <registry-service-ip>:5000/bgwar/myapp:production
        imagePullPolicy: Always
```

Expose the production dc and create a route for the service.

```
oc expose dc production --port=8080
oc expose svc production --name=production 
```
#### Deploy Blue app into production.

Promote the application to production by tagging the image stream.
```
oc tag myapp:latest myapp:production
```
```
Tag myapp:production set to bgwar/myapp@sha256:5ba60b060226456754ba2a37963209b4771c216a3da51a707fe919c620d999f8.

```
```
oc deploy production --latest --follow
```
Visit the application url and verify you see the blue rose.
```
$ curl --silent testing.$SUBDOMAIN | grep jpg
<img src="images/bluerose.jpg">
$
```

Older versions of OpenShift may need this user role for the image pull to suceed.
```
oc policy add-role-to-user edit system:serviceaccount:bgwar:default
```

#### Deploy the Green app into production.

First, build a green version of the app and deploy it into testing.

```
cp wars/green.war source/deployments/ROOT.war 
oc start-build myapp --from-dir=source --follow --wait
```
Wait for the build to suceed.

Deploy green app into test and wait for Catalina to start.
```
oc deploy testing --latest --follow
```

Visit the testing route and verify the green app is running.
```
$ curl --silent testing.$SUBDOMAIN | grep jpg
<img src="images/greenrose.jpg">
$
```

Now re-tag the green app and deploy it to production.
```
oc tag bgwar/myapp:latest bgwar/myapp:production
oc deploy production --latest --follow
--> Success
```
Wait for Catalina Server to finish starting.

```

Visit the production route to verify green app has been deployed.
```
oc get route
```

```
$ curl --silent testing.$SUBDOMAIN | grep jpg
<img src="images/greenrose.jpg">
$
```
