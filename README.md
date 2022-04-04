## Introduction

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.

ArgoCD is watching over a git repository that includes the application manifest and makes sure the actual running instance of it is meeting the desired state.

This article explains how to deploy Argo CD from the OperatorHub and how to manage applications across one or many clusters in a production environment.

Here are a few benefits of using ArgoCD:

* Automated deployment of applications to specified target environments

* Support for many configs management/templating tools (Kustomize, Helm, Ksonnet, Jsonnet, plain-YAML)

* Ability to manage and deploy to many clusters

* Rollback/Roll-anywhere to any application configuration committed in Git repository

* Web UI, which provides a real-time view of application activity

## Why the Operator?

Operators are extensions to the existing Kubernetes API and can achieve the effects of having a human operator that is taking care of the application lifecycle.

The ArgoCD Operator provides:

* Fast configuration and installation of the Argo CD components.

* Seamless upgrades to the Argo CD components.

* Back up and restore an Argo CD cluster from a point in time or on a recurring schedule.

* Aggregate and expose the metrics for Argo CD using Prometheus and Grafana.

* Autoscale the Argo CD components.

## Prerequisites

Before you begin, you should:

* Deploy one or more Vultr Kubernetes Cluster with at least 3 nodes.

* Have cluster-admin privileges.

* Have access to a Linux machine with `kubectl` and `git`.

* Have an application you want to manage

* Have a `git` repository

## Installing the ArgoCD from the OperatorHub.io

The Operatorhub is a public repository for Kubernetes operators which everyone can publish or download operators.

To begin installing:

1. Download the `installation script` of the OperatorHub

       $ curl -sL <https://github.com/operator-framework/operator-lifecycle-manager/releases/download/v0.20.0/install.sh> | bash -s v0.20.0

2. Save the following as `argocd-subscription.yaml`

       apiVersion: operators.coreos.com/v1alpha1
       kind: Subscription
       metadata:
        name: argocd-operator
        namespace: operators
        spec:
        channel: alpha
       name: argocd-operator
       source: operatorhubio-catalog
        sourceNamespace: olm
       config:
       env:
        - name: ARGOCD_CLUSTER_CONFIG_NAMESPACES
          value: '*'
        - name: DISABLE_DEFAULT_ARGOCD_INSTANCE
          value: 'true'

3. Create the `argocd-subscription.yaml`

       $ kubectl create -f argocd-subscription.yaml

4. After installation, watch your operator come up using `kubectl get csv -n operators`. After a few seconds this should be the output:

       NAME                                 DISPLAY     VERSION     REPLACES                        PHASE
       argocd-operator.v0.2.1     Argo CD     0.2.1              argocd-operator.v0.2.0   Succeeded

If the output you are getting is different, refer to TroubleShooting.

## Create the ArgoCD Instance and Set up the UI

After a successful installation of the operator, you need to create the ArgoCD instance.

Create an ArgoCD manifest file `argocd.yaml` with the following content:

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

1. Deploy the `argocd.yaml`.

       $ kubectl create -n argocd -f argocd.yaml
2. Run the command `kubectl get pods -n argocd` to see the argocd pods. The results should look like hist:

       NAME                                  READY   STATUS    RESTARTS   AGE
       argocd-application-controller-0       1/1     Running   0          4m27s
       argocd-dex-server-7bff75cd48-smgkw    1/1     Running   0          4m27s
       argocd-redis-6f7cfddbcb-2vwf5         1/1     Running   0          4m27s
       argocd-repo-server-78d874c96c-p5nqr   1/1     Running   0          4m27s
       argocd-server-64b56b96f5-j27p4        1/1     Running   0          4m27s

3. Deploy ingress-nginx controller.

       $ kubectl apply -f <https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.1/deploy/static/provider/cloud/deploy.yaml>

4. Visit the [Vultr Load Balancers dashboard](https://my.vultr.com/loadbalancers) and get the IP Address of the Load Balancer.

5. Create a `DNS` record in your domain that points to the IP of the Load Balancer.

6. Save the ArgoCD UI `ingress`  yaml to a file `argocd-ingress.yaml`.

       $ kubectl get ingress argocd-server -n argocd -o yaml > argocd-ingress.yaml
7. Edit the `argocd-ingress.yaml` and replace the <YOUR_DOMAIN> with the domain that you have created A record earlier.

       apiVersion: networking.k8s.io/v1
       kind: Ingress
       metadata:
         annotations:
           kubernetes.io/ingress.class: nginx
           nginx.ingress.kubernetes.io/backend-protocol: HTTP
           nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
         labels:
           app.kubernetes.io/managed-by: argocd
           app.kubernetes.io/name: argocd-server
           app.kubernetes.io/part-of: argocd
         name: argocd-server
         namespace: argocd
       spec:
         rules:
         - host: <YOUR_DOMAIN>
           http:
             paths:
             - backend:
                 service:
                   name: argocd-server
                   port:
                     name: http
               path: /
               pathType: ImplementationSpecific
         tls:
         - hosts:
           - <YOUR_DOMAIN>
           secretName: argocd-secret

8. Apply the changes you made to the `argocd-ingress.yaml`.

       $ kubectl apply -f  argocd-ingress.yaml

9. Set a new admin password for the ArgoCD authentication. Replace "mypassword" with your new password.

       $ kubectl -n argocd patch secret argocd-cluster -p '{"stringData": { "admin.password": "mypassword" }}'

10. Go to your domain to access the ArgoCD login page and log in with the username `admin` and the password you set.

     If you're getting blocked by the browser, follow this tutorial on [How to bypass HTTPS warning](https://www.vultr.com/docs/how-to-bypass-the-https-warning-for-self-signed-ssl-tls-certificates). To get rid of this message completely check out [Vultr integration with cert-manager](https://github.com/vultr/cert-manager-webhook-vultr) and secure the route with HTTPS certificate.

> Sometimes the DNS takes a few hours to update. If your domain doesn't work just yet, you should wait a couple of hours before trying again.

## Set up a Git Repository with an Application Manifest

For the ArgoCD to work with your application, you need to put all the resources that make up the application in a git repository. The resources can be plain yamls, helm charts, and so on.

Here is an example of a [repository with a basic manifest]((https://github.com/sahar2339/example-argocd-app)) (Namespace, Deployment, Service, Ingress)

The application is a basic python REST API using flask, you can find the [source code on GitHub](https://github.com/sahar2339/example-python-app).

The structure of the repository should be:
In the root directory of the repository, you need to make a directory for each application you want to deploy. Inside every directory, you need to include the yamls of the application.
ArgoCD knows to go over the yamls and deploy them.

## Deploy Application from Git

You can deploy your applications from the ArgoCD UI or the CLI.
> Your git repository must be public for the deployment to succeed, if you would like to deploy from a private repository refer to “Deploy from a Private Git Repository using SSH key”

### From the UI

1. Go to the domain you set for the ArgoCD UI.

2. Login  and click "New APP".

    ![Create App](https://i.imgur.com/ddWZcLa.png)

3. Fill out the form with your git repositories and other parameters.

    ![Application form](https://i.imgur.com/ttkYiTp.png)
    ![Application form](https://i.imgur.com/djf0v8t.png)

4. Click "Create".

    ![Create button](https://i.imgur.com/EAN0kUI.png)

5. Sync the app.

    ![Sync App](https://i.imgur.com/knnAdmK.png)

### From the CLI

1. Install ArgoCD cli

       $ curl -sSL -o /usr/local/bin/argocd <https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64>

       $ chmod +x /usr/local/bin/argocd

2. Create your app (replace `myapp`, `repo`, `path`, and `namespace` with your app parameters.

       $ argocd app create name --repo repo --path path --dest-server <https://kubernetes.default.svc> --dest-namespace namespace

3. Sync the app.

       $ argocd app sync myapp

## Connect a second Vultr Kubernetes Cluster

If you want the ArgoCD to manage and deploy to another cluster, you need to connect it to the ArgoCD local cluster.

1. Login into your ArgoCD local cluster with the `domain` you set earlier

       $ argocd login <http://domain:443> --insecure

2. Go to your second cluster at [Vultr Kuberenets dashboard](https://my.vultr.com/kubernetes) and download its config.

3. Point your `KUBECONFIG` environment variable to the file you downloaded

       $ export KUBECONFIG=/root/second-kubeconfig

4. Get the cluster's context

       $ kubectl config get-contexts -o name

5. The output should be looking like hist:

       vke-XXXXX-XXXX-XXXX-XXXXXXXXXXX
6. Add the `context` to ArgoCD.

       $ argocd cluster add vke-XXXX-XXXX-XXXXX-XXXXX-XXXXX -y --name=<my-second-cluster>
7. Verify that the new cluster is in the clusters list.

       $ argocd cluster list

## Deploy from a GitHub Private Repository Using SSH key

When in a production environment or when your application is confidential, the git repository you deploy from should be private and not public.
This section explains how to generate an SSH key and use it with ArgoCD.

1. Set your repository visibility to private.

2. You might already have an SSH key pair on your machine. You can check to see if one exists by listing the contents in the `.ssh` folder.

       $ ls ~/.ssh

3. If you see  `id_ed25519.pub`, you already have a key pair and you don't have to generate a new one.
 If you don't see any files, you need to generate a new key.
 You can follow this article on [How to generate SSH keys](https://www.vultr.com/docs/how-do-i-generate-ssh-keys), make sure to use the `ED25519` format.

4. Copy the contents of `id_ed25519.pub`

       $ cat ~/.ssh/id_ed25519.pub

5. Add your public key to GitHub by login into your account and clicking settings -> SSH and GPG keys-> New SSH key and paste your `SSH key`.

6. Add the SSH key to ArgoCD using the following command, make sure to replace `user`, and `repo` with your GitHub repository.

       $ argocd repo add git@github.com:<user>/<repo>.git --ssh-private-key-path ~/.ssh/id_ed25519.pub

## Create Unprivileged Users for ArgoCD

By default, the first user in the ArgoCD is the `admin` user, who has access to do everything from the ArgoCD perspective.
In a secure production environment, you may want to have many users with different roles and permissions interact with your applications.

Create a new user with the following steps:

1. login into ArgoCD using the CLI with your `domain`.

       $ argocd login <http://domain:433> --insecure

2. List the users, you should get only the default `admin`.

       $ argocd account list

3. Save the ArgoCD `ConfigMap` to a file `argocd-cm.yml` by running:

       $ kubectl get configmap argocd-cm -n argocd -o yaml > argocd-cm.yml

4. Make the following changes to the ConfigMap, make sure to replace `<username>` with the new username:

       data:  
           accounts.<username>: apiKey, login

5. This adds a new user and allows them to process an API key and login via the CLI and UI.
 Apply the changes by running:

       $ kubectl apply -f argocd-cm.yml

6. Run `argocd account list` to see the freshly created user.

7. Update the User Password by executing the following command, make sure to replace `username` and `password` with your real values.

       $ argocd account update-password --account <username> --new-password <password>

8. By default the new user has **read-only** access. If you want the user to have other permissions, update the RBAC config map. Run the following command to save the `ConfigMap`:

       $ kubectl get configmap argocd-rbac-cm -n argocd -o yaml > argocd-rbac.yml

9. Create a `role` and specify what the user has access to, by adding the following to the `argocd-rbac.yml`.
For example  `repo-manager` role grants the user access to get, create and update repositories.

       data:  
         policy.csv: |  
             p, role:repo-manager, repositories, get, *, allow  
             p, role:repo-manager, repositories, create,*, allow  
             p, role:repo-manager, repositories, update, *, allow    
10. Assign the user to the `repo-manager` role, by adding the fllowingo line under `policy.csv`:

        g, <new-user>, role:repo-manager
11. Apply the `argcd-rbac.yml` `ConfigMap`

        $ kubectl apply -f argcd-rbac.yml

## Secure the ArgoCD Ingress

When the ArgoCD ingress is first created by the operator, anyone can access it from anywhere.
To offer extra security for the ingress, you can add some firewall rules to make sure only you or your team can access it.
But before you can do that, you need to make sure the `ingress-nginx` learns the real clients IPs from the requests and not the load balancer one.

1. First, go to [Vultr Load Balancers dashboard](https://my.vultr.com/loadbalancers). Then select your load-balancer.
Navigate to `configuration`, and enable `Proxy Protocol`.
This forwards the real client IP with every request the load balancer receives.

2. Pull the `ingress-nginx-controller`  ConfigMap and save it to a file `ingress-nginx-config.yaml`

       $ kubectl get cm ingress-nginx-controller -n ingress-nginx -o yaml > ingress-nginx-config.yaml

3. Enable `Proxy Protocol` within Nginx, by adding the following line to the `ingress-nginx-config.yaml` ConfigMap.

       data:
         use-proxy-protocol: "true"
4. Apply the configuration.

       $ kubectl apply -f ingress-nginx-config.yaml
5. Restart the `ingress-nginx-controller` pods.

       $ kubectl rollout restart deployment ingress-nginx-controller -n ingress-nginx
6. Get the ArgoCD `ingress` and save it to `ingress.yaml`

       $ kubectl get ingress argocd-server -o yaml > ingress.yaml  
7. Edit `ingress.yaml` and add the following line under `annotations`, make sure to replace `<some IP>` with the IPs you want to communicate ArgoCD from.
You can find your computer IP from sites like [What is my IP](https://whatismyipaddress.com).

       nginx.ingress.kubernetes.io/whitelist-source-range: "<some IP>, <some IP>"

   This gives the `Nginx` allowlist of IPs it accepts requests from, any other request recives `403 forbidden` and can not access.

8. Apply the `ingress.yaml`

       $ kubectl  apply -f ingress.yaml

   For more complex scenarios you can add `server-snippet` to the  `ingress.yaml` and specify the firewall logic :

       nginx.ingress.kubernetes.io/server-snippet: |
       location / {
        # block one IP
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

3. The Load balancer is not connected to the cluster.

4. Network problems.

### What to do if the Operator Installment Failed or Stuck

If the operator doesn't seem to work, try to watch its status with:

    $ kubectl describe -n operators csv argocd-operator.v0.2.0

And try to look for inductive information on why the installation failed.

### What to do if the ArgoCD Instance doesn't Work as Expected

If you can't communicate with the argocd you have created, try to watch its status and verify that all the resources are exist.

    $ kubectl describe argocd -n argocd

    $ kubectl get cm,secret,deploy -n argocd

## More Information

[The ArgoCD project](https://github.com/argoproj/argo-cd)
