# Kickstart your projects on GCP

## Pre-requisites

* a GCP account with billing activated
* An idea
* A project aready initialized in GCP (we will call it kickstarter)

## What you will get

* HTTPs services
* Subdomains for each service
* Backend, monitoring, admin protected by oauth2
* Your project frontend, ofc

## Manage your GCP project (5mn)

Make sure your config points at your kickstarter project.
Get the projectid of your kickstarter project:
```
gcloud projects list
PROJECT_ID          NAME         PROJECT_NUMBER
kickstarter-xxxxx  kickstarter   xxxx
```

then set it a the current gcloud project:
```
gcloud config set project kickstarter-xxxxx
```

## Start your cluster (5mn)

Start small!

```
gcloud container clusters create cubi --machine-type=f1-micro --preemptible
Creating cluster cubi...done.
Created [https://container.googleapis.com/v1/projects/YOURPROJECT/zones/YOURZONE/clusters/cubi].
kubeconfig entry generated for cubi.
NAME  ZONE          MASTER_VERSION  MASTER_IP       MACHINE_TYPE  NODE_VERSION  NUM_NODES  STATUS
cubi  asia-east1-a  1.5.3           <YOURIP>  f1-micro      1.5.3         3          RUNNING
```

## Get a static IP

This IP will be used to access our cluster:
```
gcloud compute addresses create kubernetes-ingress --global
```

## Choose a name for your project and get your domain (5mn)

if you lack of imagination, you can still go to [namimum](http://www.naminum.com/)

Once you have it, go to [freenom](http://freenom.com) and get your project domain now!

In the DNS, link the static IP created above to the new domain.
You can add all the subdomains you wish to link.

We'll take coolnamegcp.tk as an example for this demo.

## Deploy a first service (gitea)

```
kubectl apply -f gitea-d.yml
```

As a side note, you can see the use of configmap and secret in the gitea deployment (gitea-d.yml).

## Check that things are ok and by using loadBalancer Service

```
kubectl apply -f gitea-svc-lb.yml
```

```
kubectl get svc --watch
gitea-lb   10.15.243.115   <YOURIP>   80:30884/TCP   3m
```

You can see @http://youip that gitea admin is accessible.
Create a repo and clone it!

## Change service from loadBalancer to nodePort

remove the one before

```
kubectl delete -f gitea-svc-lb.yml
```

and create the new one

```
kubectl apply -f gitea-svc.yml
```

## Create a second service (an example, rethinkb)

```
kubectl apply -f rethinkdb-d.yml
```

## Make it available for other pods in the cluster (type ClusterIP)

```
kubectl apply -f rethinkdb-svc-internal.yml
```


## Check that things are ok and by using loadBalancer Service

```
kubectl apply -f rethinkdb-svc-lb.yml
```

```
kubectl get svc --watch
rethinkdb-lb   10.15.243.115   <YOURIP>   80:30884/TCP   3m
```

You can see @http://youip that rethinkdb admin is accessible.

## Change service from loadBalancer to nodePort

remove the one before:
```
kubectl delete -f rethinkdb-svc-lb.yml
```

and create the new one:
```
kubectl apply -f rethinkdb-svc.yml
```

## Deploy a third service (wwwapp connecting to rethinkdb)

Push your app image in google container registry.

```
kubectl apply -f wwwapp-d.yml
```

## Check that things are ok and by using loadBalancer Service

```
kubectl apply -f wwwapp-svc-lb.yml
```

```
kubectl get svc
wwwapp-lb   10.15.243.115   <YOURIP>   80:30884/TCP   3m
```

You can see @http://youip that your service is accessible.

## Change service from loadBalancer to nodePort

remove the one before

```
kubectl delete -f wwwapp-svc-lb.yml
```

and create the new one

```
kubectl apply -f wwwapp-svc.yml
```

## Make it accessible through specified URLs

```
kubectl apply -f ingress-tls.yml
```

more info about ingress [here](https://kubernetes.io/docs/user-guide/ingress/).
If you don't want tls, you can use ingress-notls.yml and stop here.

## Deploy kube-lego

Make sure first that all your subdomains are accessible.

```
kubectl apply -f lego.yml
```

kube-lego will watch over all ingress with specific `kubernetes.io/tls-acme: "true"` annotations and contact let's encrypt to generate and renew certs automatically.
Better explanation [here](https://blog.jetstack.io/blog/kube-lego/).

## Check!

Unfortunately, as of now, it seems Google GKE ingress controller doesn't yet support SNI (Server Name Indication), so only one cert can be served for one IP,
which is why you will have cet name mismatch.

You can use nginx controllers instead, in the meantime.

You can [check](https://www.ssllabs.com/ssltest/) the quality of the certification.

## Implementing OAUTH2 proxying for specific services

Remove and stop rethinkdb service and deployments:
```
kubectl delete -f rethinkdb-d.yml -f rethinkdb-svc.yml
```

and create the oauth2 ones
```
kubectl apply -f rethinkdb-d-oauth2.yml -f rethinkdb-svc-oauth2.yml
```

Now check if you can still access the backend service!
Note that the callback URL is not implemented in this example, the point here is to see the oauth2 prompt.

## Don't forget to delete it :)

```
kubectl delete -f ingress-tls.yml
gcloud container clusters delete cubi
```

## Many other examples available in the folder!

Check grafana if you want to see configmap, secrets for example.
Grafana has issues with GKE ingress controller (probably redirection instead of 200?)