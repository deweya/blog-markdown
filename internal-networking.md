# Troubleshooting OpenShift Internal Networking
There are many times in OpenShift where microservices need to talk to each other internally without exposing routes to the outside world. These microservices interact via the Kubernetes [service](https://kubernetes.io/docs/concepts/services-networking/service/) API, which acts as a load balancer that resolves a set of pods to a single IP.

While OpenShift and the service API work together to abstract most of the complexity behind networking, problems can still arise when trying to interact with other microservices deployed on OpenShift. In this post we’ll talk about some of the most common problems around internal networking and how to troubleshoot them. For the examples, we’ll assume a basic app consisting of a single ClusterIP Service and Deployment, each called **hello-world**.

The following service will be used as an example for each issue described in this post.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: my-namespace
spec:
  selector:
    app: hello-world
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
```

## How To Begin Troubleshooting
The best way to begin troubleshooting is to <mark>oc rsh into the pod that is making the call</mark> to your target service. For example, if I have a deployment called rhel that is attempting to talk to another deployment called hello-world, then I would want to access the rhel pod to begin troubleshooting my networking error. For example:

```bash
oc rsh rhel-2-fqjbw -n my-namespace
```

Once inside the pod, <mark>attempt running “curl -v”</mark> against the endpoint your pod is trying to reach. This will give verbose output and will often reveal the issue behind your networking error. The table below gives an overview of some sample curl -v output and some possible errors associated with it.

| curl -v Output | Possible Errors |
| -------------- | --------------- |
| sh-4.2$ curl -v -k https://hello-world:443<br/>Could not resolve host: hello-world; Unknown error<br/>Closing connection 0<br/>curl: (6) Could not resolve host: hello-world; Unknown error | 1) [Service does not exist](#service-does-not-exist)<br/>2) [Target hostname is incorrect](#target-hostname-is-incorrect) |
| sh-4.2$ curl -v -k https://hello-world.my-namespace:443<br/>About to connect() to hello-world.my-namespace port 443 (#0)<br/>Trying 172.30.209.206…<br/>Connection timed out<br/>Failed connect to hello-world.my-namespace:443; Connection timed out<br/>Closing connection 0<br/>curl: (7) Failed connect to hello-world.my-namespace:443; Connection timed out | 1) [Isolation policy is blocking traffic](#sdn-isolation-policy-is-blocking-traffic) |
| sh-4.2$ curl -v -k https://hello-world:443<br/>About to connect() to hello-world port 443 (#0)<br/>Trying 172.30.250.96…<br/>No route to host<br/>Failed connect to hello-world:443; No route to host<br/>Closing connection 0<br/>curl: (7) Failed connect to hello-world:443; No route to host | 1) [Service selector is incorrect](#service-selector-is-incorrect)<br/>2) [Service port is incorrect](#service-port-is-incorrect)<br/>3) [Service targetPort name is not specified on the deployment](#service-targetport-name-is-not-specified-on-the-deployment) |
| sh-4.2$ curl -v -k https://hello-world:443<br/>About to connect() to hello-world port 443 (#0)<br/>Trying 10.128.2.44…<br/>Connection refused<br/>Failed connect to hello-world:443; Connection refused<br/>Closing connection 0<br/>curl: (7) Failed connect to hello-world:443; Connection refused | 1) [Service clusterIP is none](#service-clusterip-is-none)<br/>2) [Service targetPort is incorrect](#service-targetport-is-incorrect)<br/>3) [Container does not expose targetPort](#container-does-not-expose-targetport) |

Note that this only covers some of the most common networking errors that I have observed and that your particular error may not be covered in this post.

Let’s begin looking at some of the most common errors around OpenShift networking.

## Service Does Not Exist
First things first, <mark>you need a service object</mark> to be able to route traffic to the desired app. You can quickly create a service with the “oc expose” command:

```bash
oc expose deployment hello-world # For Deployment objects
oc expose deploymentconfig hello-world # For Deployment Configs
```

Alternatively, you can use the hello-world service YAML at the beginning of this post as an example to help get started. Once you have written the YAML, create the service with:

```bash
oc apply -f $PATH_TO_SERVICE_YAML
```

## Target Hostname Is Incorrect
The most common networking issues are caused by attempting to reference an incorrect host name. The host name of an app will be determined by the name of its service. Depending on whether or not the source and target apps are in the same namespace, the target host name will be either <mark>\<service-name></mark> or <mark>\<service-name>.\<namespace-name></mark>.

### Source and Target Apps in the Same Namespace
If your source and target apps are in the same OpenShift namespace, then the target hostname will simply be the name of the target service. Using the hello-world service above as an example, any app trying to talk to the hello-world app would simply use the host name **hello-world**.

### Source and Target Apps in Different Namespaces
The target host name will be a little different if the source and target apps live in different namespaces. In this case the target host name will be \<service-name>.\<namespace-name>. Using the hello-world service above as an example, any app trying to talk to the hello-world app from a different namespace would use the host name **hello-world.my-namespace**.

## SDN Isolation Policy is Blocking Traffic
The OpenShift SDN supports [three different modes for networking](https://docs.openshift.com/container-platform/4.1/networking/openshift-sdn/about-openshift-sdn.html), with the default being network policy in OpenShift 4. It could be possible that your mode’s <mark>isolation policy has not been configured</mark> to allow traffic to reach your app.

If using *network policy* mode, ensure that a [NetworkPolicy](https://docs.openshift.com/container-platform/4.1/networking/configuring-networkpolicy.html) object has been created that allows traffic to reach your target app.

If using *multitenant* mode, ensure that your source and target apps’ namespaces have been [joined together](https://docs.openshift.com/container-platform/4.1/networking/openshift-sdn/multitenant-isolation.html#nw-multitenant-joining_multitenant-isolation) to allow network traffic.

Your pods should already be able to reach each other with *subnet* mode.

## Service Selector is Incorrect
The most common way to route traffic with a service is to use a label selector that matches a label on the app’s pods. In the example service above, the hello-world service will route traffic to pods with a label **app=hello-world**. Make sure that the target Deployment or DeploymentConfig sets a <mark>label on each pod that matches the service selector</mark> and vice versa.

Here’s part of an example Deployment that sets the “app=hello-world” label that the service selector expects on each pod. Notice the “template.metadata.labels.app” value, which sets the pod “app=hello-world” label.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 1
  selector:
    matchLabels:
      app: hello-world
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
        - name: hello-world
...
```

## Service clusterIP is None
Notice in the above hello-world service that the YAML specification lacks a **clusterIP** key-value pair. This means that OpenShift will automatically assign the hello-world service an IP address. Compare that to this modified service below, in which we set the clusterIP to “None”:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
  namespace: my-namespace
spec:
  selector:
    app: hello-world
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
  clusterIP: None
```

This is actually a [headless service](https://kubernetes.io/docs/concepts/services-networking/service/#headless-services), meaning that the service is not assigned an IP address. Headless services have many different use cases, but if you’re running a simple architecture with the intention of a Deployment or DeploymentConfig being load-balanced by a service, you may have accidentally created a headless service. Check that the service has a clusterIP allocated with “oc get svc hello-world -o yaml”. If you can verify that the clusterIP is None, <mark>delete the service and apply it again without a clusterIP spec</mark>.

## Service Ports are Incorrect or are Not Exposed
Part of a service’s job is to identify the **port** and **targetPorts** of an application. The service will accept traffic to port “port” and will redirect to port “targetPort” on the running container. Another common issue around internal traffic is that these <mark>port values can be either incorrect or unexposed by the container</mark>.

### Service Port is Incorrect
The hello-world service above specifies port 443 and targetPort 8443:

```yaml
apiVersion: v1
kind: Service
...
  ports:
    - name: https
      protocol: TCP
      port: 443
      targetPort: 8443
...
```

This port, port 443, reroutes to port 8443 on the target container. Make sure that your requests are for port 443. Otherwise, the service will not be able to route your request to the targetPort of the container.

### Service targetPort is Incorrect
Your service targetPort may be incorrect if your request is hitting the service port and is still failing. Make sure your service’s targetPort is specifying a port that is exposed by the container.

### Service targetPort Name is not Specified on the Deployment
Imagine the hello-world service above exposed ports like this instead:

```yaml
apiVersion: v1
kind: Service
...
  ports:
...
    - name: https
      protocol: TCP
      port: 443
      targetPort: https
```

This port, port 443, is targeting a port called “https”. This targetPort name is referring to a port exposed on the Deployment or DeploymentConfig. Either resource will be expected to have a port specified to route to “targetPort: https”.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
...
      containers:
...
          ports:
            - containerPort: 8443
              protocol: TCP
              name: https
```

Notice at the bottom of the deployment the “ports:” stanza. Since the service is referring to the “https” targetPort, the deployment must also have a corresponding “https” port to route traffic to the desired port. In this case “https” endpoint will accept traffic at port 443 and reroute to port 8443, which is specified by the **containerPort** of the Deployment.

If your service uses a name instead of a number for its targetPort, <mark>make sure that name is also specified on the Deployment object</mark>.

### Container does not Expose Targetport
This is one that is often overlooked. Sometimes the issue is not an OpenShift problem but instead has to do with the running container. If you know that the port and targetPort are specified and configured properly, then <mark>the running container may not be exposing the targetPort</mark>.

Take the following Spring Boot **application.properties**, for example:

```
server.port=8444
...
```

Given the hello-world service above, you can expect this application.properties config to be the root cause of your networking issue. The targetPort is set to 8443, but the container is actually exposing port 8444.

We can fix this by modifying the application.properties to instead read as:

```
server.port=8443
...
```

Although this example was in Spring Boot, a similar troubleshooting approach can be taken with any runtime or application source. Make sure that your app is configured properly to expose the container specified by your service.

## Thanks for Reading!

Hopefully this was able to help you troubleshoot any networking issues you’re experiencing in OpenShift. Although this did not cover every networking issue you could possibly experience, I think it covers the most common (and arguably the most frustrating) errors.

For more information on OpenShift networking and the Kubernetes service API, check out the following links. Until next time!

- https://docs.openshift.com/container-platform/4.1/networking/understanding-networking.html (OpenShift 4.1)
- https://docs.openshift.com/container-platform/3.11/architecture/networking/networking.html (OpenShift 3.11)
- https://kubernetes.io/docs/concepts/services-networking/service/
