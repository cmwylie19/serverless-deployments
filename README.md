# Blue/Green and Canary Deployments in Serverless
This lab is about deployments in OpenShift Serverless. During the lab we will do a blue/green and canary deployment of a NodeJS application. This lab has been tested using CodeReady Containers in OpenShift 4.5.  Instructions for local development with minikube can be found [here](https://github.com/cmwylie19/minikube-serverless/blob/master/README.md).


The app that we are going to deploy has one endpoint, /greet, which is accessible via a GET request. The greet endpoint returns “hello!” when the environmental variable LANGUAGE is set to “EN”, and “hola!” when LANGUAGE is set to “ES”.   

Below is a picture of the source code of the greet endpoint for reference.   
   
   ![Greet Endpoint](greet.png)

The application has been containerized and pushed to an image repository, the image is available [here](quay.io/cmwylie19/node-server).   

## Prereqs
I have installing `wget` listed as a prerequisite to the lab in order to simply the process of installing the `kn` cli tool later on. This part is _not mandatory_ but will simplify the installation process.   

### Installing `wget`   
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

## Configuring Serverless in OpenShift
We will start by installing Serverless in OpenShift via the OpenShift Serverless Operator. If you are using [minikube](https://gitlab.consulting.redhat.com/appdev-coe/cloud-native-appdev-enablement/serverless-enablement/introduction/-/blob/master/minikube.md) you can *skip to the section on installing the knative cli*.

### Install the OpenShift Serverless Operator and subscription
```
oc apply -f knative-operator.crc.yaml 
```

### Create a project knative-serving
```
oc new-project knative-serving   
```

### Install the Knative serving operator
```
oc apply -f knative-serving.yaml
``` 


## Installing the Knative cli
Knative has a command line tool called `kn` for managing and releasing serverless applications. Below outlines how to install the tool using `wget` and how to install the `wget`  network retriever. If you do not want to install wget you can just download the `kn` binary for your operating system at the end of this section, just make sure you put the binary in your path. 


#### Installing `kn` cli with `wget` for linux
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

## Deploying our NodeJS app image into Serverless
Now it is time to deploy the image of the node application from the container registry into our serverless environment. The first thing we are going to do is create a namespace where our NodeJS Serverless application will live.   

To create the `greeter-ns` you will issue the following command:    

```
oc create namespace greeter-ns
```

On a minikube vm you will use
```
kubectl create namespace greeter-ns
```

Now that we have created the namespace we are going to deploy the NodeJS application image as a Knative service or ksvc and give the pod running the application a window of 10 seconds to scale back down to 0 if no further requests are recieved.

We are going to use `kn service create`, specify the name of the service, the image from which to use to generate the service, the autoscale window of 10 seconds and the namespace of `greeter-ns`.

``` 
kn service create greeter-service --image quay.io/cmwylie19/node-server --autoscale-window 10s -n greeter-ns
```

You should receive output similar to that of the image below. The last line of the output contains a URL where the knative service can be accessed. 

![terminal output](ksvc.png)  

## Interacting with your Knative Application in the cluster

Now that we have deployed our application, lets take a look at how we can derive information from our knative service in the cluster. We are going to take a look at several commands that can prove useful when performing routine operations in a serverless environment.

First we are going to take a closer look at the greeter service that we deployed:   
```
kn service list greeter-service -n greeter-ns
```

![terminal output](kn-service-list.png)     
You will notice from the terminal output that we are given seven columns of key information:   
- name - name of the service
- URL - where to access the service
- latest - name of the latest revision
- age - quantity of time service has been alive 
- conditions - reason if there is any failure 
- status - status of the service
- reason - explanation of failure if failure exists   

The next command we are going to issue is `kn revision list -n greeter-ns`.
```
kn revision list -n greeter-ns
``` 
![terminal output](kn-revision-list.png)  

This command returns information related to revisions of the knative services, specifically:
- name of the revisions
- service where the revision belongs 
- the percentage of the total service traffic going to the revision
- tags for the revision
- generation of the revision
- age of the revision
- conditions related to the health of the service 
- ready status 
- reasons related to health conditions of the revision

Lastly we are going to look at `kn service describe greeter-service -n greeter-ns`
```
kn service describe greeter-service -n greeter-ns
```

![terminal output](kn-describe-service.png)   

This command will print information related to:
- the namespace where the service lives
- the age of the service
- the URL from which the service can be accessed
- Information regarding the revisions, including the tag name and amount of traffic
- Condition Information regarding the health of the service

## Interacting with our serverless application 

We are going to test our application by curling against the /greet endpoint of the knative service, we should expect to receive "Hello!".   

In the last section we found that `kn service list greeter-service` is one way to return the URL associated with the knative service specified in the command. For the sake of the lab, I have extracted the URL for you using a script that extracts the third row and the second column from the output of the `kn service list greeter-service` command, appends "/greet" and curls against the endpoint.  

```
curl $(kn service list greeter-service -n greeter-ns | awk 'FNR == 2 { print $2 }')/greet  
```

You should expect to receive "Hello!". If you didn't you should go back and review your steps to make sure you didn't fat finger any of the commands.   


In the next section we are going to talk about blue green deployments. We will update our knative service by setting the environmental variable LANGUAGE to "ES" and adjusting the traffic so that our update goes unnoticed by our users. 

## Blue/Green Deployment   
Now that we have our application released into production and we have gone over a few ways to derive more information from the `kn` cli tool we are going to transition a blue/green deployment. A blue green deployment is useful when you want to deploy a new version of the application but you cannot afford downtown. One common usecase for blue green deployments is when you have a new feature that has not been tested in a production style environment and you need the QA team to go fully HAM on it before unleashing the users on it.   

In the last section we said we were going to update our knative service by setting the environmental variable LANGUAGE to "ES". Anytime when you create or update a service knative creates a revision. We are going to tag the original revision as blue, and the current revision with the environmental variable LANGUAGE set to ES as green. Lastly, we are going to shift 100% of the traffic coming into this service to the blue revision, the reason being that we are imagining that this green version contains a feature that needs to be tested in a production environment before allowing users to access it.

I have used `awk` to automate this a script for you but I want to explain how awk works:
- FNR == 2 means I want to isolate the second row
- {print $3} means I want to isolate the 3rd column 

In the next script I will run the command below and 
```
kn service list greeter-service -n greeter-ns 
```
![terminal output](kn-service-list.png)  

So, if i run `kn service list greeter-service -n greeter-ns | awk 'FNR == 2 { print $3 }'` I will have the name of the latestRevision printed to output.

Like in the last section, I have automated a one line shell script to update the service, but before executing the script, we are going to walk through and describe what it does in detail.
- update greeter greeter-service -n greeter-ns (update greeter service in the namespace greeter-ns)
- set LANGUAGE environmental variable to ES
- --tag $(kn service list greeter-service -n greeter-ns | awk 'FNR == 2 { print $3 }')=blue (Tag the original revision as blue)
- --tag @latest=green (tag the latest revision as green)
- shift 100% of the traffic to the blue revision.

### Create a green deployment
```
kn service update greeter-service -n greeter-ns --env LANGUAGE=ES --tag $(kn service list greeter-service -n greeter-ns | awk 'FNR == 2 { print $3 }')=blue --tag @latest=green --traffic blue=100,green=0
```

![terminal output](green.png)  

Now lets take a look at our revision list after updating the service and creating a new revision.
```
kn revision list -n greeter-ns
```


![terminal output](revisions.png)  
We have the blue revision with 100% of the traffic and the green revision with none.

**Note**: We can access each revision individually by the URL of the service, prefixed by the revisionTagName and a dash.   

For Example, if my knative service URL is http://knative-URL.com, I could access the green revision at http://green-knative-URL.com.   
If my knative service URL is http://google.com, i could access the green revision at http://green-google.com.   
This is important as we will need to use this technique to access the green revision since we have shifted 0% of the traffic to it.

---
> The green version is accesible via the URL of the knative service prefixed by the revision name and a dash.
>>
> If the URL of the knative service is "http://greeter-service-greeter-ns.apps-crc.testing", the URL of the green version is "http://green-greeter-service-greeter-ns.apps-crc.testing".
---   

Now lets curl against the greet endpoint of the green version of the application. This time you should get "hola!". 
```
curl $(kn service list greeter-service -n greeter-ns | awk 'FNR == 2 { print $2 }' | sed 's/greeter-service/green-greeter-service/g')/greet
```

This concludes blue green deployments in this lab, in the next section we will build on to our greeter-service and perform a canary deployment.

## Canary Deployment
A canary deployment allows you to rollout a new feature to only a subset of your users as an initial test to make sure that nothing else in your system is broken. The way you do that is by allocating a smaller percentage of the traffic to access the new feature, and not letting everyone access it at once as a way to mitigate the damage and not expose your whole userbase to a potentially flawed feature.   

We are going to update the greeter-service again for the canary deployment, this time allocating 50% of the traffic going to the revision with LANGUAGE set to “EN” and 50% goes to the revision with the LANGUAGE set to “ES”.

Before running the command, we will take a look at our revisions.

We will update the greeter-app service to shift 50% of the traffic to the blue revision and 50% to the green revision.
```
kn service update greeter-service -n greeter-ns --traffic $(kn revision list -n greeter-ns | awk 'FNR == 2 {print $1}')=50,$(kn revision list -n greeter-ns | awk 'FNR == 3 {print $1}')=50
```

After updating the service lets take a look at our revisions.
```
kn revision list -n greeter-ns
```
![terminal output](updated-revisions.png)

We see that 50% of the traffic is going to blue and the other 50% is going to green, this allowing us to slowly rollout features that will not affect 100% of the userbase until we are sure that our new feature is without bugs.

### Test Our deployment
Now we test our deployment with a shell one-liner   
```
for x in $(seq 20); do curl $(kn service list -n greeter-ns | awk 'FNR == 2 { print $2 }')/greet; echo "\n"; done;
```


Now you have deployed an app in OpenShift Serverless, created a Blue/Green Deployment, and created a Canary Deployment.

### Clean up
On OpenShift
```
oc delete namespace greeter-ns
```

On minikube
```
kubectl delete ns greet-ns
```

