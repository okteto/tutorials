# How to Develop on Kubernetes with Okteto

### Introduction

Microservice-based applications make our applications more scalable. But they also make it harder to setup local development environments. You need to run several services that you are not familiar with, with a variety of different runtimes. And you probably need to switch your development environment between different projects. Not to mention that running all these services can eat up all of your computer's available resources.

In this blog post, you’ll create a Kubernetes cluster in DigitalOcean and use it to develop a Hello World application without having to install anything locally. Yes, you won't need to install compilers, runtimes, Docker or Kubernetes locally 😍. You will use Okteto to instantly update your application as fast as you can type code. Remote Kubernetes development provides a replicable environment that’s as close to production as possible. And all this without having to give up a fast inner loop that is crucial for developer productivity.

Digital Ocean offering is ideal to develop your Kubernetes applications directly in the cloud instead of dealing with local Kubernetes distributions which consume a lot of resources and are harder to maintain. DigitalOcean Kubernetes offering is cheaper and more oriented to developers than any other Kubernetes distribution, making it ideal as your development platform.

## Preerequisites

Before you begin this tutorial you'll need the following:

1. A Kubernetes cluster running in Digital Ocean. Follow [this guide](https://www.digitalocean.com/docs/kubernetes/quickstart/) to know more.
1. `kubectl` installed and configured to communicate with your cluster, as explained [here](https://www.digitalocean.com/docs/kubernetes/how-to/connect-to-cluster/).
1. Some familiarity with Kubernetes.

## Step 1: Create the Hello World Application

In this step you will create a Hello World application and its manifests to be deployed on Kubernetes.

First, create a fresh folder for your application by executing:

```console
$ cd ~
$ mkdir hello_world
$ cd hello_world
```

Edit a new file with your favourite IDE, for example with *nano*:

```console
$ nano main.go
```

Add the following content:

```golang
package main

import (
	"fmt"
	"net/http"
)

func main() {
	fmt.Println("Starting hello-world server...")
	http.HandleFunc("/", helloServer)
	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}

func helloServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello world!")
}
```

`main.go` is a Golang web server that returns the message *Hello world!*.

Create a fresh folder for the Kubernetes manifests by executing:

```console
$ mkdir k8s
```

You can run an application on Kubernetes by creating a Kubernetes Deployment Manifest.
Edit a new file with your favourite IDE, for example with *nano*:

```console
$ nano k8s/deployment.yaml
```

This content describes a Deployment that runs the `okteto/hello-world:latest` Docker image:

> `okteto/hello-world:latest` has been built from the previous `main.go` code and [this Dockerfile](https://github.com/okteto/go-getting-started/blob/master/Dockerfile)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  selector:
    matchLabels:
      app: hello-world
  replicas: 1
  template:
    metadata:
      labels:
        app: hello-world
    spec:
      containers:
      - name: hello-world
        image: okteto/hello-world:latest
        ports:
        - containerPort: 8080
```

You can expose an application on Kubernetes by creating a Kubernetes Service Manifest.
Edit a new file with your favourite IDE, for example with *nano*:

```console
$ nano k8s/service.yaml
```

This content describes a Service that exposes the `hello-world` Deployment using a DigitalOcean load balancer:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: hello-world
spec:
  type: LoadBalancer
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
      name: http
  selector:
    app: hello-world
```

You now have everything ready to deploy the Hello World applicaation to Kubernetes.

## Step 2: Deploy the Hello World Application

In this step you will deploy the Hello World application on Kubernetes and validate that it is working properly.

Deploy your application to Kubernetes by running:

```console
$ kubectl apply -f k8s
deployment.apps "hello-world" deleted
service "hello-world" deleted
```

After about one minute, you will be able to get the IP of you application from the `EXTERNAL-IP` column:

```console
$ kubectl get service hello-world
NAME          TYPE        CLUSTER-IP   EXTERNAL-IP      PORT(S)    AGE
hello-world   ClusterIP   10.0.9.181   134.209.139.198  8080/TCP   37s
```

Open your browser and go to the IP of the Hello World Application (`134.209.139.198` in my case). Cool, your application is running and available to the world!

## Step 3: Install Okteto CLI

The [Okteto CLI](https://github.com/okteto/okteto) is an open-source project that lets you synchronize application code changes to an application running in Kubernetes. You can continue using your favourite IDE, debuggers, compilers, ... no need to commit, build, push or redeploy containers to test your application.

Install the Okteto CLI in MacOS or Linux by running:

```console
$ curl https://get.okteto.com -sSfL | sh
```

Or if you are in Windows, download the file https://downloads.okteto.com/cli/okteto.exe and add it to your `$PATH`.

Once the Okteto CLI is installed, you are ready to put the Hello World application in development mode.

## Step 4: Put the Hello World Application in Development Mode

Okteto swaps applications running in Kubernetes by a development environment. To do that, Okteto uses the information provided by the [Okteto Manifest](https://okteto.com/docs/reference/manifest/index.html). This file declares the Kubernetes Deployment to be swapped by the development environment, the docker image used as your development runtime, the working directory where local changes are synchronized, etc. 

Edit a new file with your favourite IDE, for example with *nano*:

```console
$ nano okteto.yaml
```

This content describes how to put the Hello World application on development mode:

```yaml
name: hello-world
image: okteto/golang:1
command: ["bash"]
```

Run the following command:

```console
$ okteto up
```

```console
 ✓  Development environment activated
 ✓  Files synchronized
    Namespace: default
    Name:      hello-world

Welcome to your development environment. Happy coding!
okteto>
```

The `okteto up` swaps the Hello World application by a development environment, which means:

- The Hello World application container is updated with the docker image `okteto/golang:1`. This image contains the required dev tools to build, test, debug and run the Hello World application.
- A [file synchronization service](https://okteto.com/docs/reference/file-synchronization/index.html) is created to keep your changes up-to-date between your local filesystem and your application pods.
- A remote shell is started in your development environment. Build, test and run your application as if you were in your local machine.
- Whatever process you run in the remote shell will get the same incomming traffic, the same environment variables, volumes or secrets than the original Hello Work application pods, giving you a production-like development environment 💥. 

To run the application, execute in the remote shell:

```console
okteto> go run main.go
```

```console
Starting hello-world server...
```

The first time you run the application, Go will download your dependencies and compile your application. Wait for this process to finish and test your application by opening your browser and refreshing the page of your application.

Now we are ready to start doing chages to the Hello World application.

## Step 5: Develop directly on Kubernetes

Edit the `main.go` file with your favourite IDE, for example with *nano*:

```console
$ nano main.go
```

and modify the response message to be *Hello world from DigitalOcean!*:

```golang
package main

import (
	"fmt"
	"net/http"
)

func main() {
	fmt.Println("Starting hello-world server...")
	http.HandleFunc("/", helloServer)
	if err := http.ListenAndServe(":8080", nil); err != nil {
		panic(err)
	}
}

func helloServer(w http.ResponseWriter, r *http.Request) {
	fmt.Fprint(w, "Hello world from DigitalOcean!")
}
```

Instead of building images and redeploying containers to update the Hello World application, Okteto will synchronize your changes to your development environment on Kubernetes. Cancel the execution of `go run main.go` from the remote shell by pressing `ctrl + c` and rerun your application:

```console
okteto> go run main.go
```

```console
Starting hello-world server...
```

Go back to the browser and reload the page of the Hello World Application.

Cool! Your code changes were instantly applied to Kubernetes. No commit, build or push required 😎!

## Conclusions

Kubernetes has the potential to be a great development platform, providing replicable, resource-efficient and production-like development environments. We have shown you how to use [Okteto](https://github.com/okteto/okteto) to iterate your code changes directly on Kubernetes as fast as you can type code. Head over to this [Github Repository](https://github.com/okteto/samples) to see how to use Okteto with different stacks and debuggers.

If you share a Kubernetes cluster with your team, we recommend to give each member access to a sandboxed Kubernetes namespace, configured automatically to be isolated from other developers working on the same cluster. This functionality is provided by the [Okteto](https://marketplace.digitalocean.com/apps/okteto-1) application in the [DigitalOcean Kubernetes marketplace](https://marketplace.digitalocean.com/category/kubernetes). The Okteto App trasnforms your Kubernetes cluster in a fully featured development platform with the click of a button 😎.

