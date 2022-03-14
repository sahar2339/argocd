# Introduction

לסיים את האפליקישן סט, חוקי פיירוול לאינגרס, לעשות פרוגקט בשביל הרשאות לאפליקציה
  

  

Argo CD is an open-source tool that can be used in a declarative way, to achieve continuous delivery for Kubernetes in a production environment.
What ArgoCD does is watch over a git repository that includes the application manifest, and makes sure the actual running instance of the application is up to date with the repository.
Vultr Kubernetes Engine with the addition of ArgoCD operator can lead to a powerful, automated, auditable, and easy-to-understand application life cycle.

  

This article explains how to deploy Argo CD from the OperatorHub and how to manage applications across one or multiple clusters.

  

  

Here are a few benefits of using ArgoCD:

  

  

* Automated deployment of applications to specified target environments

  

* Support for multiple config management/templating tools (Kustomize, Helm, Ksonnet, Jsonnet, plain-YAML)

  

* Ability to manage and deploy to multiple clusters

  

* Rollback/Roll-anywhere to any application configuration committed in Git repository

  

* Web UI which provides a real-time view of application activity

  

  

## Why the operator?

  

  

Operators are clients of the Kubernetes API that act as controllers for a Custom Resource, operators can be used as extensions to the already existing Kubernetes API and follow Kubernetes principles to achieve the effects of having a human operator that is taking care of the application lifecycle, In other words, the Operator does the "Operation" part for you.


Much like the way Kubernetes watch over Deployments or Pods and always try to reach their desired state, the operator is looking after a custom resource.

In this example your custom resource is ArgoCD.



## Prerequisites

Before you begin, you should:

 

* Deploy one or more Vultr Kubernetes Cluster with at least 3 nodes.

* Have access to a Linux machine with `kubectl` and `git`.

 
* Have an application you want to manage
* Have a `git` repository


  

## Installing the ArgoCD from the OperatorHub.io


The operator hub is A public repository for Kubernetes operators which everyone can publish or download operators.


To begin installing: 

  
  

1. Download the `installation script` of the OperatorHub 

  

		$ curl -sL https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.20.0/install.sh | bash -s v0.20.0



2. Install the `ArgoCD operator`

 

		$ kubectl create -f https://operatorhub.io/install/argocd-operator.yaml



3. After installation, watch your operator come up using the next command.

  
		$ kubectl get csv -n operators

  
After a few seconds this should be the output:

  
	NAME                                 DISPLAY     VERSION     REPLACES                        PHASE
	argocd-operator.v0.2.1     Argo CD     0.2.1              argocd-operator.v0.2.0   Succeeded


  

If the output you are getting is different, refer to TroubleShooting.

  

## Create the ArgoCD instance and set up the UI

 
  

After a successful installation of the operator, you need to create the ArgoCD instance.

Create an ArgoCD manifest file `argocd.yaml` with the following content


> This example YAML utilizes ingress and load balancer as a way to access the UI.



	apiVersion: v1
	kind: Namespace
	metadata:
	  labels:
	    kubernetes.io/metadata.name: argocd
	  name: argocd
	---
	apiVersion: argoproj.io/v1alpha1
	kind: ArgoCD
	metadata:
	  name: argocd
	  namespace: argocd
	  labels:
	      example: basic
	spec:
	  server:
	    ingress:
	      enabled: true
	    insecure: true

1. deploy the `argocd.yaml`.

  

		$ kubectl create -n argocd -f argocd.yaml

2.  Run the command `kubectl get pods -n argocd ` to see the argocd pods being created. the results should look similar to:
3. 
		NAME                                  READY   STATUS    RESTARTS   AGE
		argocd-application-controller-0       1/1     Running   0          4m27s
		argocd-dex-server-7bff75cd48-smgkw    1/1     Running   0          4m27s
		argocd-redis-6f7cfddbcb-2vwf5         1/1     Running   0          4m27s
		argocd-repo-server-78d874c96c-p5nqr   1/1     Running   0          4m27s
		argocd-server-64b56b96f5-j27p4        1/1     Running   0          4m27s



4. Deploy ingress-nginx controller.

		$ kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml

  

  

5. Visit the Load Balancers dashboard at <https://my.vultr.com/loadbalancers> and retrieve the IP Address of the Load Balancer.

  

  

6. Create a `DNS` record in your domain that points to the IP of the load balancer.

  

  

7. Save the ArgoCD UI `ingress`  yaml to a file `argocd-ingress.yaml`.
		
		# kubectl get ingress argocd-server -o yaml > argocd-ingress.yaml

  
8. Edit the `argocd-ingress.yaml` and replace the <YOUR_DOMAIN> with the domain that you have created A record earlier.

		$ vi argocd-ingress.yaml

    
		spec:
		rules:
		- host: <YOUR_DOMAIN>
		http:
		paths:
		- backend:
		service:
		name: example-argocd-server
		port:
		name: HTTP
		path: /
		pathType: ImplementationSpecific
		tls:
		- hosts:
		- <YOUR_DOMAIN>
		secretName: argocd-secret
		status:
		load-balancer:
		ingress:
		- IP: xxx.xxx.xxx.xxx 

  
 9. Apply the changes you made to the `argocd-ingress.yaml`.
 
		 $ kubectl apply -f  argocd-ingress.yaml
		 

10. Set a new admin password for the ArgoCD authentication. Replace "mypassword" with your desired password.

 

		$ kubectl -n argocd patch secret argocd-cluster \ -p '{"stringData": { "admin.password": "mypassword" }}'

11. Go to your domain to access the ArgoCD login page and log in with the username `admin` and the password you just set.

  

> Sometimes the DNS takes a few hours to update. If your domain doesn't work just yet, you should wait a couple of hours before trying again.


## Setup a git repository with an application manifest

For the ArgoCD to work with your application you need to put all the resources that make up the application in a git repository. The resources can be plain yamls, helm charts,  etc.

Here is an example of a repository with a simple manifest (Namespace, Deployment, Service, Ingress): 
<https://github.com/sahar2339/example-argocd-app>
The application is a basic python REST API using flask, you can find the source code here:
<https://github.com/sahar2339/example-python-app>

The structure of the repository should be as follows:
In the root directory of the repository, you need to make a directory for each application you want to deploy. Inside every directory, you need to include the yamls of the application. 
ArgoCD will know to go over the yamls and deploy them.


  

## Deploy your application from git

You can deploy your applications from the ArgoCD UI or the CLI.
> Your git repository must be public for the deployment to succeed, If you would like to deploy from a private repository refer to "Deploy from a private git repository using SSH key" 


### From the UI
  
1. Go to the domain you set for the ArgoCD UI

2. Create a project for your application

2. Login  and click "New APP"

  img

3. Fill the form with your git repositories and other parameters.

   img

4. Click "Create"

  img

5. Sync the app

img

### From the CLI

  

1. Install ArgoCD cli

  

		$ curl -sSL -o /usr/local/bin/argocd https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64
		$ chmod +x /usr/local/bin/argocd

2. Create an ArgoCD project for your application.

	

4. Create your app (replace `myapp`, `repo`, `path`, and `namespace` with your app parameters.

  

		$ argocd app create name --repo repo --path path --dest-server https://kubernetes.default.svc --dest-namespace namespace

5. Sync the app.

  

		$ argocd app sync myapp

  

## Connect a second Vultr Kubernetes Cluster

If you want the ArgoCD to manage and deploy to another cluster, you need to connect it to the ArgoCD local cluster.

  

1. Login into your ArgoCD local cluster with the `domain` you set earlier 

  

		$ argocd login <http://domain:443> --insecure

2. Go to your second cluster at <https://my.vultr.com/kubernetes> and download its config.

  

3. Point your `KUBECONFIG` environment variable to the file you just downloaded

		$ export KUBECONFIG=/root/second-kubeconfig

4. Get the cluster's context

		$ kubectl config get-contexts -o name

5. The output should be looking similar to that:

		vke-XXXXX-XXXX-XXXX-XXXXXXXXXXX

7. Add the `context` to ArgoCD

 
		$ argocd cluster add vke-XXXX-XXXX-XXXXX-XXXXX-XXXXX -y

8. Verify that the new cluster is in the clusters list.

		$ argocd cluster list


## Multi cluster applications with ApplicationSets


  
##  Security In a production environment 

### Deploy from a GitHub private git repository using SSH key

When in a production environment or when your application is confidential, the git repository you deploy from should be private and not public.
This section will explain how to generate an SSH key and use it with ArgoCD. 

1.  Set your repository visibility to private.

2. You might already have an SSH key pair on your machine. You can check to see if one exists by listing the contents in the `.ssh` folder.
	
		$ ls ~/.ssh
	
3. If you see  `id_ed25519.pub`, you already have a key pair and you don't have to generate a new one.
 If you don't see any files you need to generate a new key. 
 You can follow this article, make sure to use the `ED25519` format.
<https://www.vultr.com/docs/how-do-i-generate-ssh-keys>

4. Copy the contents of `id_ed25519.pub`


	   $ cat ~/.ssh/id_ed25519.pub

5.  Add your public key to GitHub by login into your account and clicking settings -> SSH and GPG keys-> New SSH key and paste your `SSH key`.

6. Add the SSH key to ArgoCD using the following command, make sure to replace `user`, and `repo` with your GitHub repository.
	
		$ argocd repo add git@github.com:<user>/<repo>.git --ssh-private-key-path ~/.ssh/id_ed25519.pub

  
### Create unprivileged users for ArgoCD
By default, the first user that is created with the ArgoCD is the `admin` user, who has access to do everything from the ArgoCD perspective.
In a secure production environment, you may want to have multiple users with different roles and permissions to interact with your applications. 

Create a new user with the following steps:

1. login into ArgoCD using the CLI with your `domain`.
	
	   $ argocd login <http://domain:433> --insecure

2. List the current users, you should get only the default `admin`.
	
	   $ argocd account list

3. Save the ArgoCD `ConfigMap` to a file `argocd-cm.yml` by running:

	   $ kubectl get configmap argocd-cm -n argocd -o yaml > argocd-cm.yml

4. Now make the following changes to the ConfigMap, make sure to replace `<username>` with the new username:

	   data:  
         accounts.<username>: apiKey, login

5. This will add a new user and allow them to process an API key as well as login via the CLI and UI.
	Apply the changes by running:
	
	   $ kubectl apply -f argocd-cm.yml

6. Run `argocd account list` to see the freshly created user.

7. Update the User Password by executing the following command, make sure to replace `username` and `password` with your real values.

	   $ argocd account update-password --account <username> --new-password <password>

8. By default the new user has **read-only** access. If you want the user to have other permissions, the RBAC config map will need to be updated. Run the following command to save the `ConfigMap`:

	   $ kubectl get configmap argocd-rbac-cm -n argocd -o yaml > argocd-rbac.yml

9. Now create a `role` and specify what the user has access to, by adding the following to the `argocd-rbac.yml`.
For example  `repo-manager` role grants the user access to get, create and update repositories.
 
		data:  
		  policy.csv: |  
	       p, role:repo-manager, repositories, get, *, allow  
	        p, role:repo-manager, repositories, create, *, allow  
	       p, role:repo-manager, repositories, update, *, allow  

10. Assign the user to the `repo-manager` role, by adding:

		g, <new-user>, role:repo-manager

11. Apply the `argcd-rbac.yml` `ConfigMap`

	    $ kubectl apply -f argcd-rbac.yml

### Secure the UI ingress

When the ArgoCD ingress is first created by the operator, anyone can access it from anywhere.
To provide additional security for the ingress, some firewall rules can be added to make sure only you or your team can access it.

1. First get the ArgoCD `ingress` and save it to `ingress.yaml`
	
	   $ kubectl get ingress argocd-server -o yaml > ingress.yaml  
	   
2. Edit `ingress.yaml` and add the following line under `annotations`, make sure to replace `<some IP>` with the IPs you want to manage ArgoCD from.
You can find your computer IP from sites like <https://whatismyipaddress.com>.
 

	   nginx.ingress.kubernetes.io/whitelist-source-range: "<some IP>, <some IP>"
	   
This will provide the `Nginx` whitelist of IPs it will accept requests from, any other request will be dropped.

3. Apply the `ingress.yaml`

	   $ kubectl  apply -f ingress.yaml

For more complex scenarios you can add `server-snippet` to the  `ingress.yaml` and specify the firewall logic :

	nginx.ingress.kubernetes.io/server-snippet: |
	location / {
	  # block one ip
	  deny    192.0.2.0;
	  # allow anyone in 198.51.100.0/24;
	  allow   198.51.100.0/24;
	  # drop rest of the world 
	  deny    all;
	}




## Troubleshooting

  

Here are some common reasons for things to not work as expected:

  

1. The cluster is out of resources.

2. You don't have cluster-admin privileges.

3. The Loadbalancer is not connected to the cluster.

4. Network problems.


  

#### What to do if the operator installment failed or stuck

  

If the operator doesn't seem to work correctly try to watch its status with

  

	$ kubectl describe -n operators csv argocd-operator.v0.2.0

And try to look for inductive information on why the installation failed.

  

#### What to do if the ArgoCD instance doesn't work as expected

  

If you cant communicate with the argocd you just created try to watch its status and verify that all the resources are exist

  

	$ kubectl describe argocd -n argocd

	$ kubectl get cm,secret,deploy -n argocd

  

## More information

  

ArgoCD project: <https://github.com/argoproj/argo-cd>
