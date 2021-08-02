---
title: "Building a Kubernetes Proof of Concept"
date: 2018-10-12T16:00:00-04:00
draft: false
summary: "Do you want to introduce Kubernetes to your company? Start here! This article was originally posted to opensource.com"
tags: ["Kubernetes"]
---

**NOTE**: This post is a copy of what I wrote [for opensource.com](https://opensource.com/article/18/3/building-kubernetes-proof-concept). That was waaaaay back in March of this year, just now getting around to re-posting it. Going to try and be more regular with posts here.

## Why Use Kubernetes?

Unfortunately, "because it's cool" isn't a good enough reason to use a technology. That being said, Kubernetes is REALLY cool.

There are a ton of use-cases from hosting your own Function as a Service (FaaS) to a full-blown application (both the microservices and monolith flavors). Sometimes you just need a cronjob to run once a day and do a thing. Throw the script into a container and you've got yourself a perfect candidate for the k8s CronJob object.

The real question though: Will Kubernetes bring business value? As always, the answer is that "it depends". If your main application is already microservice-ish, you can make a good argument that some of the services could be broken off into containers managed by kubernetes and better utilize those precious CPU cycles. It gets a little tougher when you attempt shoving a monolith into a container - [but it is possible](https://opensource.com/article/18/2/how-kubernetes-became-solution-migrating-legacy-applications)! Another thing to consider is performance. There is a lot of complex networking involved with containerized services in k8s. Your application may suffer a bit of response time increase if you're used to it running all on one machine.

Ok, lets say you've decided it will fit into your use case. What now?

## Building the Proof of Concept

I'm not going to go over the details of deploying a cluster here, there are [plenty](https://github.com/kelseyhightower/kubernetes-the-hard-way) [of](https://kubernetes.io/docs/setup/independent/create-cluster-kubeadm/) [guides](https://kubernetes.io/docs/getting-started-guides/kops/) out there already. What we want to focus on is getting something up and running quickly to prove our case. I should also note that there are services available to provide a k8s cluster with minimal hassle: Google Cloud's [GKE](https://cloud.google.com/kubernetes-engine/), Microsoft Azure's [AKS](https://docs.microsoft.com/en-us/azure/aks/), and Red Hat's [Openshift](https://www.openshift.com/). As of this writing, Amazon's service - [EKS](https://aws.amazon.com/eks/) - is not available to most folks, but it might be the best option in the future if you or your company is heavily invested in AWS. (EDIT: This post was originally written in March of 2018, EKS is now generally available.)

If none of those options are feasible for your PoC, you can accomplish a lot with [minikube](https://github.com/kubernetes/minikube) and a laptop.

## What to include in your PoC

So you've got a cluster. What sort of things should you start showing off? Ideally, you'd be able to operationalize a microservice or small app that your team manages. If time is a limiting factor, it's still possible to give a great presentation of an example application being deployed, scaled, and upgraded. In my opinion, the following is a good list of features to display in your PoC:

1. A [Deployment](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) with replicas
2. A [Service](https://kubernetes.io/docs/concepts/services-networking/service/) that exposes a set of pods internally to the cluster
3. An [ExternalName Service](https://kubernetes.io/docs/concepts/services-networking/service/#services-without-selectors) that creates an internal endpoint for a service outside of the cluster
4. [Scaling](https://kubernetes.io/docs/getting-started-guides/ubuntu/scaling/) those deployments up and down
5. [Upgrade](https://kubernetes.io/docs/tutorials/kubernetes-basics/update-intro/) a deployment with a new container image tag

Bonus points if any or all of that can be automated with a CI/CD pipeline that builds and deploys containers with few manual steps.

Lets look at some config files that will help you accomplish this.

## Example PoC

For all of these examples, I'll be using the [official Nginx container image](https://hub.docker.com/_/nginx/). It would be sufficient to use this in your PoC to demonstrate the functionality of kubernetes. Of course, if you have a containerized service already within your company, use that!

Also, quick note: I'm assuming you have [installed kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/) and [configured it](https://kubernetes.io/docs/tasks/tools/install-kubectl/#configure-kubectl) to communicate with your new cluster or minikube install. Minikube will actually set up your kube config and context for you.

I'll include the source code of all my example configs in [this repo](https://github.com/lucasreed/k8s-poc-configs). I have tested these on a minikube install.

### Deployment

We'll start off with the yaml and then we'll dissect the various parts of it. The `FILENAME` indicator in my code snippets indicate the filename in the repository.

```yaml
# FILENAME: k8s-configs/nginx-deploy-v1.12.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: poc-nginx-deploy
spec:
  selector:
    matchLabels:
      app: poc-nginx
  replicas: 2
  template:
    metadata:
      labels:
        app: poc-nginx
        version: "1.12-alpine"
    spec:
      containers:
      - name: poc-nginx
        imagePullPolicy: Always
        image: nginx:1.12-alpine
        ports:
        - name: http
          containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 3
```

A Deployment object is an abstraction level above Pods (and [ReplicaSets](https://kubernetes.io/docs/concepts/workloads/controllers/replicaset/)). Pods are the objects that run a set of one or more containers. You can read more about them [here](https://kubernetes.io/docs/concepts/workloads/pods/pod/).

Lets talk about the metadata. Specifically labels. Label keys are arbitrary and you can set them to whatever you want. For instance you could have objects with labels for the application version number, application name, or application tier. Here we just give `app` - for the name of our app - and `version` - where we'll track the current deployed version of the app. Labels allow various parts of kubernetes to find out about other parts by matching against their labels. For instance, if we have some other Pods already running that are labeled `app: poc-nginx`, when we apply this Deployment for the first time, the `spec.selector.matchLabels` section tells the Deployment to bring any pods with those labels under control of the Deployment object.

The `spec.template.spec` section is where we define the Pod definition that this deployment should manage. `containers` could have more than one container defined for the pod, but most of the time Pods only control one container. If there are multiple, you are saying that those two containers ALWAYS should be deployed together on the same node. If one of the containers fails though, the whole Pod will be relaunched meaning the container that was still healthy will be relaunched along with the unhealthy one. A full list of the Pod spec variables available can be found [here](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.9/#podspec-v1-core).

One last note on the Deployment config: in the container ports section above, I give port 80 the `name` of http. This is also arbitrary and optional. You can name the ports anything you want, or exclude the `name` config altogether. Giving a name to a port allows you to utilize the name instead of the port number in other configs, such as Ingresses. This is powerful because now you can change the port number your Pod container listens on by changing one config line instead of every other config line that references it.

We would add this Deployment to the kubernetes cluster by running:

```bash
kubectl create -f k8s-configs/nginx-deploy-v1.12.yaml
```

### Service

The Service config for our nginx Deployment would look something like this:

```yaml
# FILENAME: k8s-configs/nginx-svc.yaml
apiVersion: v1
kind: Service
metadata:
  name: poc-nginx-svc
  labels:
    app: poc-nginx
spec:
  type: NodePort
  ports:
  - port: 80
    protocol: TCP
    targetPort: http
  selector:
    app: poc-nginx
```

As a quick rundown, a service object just sets up a single endpoint for inter-cluster communication to a set of Pods. With the type set to NodePort though, it also allows access to the service from outside the cluster if your worker machines are available to your company network. NodePort chooses a random high level port number between 30000-32767 (unless you specify the port it should use) and then every machine in the cluster will map that port to forward traffic your Pods.

Notice in the `spec.selector` section above, we are using the label of our Pods (created by our Deployment) to tell the service object where traffic should be sent.

We can add this Service by running:

```bash
kubectl create -f k8s-configs/nginx-svc.yaml
```

### External Service

This portion of your PoC is optional and I haven't created a config for this in my example repo. But, lets say you have a database cluster in Amazon RDS that you want multiple apps to interact with. To make this easier you can create a Service object of the Type `ExternalName`. All this does is create a CNAME record in your kube dns that points the svc endpoint to whatever address you give it. The CNAME can be hit from any namespace with `<service_name>.<namespace>.svc.<cluster_name>` Here's an example config:

```yaml
kind: Service
apiVersion: v1
metadata:
  name: poc-rds-db
  namespace: default
  labels:
    app: poc-db
spec:
  type: ExternalName
  externalName: my-db-0.abcdefghijkl.us-east-1.rds.amazonaws.com
```

Now when something inside the cluster looks for `poc-rds-db.default.svc.minikube` over DNS (assuming a minikube cluster here), it will find a CNAME pointing to `my-db-0.abcdefghijkl.us-east-1.rds.amazonaws.com`.

### Accessing our Nginx Deployment

Now we have a deployment up with a service allowing us to talk to it at least within the cluster. If you're using minikube you can reach your nginx service like so:

```bash
# Take note of the IP address from this command
minikube status
minikube: Running
cluster: Running
kubectl: Correctly Configured: pointing to minikube-vm at 192.168.99.100

# Take note of the port number from this command
kubectl get svc/poc-nginx-svc
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
poc-nginx-svc   NodePort   10.103.171.66   <none>        80:32761/TCP   5s
```

Using the above examples, we could use our browser to hit http://192.168.99.100:32761 to see our service which is just the nginx welcome screen at this point.

## Scaling and Upgrading

The exciting topics of scaling and upgrading are going to be the bread and butter of your PoC. It's so easy to do that they may even seem anticlimactic. Here is how I would scale up our deployment in a pinch:

```bash
# This will take us from 2 replicas to 5
kubectl scale deploy/poc-nginx-deploy --replicas=5
```

Yeah. That's it. I did say this is how I would do it ***in a pinch***. This is how you can temporarily update a deployment to have more capacity, but if you are permanently changing the number of replicas, you should update the yaml files and run `kubectl apply -f path/to/config.yml` and, of course, keep all your yaml configs in source control.

Now with upgrading, the default upgrade strategy is a rolling-update. This will ensure no downtime in your application as it brings up a Pod with the new version (or whatever configuration that was changed) before any containers are taken offline.

Lets make a quick adjustment to our deployment in a new yaml file to bump up the image version to 1.13 instead of 1.12. I also keep our replicas up at 5 for this version instead of 2:

```yaml
# FILENAME: k8s-configs/nginx-deploy-v1.13.yaml
apiVersion: apps/v1beta2
kind: Deployment
metadata:
  name: poc-nginx-deploy
spec:
  selector:
    matchLabels:
      app: poc-nginx
  replicas: 5
  template:
    metadata:
      labels:
        app: poc-nginx
        version: "1.13-alpine"
    spec:
      containers:
      - name: poc-nginx
        imagePullPolicy: Always
        image: nginx:1.13-alpine
        ports:
        - name: http
          containerPort: 80
        livenessProbe:
          httpGet:
            path: /
            port: http
          initialDelaySeconds: 5
          periodSeconds: 3
```

Before we upgrade, open another terminal window and keep an eye on your pods:

```bash
watch kubectl get po --show-labels -l app=poc-nginx
```

You'll notice I'm making use of labels by passing the `-l` flag to limit the output to any pod with the label `app` as `poc-nginx`
Your output should look similar to this:

```bash
poc-nginx-deploy-75c8f68dd6-86js8   1/1       Running   0          7m        app=poc-nginx,pod-template-hash=3174924882,version=1.12-alpine
poc-nginx-deploy-75c8f68dd6-pvh2p   1/1       Running   0          7m        app=poc-nginx,pod-template-hash=3174924882,version=1.12-alpine
poc-nginx-deploy-75c8f68dd6-sfkvl   1/1       Running   0          15m       app=poc-nginx,pod-template-hash=3174924882,version=1.12-alpine
poc-nginx-deploy-75c8f68dd6-stcqk   1/1       Running   0          7m        app=poc-nginx,pod-template-hash=3174924882,version=1.12-alpine
poc-nginx-deploy-75c8f68dd6-z6bgz   1/1       Running   0          15m       app=poc-nginx,pod-template-hash=3174924882,version=1.12-alpine
```

Now we just run the following in another terminal window and watch the magic happen:

```bash
kubectl apply -f k8s-configs/nginx-deploy-v1.13.yaml
```

## End Notes

This is a good start for your PoC, but we are just brushing the surface! I hope this article piques your interest and you dive in. If so, here are a couple suggested readings from the Kubernetes documentation:

* [Ingress](https://kubernetes.io/docs/concepts/services-networking/ingress/)
* [Rolling Update with maxSurge and maxUnavailable](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#rolling-update-deployment)
* [StatefulSet](https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/)
* [DaemonSet](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/)
* [CronJob](https://kubernetes.io/docs/concepts/workloads/controllers/cron-jobs/)
