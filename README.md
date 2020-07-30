# Blue/Green and Canary Deployments in Serverless
This lab is about deployments in OpenShift Serverless. During the lab we will do a blue/green and canary deployment of a NodeJS application. This lab has been tested in OpenShift 4.3 and locally with minikube.  Instructions for local development with minikube can be found [here](https://gitlab.consulting.redhat.com/appdev-coe/cloud-native-appdev-enablement/serverless-enablement/introduction/-/blob/master/minikube.md).


The app that we are going to deploy has one endpoint, /greet, which is accessible via a GET request. The greet endpoint returns “hello!” if the environmental variable LANGUAGE is set to “EN”, and “hola!” if LANGUAGE is set to “ES”.   

Below is a picture of the source code of the greet endpoint for reference.   
   
   ![Greet Endpoint](greet.png)

The application has been containerized and push to quay container registry, the image is available [here](quay.io/cmwylie19/node-server).   


## Installing Serverless on OpenShift
We will start the lab by installing Serverless in OpenShift via the OpenShift Serverless Operator. If you are using [minikube](https://gitlab.consulting.redhat.com/appdev-coe/cloud-native-appdev-enablement/serverless-enablement/introduction/-/blob/master/minikube.md) you can skip to the section on installing the knative cli.

### Install the serverless operator
```
$ oc apply -f operator.yaml   
```

### Deploy the serverless operator subscription
```
$ oc apply -f operator-subscription.yaml   
```

### Create a project knative-serving
```
$ oc new-project knative-serving   
```

### Install the Knative serving operator
```
oc apply -f knative-serving
```

## Installing the Knative cli
Knative has a command line tool called `kn` for managing and releasing serverless applications. Below outlines how to install the tool using `wget` and how to install `wget`  network retriever. If you do not want to install wget you can just download the `kn` binary for your operating system at the end of this section, just make sure you put the binary in your path. 

#### Installing `wget`   
_You may need the prefix sudo depeding on whether are you using linux or mac and the distro you are using._   
   
Install `wget` with `dnf`:
```
dnf install wget
```

Install `wget` with `yum`:
```
yum -y install wget
```

Install `wget` with `apt-get`:
```
apt-get install wget
```

Install `wget` with `brew`:
```
brew install wget
```

#### Installing `kn` cli with `wget` for linux

Install `kn` cli tool with `wget`:
```
wget https://github.com/knative/client/releases/download/v0.16.0/kn-linux-amd64 && mv kn-linux-amd64 kn && chmod 777 kn && sudo mv kn /usr/local/bin/
```

#### Installing `kn` cli with `wget` for mac
```
wget https://storage.googleapis.com/knative-nightly/client/latest/kn-darwin-amd64 && mv kn-linux-amd64 kn && chmod 777 kn && sudo mv kn /usr/local/bin/
```


#### Links for installing `kn` for various operating systems
- [mac](https://storage.googleapis.com/knative-nightly/client/latest/kn-darwin-amd64)
- [linux](https://github.com/knative/client/releases/download/v0.16.0/kn-linux-amd64) 
- [windows](https://storage.googleapis.com/knative-nightly/client/latest/kn-windows-amd64.exe)

In the next section we are going to deploy the image of the node application into the serverless environment.   

## Deploy our NodeJS app image into OpenShift Serverless
Now it is time to deploy the image of the node application from the container registry into our serverless environment. The first thing we are going to do is create a new project (or namespace) where our application will live.   

In OpenShift or Minishift you will use

```
$ oc new-project greeter-ns
```

On a minikube vm you will use
```
$ kubectl create namespace greeter-ns
```

Have created the greeter-ns project we are going to deploy the NodeJS application image as a Knative service or ksvc. We are going to give a window of 10 seconds for the pod to scale down to 0 if no further requests are recieved. We are going to pipe the output to a file so that we can easily refer to the service url later.

``` 
$ kn service create greeter-app --image quay.io/cmwylie19/node-server --autoscale-window 10s > greet.v1.yaml
```

Curl against the url in the last line of the terminal output   
![terminal output](ksvc.png)   
> curl $(tail -1 greet.v1.yaml)/greet  

That's it!

## Blue/Green Deployment 
Now that we have our application released into production we are going to do a blue/green deployment. To help demonstrate the different versions of the application we are going to change the LANGUAGE environmental variable in the container to ES to make the greet endpoint return “hola” in the new version instead of hello.


### Create a green deployment
```  
$ kn service create greeter-app-green --image quay.io/cmwylie19/node-server --env LANGUAGE=ES > greet.v2.yaml
```

![terminal output](green.png)  

### Curl against the green version
Now you are going to make a GET request to the new green version of the application. This time you should get "hola!". 

> curl $(tail -1 greet.v2.yaml)/greet

Now we have successfully deployed a green version of our OpenShift Serverless application!

## Canary Deployment
This time we are going to do a canary deployment of the original version of our knative service. Our goal is that 50% of the traffic goes to the container with LANGUAGE set to “EN” and 50% goes to the container with the LANGUAGE set to “ES”.

### Look at the revisions
```
kn revision list
```
The idea is that we edit the configuration of the knative service node-server to split 50% of the traffic to original revision(blue), and 50% to the latest revision(green).

```
kn service update greeter-app --traffic $(kn revision list | awk 'FNR == 2 {print $1}')=50,$(kn revision list | awk 'FNR == 3 {print $1}')=50
```

### Test Our deployment
Now we test our deployment with a shell one-liner   
```
$ for x in $(seq 20); do curl $(tail -1 greet.v1.yaml)/greet done
```


Now you have deployed an app in OpenShift Serverless, created a Blue/Green Deployment, and created a Canary Deployment.

### Clean up
> oc delete project node-server-project