# Flask App Container Deployment

This is a test deployment of a flask application to a K8 cluster using a Helm template for Kubernetes

What is included:

* Flask Micro Blog app, not separated into microservices for ease of testing
* A WHL file for app distribution
* Helm charts for package installation

## Getting Started

These instructions will provide usage and distribution examples

### Supported Platforms

Ubuntu 18.10

### Prerequisites

Software needed: 
* Docker
* Kubernetes
* Helm

### Background
The relationship between a package manager like apt-get and Ubuntu is essentially the same as the relationship between HELM and Kubernetes. Helm is a package manager for Kubernetes that allows you to define, install, and upgrade charted applications in addition to deploying them. 

In order to create charts for HELM templates, we need to understand how Docker and Kubernetes come together first. 

#### Docker
Docker is container deployment software that packages applications into containers. Rather than having each application run in its own virtual machine which requires the full OS, docker containers only include the application and its dependencies and share the OS with other containers.

As a result, containers are 

Isolated and Consistent
Lightweight
Portable

Figure 1: Container internals

 

More information on containers

#### Kubernetes
Kubernetes is container management software that automates the deployment, scaling, and management of containerized applications.

Figure 2: Kubernetes Layout



The architecture is centered around a cluster which consists of a Master and Nodes.

##### Master

the access point for admins and developers to manage scheduling and deployment of containers
stores state and configuration in the data store etcd

##### Nodes

Each node has an agent called a kubelet
Manages the state of the node: starting, stopping, and maintaining containers
Has a network proxy that also works as a load balancer (Kube-proxy)
Pods represent a running process on a cluster and can be created or deployed. 
A pod is an encapsulated application container that packages configurations and options that govern how the container should run
Represents one instance of the application container
More information on Kubernetes

#### HELM Templates
Charts are a packaging format for HELM which is a collection of files that describe Kubernetes resources. 

Layout:
```
app_name/
  Chart.yaml          # A YAML file containing information about the chart
  LICENSE             # OPTIONAL: A plain text file containing the license for the chart
  README.md           # OPTIONAL: A human-readable README file
  requirements.yaml   # OPTIONAL: A YAML file listing dependencies for the chart
  values.yaml         # The default configuration values for this chart
  charts/             # A directory containing any charts upon which this chart depends.
  templates/          # A directory of templates that, when combined with values,
                      # will generate valid Kubernetes manifest files.
  templates/NOTES.txt # OPTIONAL: A plain text file containing short usage notes
```

Deploying a Sample Application

These are some steps for deploying a sample flask application to a K8s Cluster

First, we need a test application to containerize. I used micro-blog, flaskr for test deployment

Flaskr Layout:

/home/user/Projects/flask-tutorial
```
├── flaskr/
│   ├── __init__.py
│   ├── db.py
│   ├── schema.sql
│   ├── auth.py
│   ├── blog.py
│   ├── templates/
│   │   ├── base.html
│   │   ├── auth/
│   │   │   ├── login.html
│   │   │   └── register.html
│   │   └── blog/
│   │       ├── create.html
│   │       ├── index.html
│   │       └── update.html
│   └── static/
│       └── style.css
├── tests/
│   ├── conftest.py
│   ├── data.sql
│   ├── test_factory.py
│   ├── test_db.py
│   ├── test_auth.py
│   └── test_blog.py
├── dist/
│   ├── DockerFile
│   ├── flaskr-1.0.0-py2-none-any.whl
│   └── micro-blog-k8.yaml
├── venv/
├── setup.py
└── MANIFEST.in
```

### Dockerizing your application
Applications can be containerized through a DockerFile

A base image is used to build your app on top of and there are specific ones that include:

Java
Python
Nginx
If you are building a docker image from scratch you would need to use a base OS image.

For my application, I used Ubuntu 18.10 and instead of copying in all the directories / extra files, I packaged my application into a .whl file (though this is a bit more overhead).

DockerFile:
```
# Use an official Python runtime as a parent image
FROM ubuntu:18.10
RUN apt-get update
RUN apt-get install -y python python-pip
# Set the working directory to /app
WORKDIR ./dist
# Copy the current directory contents into the container at /app
COPY ./ ./dist
RUN ls
# Install the app
RUN pip install ./dist/flaskr-1.0.0-py2-none-any.whl
# Make port 5000 available to the world outside this container
EXPOSE 5000
# Python3 is configured to use ASCII
ENV LC_ALL C.UTF-8
ENV LANG C.UTF-8
# set flask environment variable
ENV FLASK_APP flaskr
ENV FLASK_ENV development
# setup database
RUN flask init-db
# run the server
CMD ["flask", "run", "--host=0.0.0.0"]
``` 

To create your image, go to the directory with the DockerFile:
```
$ docker build -t micro-blog .
```
 
To see your image:
```
$ docker image ls
```

Run your container; The application has port 5000 exposed to localhost, so use the host as the access point. In my case, I'm running this on minikube so my endpoint is http://192.168.99.100:5000/
```
$ docker run -p 5000:5000 micro-blog
* Serving Flask app "flaskr" (lazy loading)
* Environment: development
* Debug mode: on
* Running on http://0.0.0.0:5000/ (Press CTRL+C to quit)
* Restarting with stat
* Debugger is active!
* Debugger PIN: 162-242-292
```

Uploading to a repository: (Use a private one!)
```
$ docker login

$ docker tag micro-blog <username>/micro-blog

$ docker push <username>/micro-blog
```

Now you can run it anywhere by first logging in and running
```
$ docker run p 5000:5000 <username>/micro-blog
```
 

Cleanup:
```
$ docker ls

CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
8f052e29b764 <username>/micro-blog "flask run --host=0.…" 18 seconds ago Up 17 seconds 5000/tcp friendly_cohen

$ docker stop friendly_cohen

$ docker image rm micro-blog

$ docker image rm <username>/micro-blog
```

### Deploying through Kubernetes
With a running K8 cluster, you can automate the deployment of containers using Deployments

Deployment configuration file:
```
apiVersion: apps/v1 # for versions before 1.9.0 use apps/v1beta2
kind: Deployment
metadata:
name: micro-blog-deployment
spec:
selector:
matchLabels:
app: micro-blog
template:
metadata:
labels:
app: micro-blog
spec:
containers:
- name: micro-blog
image: <Username>/micro-blog:latest
ports:
- containerPort: 5000
``` 

Run the following commands on the K8s cluster to create a deployment and expose it. 
```
$ kubectl create -f/micro-blog-k8.yaml

$ kubectl expose deployment micro-blog-deployment --type="NodePort" --port 5000

$ kubectl get services

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 11d
micro-blog-deployment NodePort 10.107.228.87 <none> 5000:30557/TCP 6m
```

The application is accessible at http://192.168.99.101:30557/

### Deploying through HELM Templates
To create a HELM chart, run
```
$ helm create flaskr-chart
```

Which will create the necessary file structure for you. Inside of the templates folder will contain some files generated for basic app deployment.

For Flaskr, all we will need to do is change the port numbers 
* inside of deployment.yaml to 5000 from 80.
* inside of values.yaml, change type to NodePort and port to 5000
service.yaml should look like this, (cut out a few parameters)
```
apiVersion: v1
kind: Service
metadata:
name: {{ include "flaskr-chart.fullname" . }}
labels:
app.kubernetes.io/name: {{ include "flaskr-chart.name" . }}
helm.sh/chart: {{ include "flaskr-chart.chart" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
app.kubernetes.io/managed-by: {{ .Release.Service }}
spec:
type: {{ .Values.service.type }}
ports:
- port: {{ .Values.service.port }}
targetPort: http
protocol: TCP
name: http
selector:
app.kubernetes.io/name: {{ include "flaskr-chart.name" . }}
app.kubernetes.io/instance: {{ .Release.Name }}
```

Next we need to install and deploy the HELM chart:
```
$ helm install ./flaskr-chart/

$ kubectl get services

NAME TYPE CLUSTER-IP EXTERNAL-IP PORT(S) AGE
kubernetes ClusterIP 10.96.0.1 <none> 443/TCP 11d
micro-blog-deployment NodePort 10.107.228.87 <none> 5000:30557/TCP 6m
```
The application is accessible at http://192.168.99.101:30557/

### Distributing Helm Charts
Now that we have a chart that is installable, we now have to package it up and upload it somewhere so that we can share it with other developers.

To package up flaskr, we run this command:
```
$ helm package flaskr-chart
```

This creates a tar file that can be installed on any K8 cluster

Helm repos use a HTTP server that contains an index.yaml file and all the corresponding chart files.
* For this example, I used github pages to service my files
To populate the index.yaml file, first upload files to your repository and setup your webserver. Then, run
```
$ helm repo add micro-blog https://vinhnguyen500.github.io/micro_blog/flask_tutorial/
```
You should be able to see your newly added repo by running repo list
```
$ helm repo list
NAME URL
stable https://kubernetes-charts.storage.googleapis.com
local http://127.0.0.1:8879/charts
micro-blog https://vinhnguyen500.github.io/micro_blog/flask_tutorial/
$ helm search micro-blog
NAME CHART VERSION APP VERSION DESCRIPTION
micro-blog/flaskr-chart 0.1.0 1.0 A Helm chart for Kubernetes
```
Now you can install this chart from any k8 cluster!
```
$ helm install micro-blog/flaskr-chart
```

## Built With

* [Docker](https://www.docker.com/) - Container creation
* [Kubernetes](https://kubernetes.io/) - Container management
* [Helm](https://helm.sh/) - Kubernetes Package Management

## Authors

* **Vinh Nguyen**


