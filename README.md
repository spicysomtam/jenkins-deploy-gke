# Introduction

Create an Google Cloud Platform (gcp) kubernetes cluster (gke) by running a Jenkins pipeline. Sometimes you just want to deploy something without having to figure out how to do it; on demand so to speak.

This is designed to be a simple deploy; just get a cluster up and running.

Official instructions are [here](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-regional-cluster).

Since creating a gke cluster is fairly simple with the `gcloud` cli, there is no need to use something like terraform, etc

This pipeline does not create a user managed service account to create the cluster; instead a service account previously defined in Jenkins is used. Its a common pattern that credentials are defined in Jenkins manually (eg a service account) and then pipelines use the credential.

# Credentials for gcp

Pipeline uses a service account with a key. You can create this via the `gcloud` command line or via the Azure portal; see instructions [here](https://cloud.google.com/iam/docs/creating-managing-service-accounts). I used these `gcloud` commands to create a service account with suitable roles:
```
gcloud iam service-accounts  create jenkins-gcp  --description="Jenkins access" --display-name="Jenkins access"
gcloud iam service-accounts list
DISPLAY NAME                            EMAIL                                                     DISABLED
Jenkins access                          jenkins-gcp@annular-haven-312209.iam.gserviceaccount.com  False
Compute Engine default service account  822320205400-compute@developer.gserviceaccount.com        False
App Engine default service account      annular-haven-312209@appspot.gserviceaccount.com          False
export PROJECT=$(gcloud info --format='value(config.project)')
export SA_EMAIL=$(gcloud iam service-accounts list --filter="name:jenkins-gcp" \
  --format='value(email)')
gcloud projects add-iam-policy-binding --member serviceAccount:$SA_EMAIL \
  --role roles/compute.instanceAdmin $PROJECT
gcloud projects add-iam-policy-binding --member serviceAccount:$SA_EMAIL \
  --role roles/compute.networkAdmin $PROJECT
gcloud projects add-iam-policy-binding --member serviceAccount:$SA_EMAIL \
  --role roles/iam.serviceAccountUser $PROJECT
gcloud projects add-iam-policy-binding --member serviceAccount:$SA_EMAIL  \
   --role roles/container.admin $PROJECT
```

We then need to generate a set of credentials (keys) for the service account, and then store this as a credential in Jenkins. Generating the key file:
```
gcloud iam service-accounts keys create --iam-account $SA_EMAIL jenkins-gcp.json`
```
Then add the service account as a Jenkins credential using kind `Secret file`; this will require you to upload the file `jenkins-gcp.json` generated in the last step, and then give it a name and id (I called them both `jenkins-gcp`).

When running the pipeline in Jenkins, you will need to select this credential on the parameters page.

# Adding gcloud as a Jenkins tool

Jenkins tools are away of integrating external tools without needing to install them in the Jenkins server operating system (os); the power of this that you can typically define different versions of tools rather than being limited to a single version installed in the os.

Instructions for install:
* Visit `Manage Jenkins->Manage Plugins` and install the `Gcloud SDK plugin` and restart Jenkins. 
* Visit `Manage Jenkins->Global Tool Configuration` and browse to the `Google Cloud SDK` section. 
* Click on `Google Cloud Installations` then `Add Google Cloud SDK`.
* Set name to `Latest`.
* Tick install automatically.
* Click on `Add Installer` and select `Install from Google.com`.
* Save tool config.

# Running the pipeline

Create a Pipeline in Jenkins and point it at a git repo where this code is hosted. 

Specify `Jenkinsfile` to be used as the jenkinsfile and the correct branch/ref.

Do a dummy build of the pipeline to force populating the parameters settings. Just run the pipeline and don't allow anything to run (it will probably fail for first run).

You have the choice of `Create` or `Destroy` the cluster.

As mentioned above, a credential using a service account key (Secret file) is required to access your GCP account to do the deployment.

# Accessing the gke cluster from kubectl

`kubectl` is the kubernetes client. You should be able to install it via your package manager on your operating system.

Use the following to get your kube config (cluster name is the name specified via the Jenkins paramater; eg `demo`):

```
gcloud container clusters get-credentials demo --region europe-west4
```

Then confirm you can see everything deployed in the cluster:

```
kubectl get all -A
```

Check the nodes deployed:

```
kubectl get nodes
NAME                       STATUS    ROLES     AGE       VERSION
aks-nodepool1-26648723-0   Ready     agent     27m       v1.9.11
```
