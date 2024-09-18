# Kubernetes-MongoDB-Monitoring project

## ℹ️ Overview
This project focuses on:
- running a Kubernetes cluster locally using Minikube
- spinning up two applications, mongo-db (backend) and mongo-express (frontend) and ensuring that they interact with each other
- implementing monitoring with Prometheus/Grafana stack
- implementing ArgoCD tool in the cluster

The Kubernetes project consists of several yaml files. 

**mongo-db.yaml**: the configuration of the mongodb database, which serves as the backend of our project.
**mongo-service.yaml**: the "service" which allows the backend to be accessible on certain port, in the cluster.
**express-app.yaml**: the configuration of the mongo express application which serves as the frontend of our project.
**express-service.yaml**: the "service" which allows the frontend to be accessible on certain port, e.g. by our browser.


## :wrench: Kubernetes  
Both MongoDB and MongoExpress applications are pulled from the respective images from `Docker Hub`, and run in containers, in our Kubernetes cluster.

When setting up the express-app.yaml, we need to provide certain *environmental variables*, referencing the backend, and as well as *credentials* for the app to access MongoDB. 
For providing such configurations and secrets, we make use of the *configmap* and *secret* yaml files of kubernetes.

For simplicity, we set the applications to run in single replicas.

Start the cluster: 
```bash
minikube start, to start the cluster
```

Ensure that minikube is running, and move to the path with all Kubernetes manifests/yaml files. You can "apply" all files:
```bash
kubectl apply -f .
```

if you dont encode secrets in advance, you will get the following error when applyinh the *secret.yaml* file: `Secret in version "v1" cannot be handled as a Secret: illegal base64 data".`
To avoid that, encode them in base64: 
```bash
echo -n "XXXXXX" | base64
``` 

In turn, the cluster with decode them.

Now you can verify the available resources in Kubernetes, e.g. by running:

`kubectl get deployments`
`kubectl get pods`
`kubectl get services`
`kubectl get secrets`
`kubectl get configmaps`

You can also see details of a running pod, eg:
```bash
kubectl describe pod [PODNAME]
``` 

If your yaml files use multiple namespaces, make sure to either define the namespace in the yaml file, or specify the namespace when applying them:
```bash
kubectl apply -f . --namespace=[namespace-name]
```

When your deployments are running, you can port-forward your express service from the cluster to localhost, and access it at port 27017:
```bash
minikube service express-service
```

The Mongo Express app can access the MongoDB in the cluster, and from there you can interact with your NoSQL DB, e.g. create a Database, or Collection. 


## 🔨 ARGO CD

 ArgoCD is a popular continuous delivery (CD) tool specifically designed for Kubernetes. It offers several benefits that make it an effective solution for deploying and managing applications in Kubernetes environments. Below are the key advantages of using ArgoCD:

1. Declarative GitOps Approach. The entire application state (manifests, configurations, secrets, etc.) is stored in a Git repository. This provides several benefits:

- `Version Control`: Every change to the deployment configuration is versioned in Git, making it easy to track, audit, and roll back changes.
- `Single Source of Truth`: The Git repository serves as the definitive source of truth for the desired state of the Kubernetes cluster.
- `Declarative`: Kubernetes resources are managed declaratively. ArgoCD ensures that the cluster's actual state matches the desired state described in the Git repository.

2. Automated Syncing
ArgoCD constantly monitors the Git repository and the actual state of the Kubernetes cluster. It will automatically apply changes whenever there is a drift between the Git repository (desired state) and the cluster (actual state).

3. Application Health Monitoring
ArgoCD provides real-time monitoring of the applications deployed in the Kubernetes cluster, e.g. Deployments, StatefulSets, Services, etc. It can provide notifications if otherwise.

4.  Integrated with Popular Tools and Workflows
ArgoCD integrates seamlessly with tools that are commonly used in Kubernetes-based workflows, e.g. Kustomize and Helm, as well as CI/CD pipeline toolkits, e.g. Github Actions and Jenkins.

For setting up ArgoCD in the cluster, you can follow the [official documentation](https://argo-cd.readthedocs.io/en/stable/getting_started/)

After installation of ArgoCD, you can check the services in the argocd namespace that you have created, and port-forward the one called "argocd-server":
```bash
kubectl port-forward -n argocd svc/argocd-server 8080:443
```

To access the UI of ArgoCD in localhost:8080, you will need to extract the secret that resides in the argocd namespace,
```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o yaml
```

and then decode it: 
```bash
echo [SECRET] | base64 --decode
```

We then need to write a configuration file for ArgoCD, to connect it to the git repo, where our code -the kubernetes configuration files- is hosted. 
We can then place that file in the same level as the parent folder of the Kubernetes files. In my repo, it is called `application.yaml`.

To apply the configuration: 
```bash
kubectl apply -f application.yaml (while being on the write directory path)
```

Then, the ArgoCD UI has been updated with a new "application" created, as well as the relevant components, e.g. services and deployments, that have been created in the cluster, and are tracked now by ArgoCD.

As per our ArgoCD configuration, ArgoCD will **poll the Git repository every few minutes**, and overwrite any unexpected changes that might have been done by developers, imperatively.


## :mag: MONITORING

For monitoring, I will be using a collection of Kubernetes manifests, Grafana dashboards and Prometheus rules, thanks to existing Helm Chart [kube-prometheus-stack](https://artifacthub.io/packages/helm/prometheus-community/kube-prometheus-stack).

You need `Helm` installed for that.

Following the guidelines in the link above, one needs to first add the remote repo to the list of repos that will be used locally:
```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

With `helm repo list` you can see the repos you have access to.

Then, using a `release name` (e.g. "monitoring"), you can install the helm chart in the cluster: 
```bash
helm install [RELEASE_NAME] prometheus-community/kube-prometheus-stack
```

With `helm list`, you can verify the helm charts that have been installed, and with 
```bash
kubectl get svc
```
you can see the different services running after installation, in our Minikube cluster. 
We can access the Grafana UI of the monitoring-grafana service, by adjusting its node type to "Nodeport". 
For that to happen we can use: 
```bash
kubectl edit svc monitoring-grafana
```

Now that it is a Nodeport, we can expose it to localhost via: 
```bash
minikube service monitoring-grafana
```

For properly configuring the values of the helm chart, we can do: 
```bash
helm show values prometheus-community/kube-prometheus-stack > promstack_values.yaml
```

Then we can locate, and update, the configuration of the grafana installation, e.g. its password,  or its service type (NodePort):
```bash
helm upgrade promstack prometheus-community/kube-prometheus-stack --values=promstack_values.yaml
```

Configuration options for the grafana helm chart are available in the [documentation page](https://github.com/grafana/helm-charts/blob/main/charts/grafana/README.md). This chart is part of this full prometheus-grafana installation.

Now, to monitor a 3rd party service (in our case, MongoDB) using Prometheus, we will make use of the `Prometheus Exporter`, which is Prometheus' way to find our pod/deployment.

Each application that runs in our cluster, will need to have its own Exporter. The Exporter is a "translator" from apps' data to Prometheus understandable metrics. 
If you do "kubectl get pod" you can already see an Exporter installed in our cluster. What it does, it is translating the Metrics of our minikube nodes to Prometheus, under the hood.

There are 3 components you need when deploying an Exporter:
1. an Exporter Application, that exposes the metrics/endpoint
2. an Exporter Service, so that Prometheus can connect to it
3. a ServiceMonitor. Prometheus needs to know that there is a new Endpoint ready to be scraped, and we can do that using a ServiceMonitor for our Exporter application

Instead of creating ourselves all these files, we can install a **helm chart** that already has those put together, following the [guide here](https://github.com/prometheus-community/helm-charts/tree/main/charts/prometheus-mongodb-exporter).

We can follow a similar approach as above, with `helm show values`, in order to get the original export values, and then update them.

When done with the editing the configurations, we can run:
```bash
helm install mongodb-exporter prometheus-community/prometheus-mongodb-exporter -f exporter_values.yaml
```

By running `kubectl get po, svc` we see that the Exporter pod & service have been created. 
We need also to check if we have a releant Service Monitor, and we can do that with the command `kubectl get servicemonitor`.

After deploying the `Exporter`, we can port-forward its `Service`, and see the metrics of MongoDB being collected.
If we port-forward the `Prometheus service`, and access it in localhost, we can see the Endpoints that Prometheus is observing, and now the MongoDB exporter is one of them. 
Lastly, since those metrics are tracked by Prometheus, we can check them in a graphical form in `Grafana`.