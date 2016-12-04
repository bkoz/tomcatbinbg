# Blue/Green binary deployment to Tomcat on OpenShift v3.3

## (Work in progress) Deploy from testing to production.

### Configuration

#### Set OPENSHIFT_SERVER to match your OpenShift API server.

```
OPENSHIFT_SERVER=masteroselab-bkocp33-utzuqe0l.srv.ravcloud.com:8443
```
Clone this git repo locally.

```
git clone https://github.com/bkoz/tomcatbinbg.git
cd tomcatbinbg
cp wars/blue.war source/deployments/ROOT.war
```
Login to OpenShift, create a project, a new build and start the build.

```
oc login $OPENSHIFT_SERVER
oc new-project bgwar
```

#### Create the test enviroment.
```
oc new-build --image-stream=jboss-webserver30-tomcat7-openshift --name=myapp --binary=true
oc start-build myapp --from-dir=source
```

Wait for the build and registry push to suceed.

`oc logs bc/myapp --follow`

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
Get the hostname of the route and visit your application using a web browser.

```
oc get route
```
```
NAME      HOST/PORT                      PATH      SERVICES   PORT      TERMINATION
testing   testing.ose-apps.haveopen.com            testing    8080      
```

#### Create the production environment.

Create the production deployment config. This will trigger off of the `production` image stream tag. 

```
oc create deploymentconfig production --image=<registry-service-ip>:5000/bgwar/myapp:production
```
The deployment will initially fail because the image stream tag does not exist. Cancel the deployment 
to make sure.
```
oc deploy production --cancel
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
oc deploy production --latest
```
Visit the application url and verify you see the blue rose.


Earlier versions of OpenShift may need this user role for the image pull to suceed.
```
oc policy add-role-to-user edit system:serviceaccount:bgwar:default
```

#### Deploy the Green app into production.

First, build a green version of the app and deploy it into testing.

```
cp wars/green.war source/deployments/ROOT.war 
oc start-build myapp --from-dir=source
oc deploy testing --latest
```

Now re-tag the green app and deploy it to production.
```
oc tag bgwar/myapp:latest bgwar/myapp:production
oc deploy production --latest
oc logs dc/production -f
--> Success
```
Wait for Catalina Server to finish starting.

```

Visit the production route to verify green app has been deployed.
```
oc get route
```

