# Workshop : Introduction to Helm 3

This workshop is done on a Debian based linux system (linux Mint). The helm version used is v3.0.2. Kubernetes v1.15.3 is running using minikube.  
Be sure to have a Kubernetes cluster up and running before starting this workshop.

## Resources

* Official Website : [https://helm.sh/](https://helm.sh/)
* Github : [https://github.com/helm/helm](https://github.com/helm/helm)

## Installation

To install Helm, follow the instruction on Github :

1. Download the latest version from the [Github release page](https://github.com/helm/helm/releases/)
1. Once downloaded, extract the binary and add it to the path
1. Test that the binary is working using the following command :  
```bash
$ helm version
version.BuildInfo{Version:"v3.0.2", GitCommit:"19e47ee3283ae98139d98460de796c1be1e3975f", GitTreeState:"clean", GoVersion:"go1.13.5"}
```
It should print the client version (as showed above).

4. (optional) Enable autocompletion, add the following line in your `.bashrc` file (or any other file loaded at startup). 
```bash
# add in your ~/.bashrc file
source <(helm completion bash)
# source the file, or start a new terminal session
$ source ~/.bashrc
```


## Basic concept

Helm is a package manager for Kubernetes. Meaning it allows you to quickly install and configure resources in your Kubernetes cluster.  
There are 3 big concept in Helm :
* **Chart** : it's a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster. Think of it like the Kubernetes equivalent of a Homebrew formula, an Apt dpkg, or a Yum RPM file.
* **Repository** : it's the place where charts can be collected and shared. It’s like Perl’s CPAN archive or the Fedora Package Database, but for Kubernetes packages.
* **Release** is an instance of a chart running in a Kubernetes cluster. One chart can often be installed many times into the same cluster. And each time it is installed, a new release is created. Consider a MySQL chart. If you want two databases running in your cluster, you can install that chart twice. Each one will have its own release, which will in turn have its own release name.

> Helm installs **charts** into Kubernetes, creating a new **release** for each installation. And to find new charts, you can search Helm chart **repositories**.

## Deploy a chart

### Add a repository

In order to install an existing chart, you need to add the repository first (just like you'll do with an apt repository for example). 

Let's add the official Helm chart repository.
```bash
$ helm repo add stable https://kubernetes-charts.storage.googleapis.com/
"stable" has been added to your repositories
```
Once the repo added, you can browse it :
```bash
$ $ helm search repo stable
NAME                                 	CHART VERSION	APP VERSION            	DESCRIPTION                                       
stable/acs-engine-autoscaler         	2.2.2        	2.1.1                  	DEPRECATED Scales worker nodes within agent pools 
stable/aerospike                     	0.3.2        	v4.5.0.5               	A Helm chart for Aerospike in Kubernetes          
stable/airflow                       	5.2.4        	1.10.4                 	Airflow is a platform to programmatically autho...
stable/ambassador                    	5.3.0        	0.86.1                 	A Helm chart for Datawire Ambassador              
stable/anchore-engine                	1.4.0        	0.6.0                  	Anchore container analysis and policy evaluatio...
...
```

### Install a chart

Now that everything is set, we can install a chart. This is done with the `helm install` command.

Let's try and install **MySQL**.
```bash
# Just like with apt when you update the repo, you have to update the helm repo first
$ helm repo update
Hang tight while we grab the latest from your chart repositories...
...Successfully got an update from the "stable" chart repository
Update Complete. ⎈ Happy Helming!⎈ 
# Now let's install mysql. The resource will be named mydb
$ helm install mydb stable/mysql
NAME: mydb
LAST DEPLOYED: Mon Jan 20 13:24:26 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
MySQL can be accessed via port 3306 on the following DNS name from within your cluster:
mydb-mysql.default.svc.cluster.local
...
```

Et voilà ! You have a **MySQL** database deployed, up and running in your Kubernetes cluster.

What does that mean in your Kubernetes cluster ? It means that Helm has deployed the following list of resources :
* A [**deployment**](https://kubernetes.io/docs/concepts/workloads/controllers/deployment/) based on the MySQL image with one replicas.
* A [**service**](https://kubernetes.io/docs/concepts/services-networking/service/) of type `ClusterIP` targetting your deployment.

You can see the newly created resources using a `kubectl`command :
```bash
$ kubectl get deploy,svc -l app=mydb-mysql
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/mydb-mysql   1/1     1            1           12m

NAME                 TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
service/mydb-mysql   ClusterIP   10.106.123.97   <none>        3306/TCP   12m

```
If you want to see all the resources installed with Helm, you can use
```bash
$ helm list
NAME	NAMESPACE	REVISION	UPDATED                                	STATUS  	CHART      	APP VERSION
mydb	default  	1       	2020-01-20 13:24:26.942633983 +0100 CET	deployed	mysql-1.6.2	5.7.28
```
Here we should be able to find our newly deployed MySQL.

### Under the hood

As you can guess, everything didn't just magically pop out from the Internet. In fact, Helm store everything in your local environment. Below are all the directory Helm uses (*in a Linux environment*) :
* `$HOME/.config/helm/` : that's the config. 
* `$HOME/.cache/helm/` : that's the cache. Here you'll find the repo index, the templates used, ...
* `$HOME/.local/share/helm/` : that's the data.

To see how our MySQL chart works and to see what's configured, we can have a look inside the `$HOME/.cache/helm/repository` directory. Here you should find a **tgz** archive containing the MySQL chart source code *mysql-1.6.2.tgz*.

Let's have a look at the content. This way we'll understand the basique structure of a Helm chart.
```bash
$ tree -L 2 mysql
mysql
├── Chart.yaml
├── README.md
├── templates
│   ├── configurationFiles-configmap.yaml
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── initializationFiles-configmap.yaml
│   ├── NOTES.txt
│   ├── pvc.yaml
│   ├── secrets.yaml
│   ├── serviceaccount.yaml
│   ├── servicemonitor.yaml
│   ├── svc.yaml
│   └── tests
└── values.yaml

2 directories, 13 files
```
Let's list some files and describe their purpose :
* `Chart.yaml` : This file contains all the metadata. Here you'll find some info about the authors, the sources, some description, ...
* `templates/` : This directory contains all the Kubernetes resources files. Here you'll find the descriptor for all resources used by this chart. You can see the `kind: Deployment` resource file (`deployment.yaml`) as well as the `kind: Service` resource file (`svc.yaml`) along with other resources used. The resources files are all configurable. If you look at one of them, you'll see the variable block `{{ .Values.imagePullPolicy }}`. As you can see, there are some variables starting with `.Template`, `.Chart` or `.Release`. Those are values extracted from built-in object in Helm. You can find more about them [in the official documentation](https://helm.sh/docs/chart_template_guide/builtin_objects/).
* `values.yaml` : This file contains all the values used by the template. This is the file you want to modify to customize your release.

Now that we have a basic understanding of how Helm works, let's try and build our own chart !

## Building a chart

Here we will create a simple chart based on the [whoami image](https://github.com/containous/whoami) (from [Containous](https://containo.us/)).

### Create the chart files

To deploy a chart, we need all the source code with the correct directory structure. Lucky us, Helm comes with a scaffolding command which does everything for us. Let's use it and create our chart :
```bash
$ helm create whoami
Creating whoami
$ tree whoami/
whoami/
├── charts
├── Chart.yaml
├── templates
│   ├── deployment.yaml
│   ├── _helpers.tpl
│   ├── ingress.yaml
│   ├── NOTES.txt
│   ├── serviceaccount.yaml
│   ├── service.yaml
│   └── tests
│       └── test-connection.yaml
└── values.yaml

3 directories, 9 files
```
Let's configure our template.  

### Remove unnecessary resources
As you can see, by default, Helm create a Kubernetes descriptor for each of those resources : `Deployment`, `Service`, `Ingress` and `ServiceAccount`. We won't be using the `ServiceAccount` nor the `Ingress`, so let's change the `values.yaml` file to disable them :
```yaml
# file whoami/values.yaml
serviceAccount:
  # Specifies whether a service account should be created
  create: false


ingress:
  enabled: false
```
### Customize our template
First thing first, let's configure our deployment. The only thing we'll change here is the image used for the application (by default, Helm generate a template for Nginx).
```yaml
# file whoami/values.yaml
image:
  repository: containous/whoami # Here, change nginx with the correct image

...

# file whoami/Chart.yaml
appVersion: v1.4.0
```
The service does not need any modification, as the default configuration works just fine for our example.

### Deploy our chart
To deploy our chart, the command is the same as the one we used for MySQL.
```bash
$ helm install whoami whoami/
NAME: whoami
LAST DEPLOYED: Tue Jan 21 10:59:51 2020
NAMESPACE: default
STATUS: deployed
REVISION: 1
NOTES:
...
```
Our chart is now deployed and you can check all the resources inside the cluster
```bash
$ kubectl get deploy,svc -l app.kubernetes.io/name=whoami
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/whoami   1/1     1            1           2m31s

NAME             TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)   AGE
service/whoami   ClusterIP   10.107.121.210   <none>        80/TCP    2m31s
```
We can also try and curl our service to see if everything works fine. To do that, we need to do it from inside the cluster as our service if of type `ClusterIP`. So below, we create a pod from which we curl our service :
```bash
# we create a simple pod based on centos:7
$ kubectl run centos --image=centos:7 --generator=run-pod/v1 --command sleep infinity
# we connect into the centos pod in order to call our service
$ kubectl exec -ti centos bash
[root@centos /]# curl whoami:80/api
{"hostname":"whoami-f97c46bf5-fzrq9","ip":["127.0.0.1","172.17.0.8"],"headers":{"Accept":["*/*"],"User-Agent":["curl/7.29.0"]},"url":"/api","host":"whoami","method":"GET"}
[root@centos /]# exit
```

Bravo ! You just created your first Helm chart, configured it, released it and made sure it is working !

## Cleanup

Every good thing has an end. It's time for some cleanup.
```bash
# delete our centos pod
$ kubectl delete po centos --force --grace-period=0
# uninstall our whoami Helm release
$ helm uninstall whoami 
release "whoami" uninstalled
# uninstall our mydb Helm release
$ helm uninstall mydb 
release "mydb" uninstalled
# make sure you have removed all the unnecessary resources
$ kubectl get all


NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   24h
```