# argoCD-tutorial
simple getting started guide for Argo CD

Argo CD is a declarative, GitOps continuous delivery tool for Kubernetes.
Argo CD follows the GitOps pattern of using Git repositories as the source of truth for defining the desired application state. 
Kubernetes manifests can be specified in several ways:

* kustomize applications
* helm charts
* ksonnet applications
* jsonnet files
* Plain directory of YAML/json manifests
* Any custom config management tool configured as a config management plugin

We will use simple yaml definitions and helm charts in this tutorial

This tutorial uses minikube on a mac.
It expects that homebrew and docker is already installed.


## fisrt steps
 
let's install minikube

    # check if virtualization is supported on macOS
    sysctl -a | grep -E --color 'machdep.cpu.features|VMX' 
    # VMX should be in the output (and it should be colored)
    brew install minikube
    brew install kubectl
    minikube start --driver=hyperkit
    # if that does not work you migh have to execute: brew install hyperkit 
    # you can later stop minikube via: minikube stop
    
let's connect our kubectl against local minikube

    kubectl config use-context minikube
    kubectl get pods --context=minikube
    minikube dashboard
    
let's start Argo CD

    kubectl create namespace argocd --context=minikube
    kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --context=minikube 
    
wait until everthing is started

![argocd](https://raw.githubusercontent.com/michaelgruczel/argoCD-tutorial/master/images/argocdstarted.png "argo cd") 

let's install an argo cd CLI:

    brew tap argoproj/tap
    brew install argoproj/tap/argocd
    
and make the API accessable locally:

    kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}' --context=minikube
    kubectl port-forward svc/argocd-server -n argocd 8080:443 --context=minikube
    # now you can access it via: localhost:8080
    # Argo CD runs both a gRPC server (used by the CLI), as well as a HTTP/HTTPS server (used by the UI). 
    # Both protocols are exposed by the argocd-server service object on the following ports:
    # 443 - gRPC/HTTPS
    # 80 - HTTP (redirects to HTTPS)     
    # our port forward is good enough for local tests in real system you would probably define ingress rules    

let's login and set up a password for our admin user

    kubectl get pods -n argocd -l app.kubernetes.io/name=argocd-server -o name | cut -d'/' -f 2
    # will return the inital password
    argocd login localhost:8080
    argocd account update-password

let's connect argo cd to our minikube cluster

    argocd cluster add    
    argocd cluster add minikube

let's connect this repository https://github.com/argoproj/argocd-example-apps to our 
cluster using the UI (can be done via API as well of course):

    open localhost:8080
    # login via admin user and created password
    # click new app and enter data
    #
    # app name: guestbook
    # project: default
    # sync policy: Manual
    # repo url: https://github.com/argoproj/argocd-example-apps.git 
    # path: guestbook
    # cluster in-cluster (https://kubernetes.default.svc)
    # namespace default
    
![createapp](https://raw.githubusercontent.com/michaelgruczel/argoCD-tutorial/master/images/createApp.png "create app")

The app is not synced, means the deployment in kubernetses differs from git definition.
Let's change that:
    
    argocd app get guestbook
    argocd app sync guestbook
    # this could be done over the UI for sure as well
    # for development cases you can even sync local definitions
    # argocd app sync APPNAME --local /path/to/dir/
    # normally you would enable automatically sync of apps
    argocd app set guestbook --sync-policy automated

You can configure auto prune logic 
(means if the service is removed in git it will be removed in kube as well)
and self healing 
(means if someone make changes to the cluster argoCD will override it)

    argocd app set guestbook --auto-prune
    argocd app set guestbook --self-heal

This deployment is based on simple yaml descriptions, see

    open https://github.com/argoproj/argocd-example-apps/tree/master/guestbook

applications can be configured in a ArgoCD via a config as well (we used the API).
This would like this for our case:

    apiVersion: argoproj.io/v1alpha1
    kind: Application
    metadata:
      name: guestbook
      namespace: argocd
    spec:
      project: default
      source:
        repoURL: https://github.com/argoproj/argocd-example-apps.git
        targetRevision: HEAD
        path: guestbook
      destination:
        server: https://kubernetes.default.svc
        namespace: guestbook

Argo cd will search in a folder named guestbook within https://github.com/argoproj/argocd-example-apps.git
for deployment configs.
    
## use helm 

It is possible to define more complex deployments, then just a simple app. 
For example we might want to deploy a full stack. Argo supports that, but more 
convient is a kubernetes package manager like helm with helm charts for it. 
A Chart is a Helm package. It contains all of the resource definitions necessary to run an application, tool, or service inside of a Kubernetes cluster. 
Helm Charts help you define, install, and upgrade even the most complex Kubernetes application.

    brew install helm
    
With Helm it is easy to share complex script stored in repositories. Let's add a repo and install mysql from that repo.
 
    helm repo add stable https://kubernetes-charts.storage.googleapis.com/
    helm search repo stable
    helm install stable/mysql --generate-name
    
mysql should be added to kube now

![mysql](https://raw.githubusercontent.com/michaelgruczel/argoCD-tutorial/master/images/mysql.png "mysql")
    
Let's remove mysql for now again
        
    helm ls
    helm uninstall <the name you got>
    
If you search for ready packages you the helm commands

    helm search hub wordpress
    
this will give you the repos including wordpress which you could install.
the website https://hub.helm.sh might be a good place to start searching.   
You can check whether a chart offers configuration options 
    
    helm show values stable/mysql
    echo '{mysqlPassword: xyz, mysqlDatabase: user0db}' > config.yaml
    helm install -f config.yaml stable/mysql --generate-name
    
You can then override any of these settings in a YAML formatted file, and then pass that file during installation.    
Of course you can set uo your own helm charts, which normally has this structure:

    mychart/
      Chart.yaml
      values.yaml
      charts/
      templates/
         deployment.yaml
         service.yaml
         helpers.tpl
         configmap.yaml
         NOTES.txt
      ...

The folders mean:

* templates - When Helm evaluates a chart, it will send all of the files in the templates/ directory through the template rendering engine.
* deployment.yaml - A basic manifest for creating a Kubernetes deployment
* service.yaml - A basic manifest for creating a service endpoint for your deployment
* helpers.tpl - A place to put template helpers that you can re-use throughout the chart
* NOTES.txt - The "help text" for your chart. This will be displayed to your users when they run helm install
* configmap.yaml - a ConfigMap is simply a container for storing configuration data. Other things, like pods, can access the data in a ConfigMap
* values.yml - default values for a chart
* Chart.yaml

Take a look into https://github.com/helm/charts/tree/master/stable to understand all details.
when you want to create your own chart start with:

    helm create mychart

after you set up everything, you can apply it via

    helm install mychart ./mychart

Configure the help app the same way, you define the other app over the UI

let's override the default value with specific values

    git clone git@github.com:argoproj/argocd-example-apps.git  
    cd argocd-example-apps      
    argocd app set helm-guestbook --values values-production.yaml

you could ovveride simple parameters or release names

    argocd app set helm-guestbook -p service.type=LoadBalancer
    argocd app set helm-guestbook --release-name myRelease
 
## other important links

some more interesting links:

* if you want to try it with a private repo, then check https://argoproj.github.io/argo-cd/user-guide/private-repositories/