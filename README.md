# drone-demo
Drone is a Continuous Integration (CI) that we can use to build and push docker images ideally for use in a Kubernetes cluster. This project details getting some basic pipelines up and running, with the goal of pushing an image to a container registry. This should hopefully evolve over time as features are figured out.

## Pre-requisites
- As we are intending to use this in conjunction with Argo, then it is expected Argo has been set up too as per https://github.com/nimbleapproach/argo-demo, this should also ensure various things we need are also set up.
- Familiarity with kubectl
- You have configured Argo to access a repo where we'll put the drone deployment, will be referred to as *git@github.com:youruser/yourrepo.git* in these docs

## Guide
### Create OAuth application in GitHub
This will allow Drone to connect to our repositories in order to run pipelines, it will additionally set up webhooks so that pipelines can be automatically started depending on actions taken.
- OAuth apps are added at the account level
- In the account settings go to Developer Settings -> OAuth Apps -> New OAuth App
- Add a name e.g. "Drone Server"
- Add homepage URL e.g. http://nimblepaultestdrone.uksouth.cloudapp.azure.com
- Add callback URL e.g. http://nimblepaultestdrone.uksouth.cloudapp.azure.com/login
- **N.B. I tested this with http, https may be preferred it *should* work**
- Register the Application
- Note down the client ID, and generate a client secret, note that down too.

### Create test repo to build from
Copy the files from the "drone app" folder in this repository to a repository of your choice.
This is just a basic helloworld app with a docker build and an arbitrary test, we will need one quick modification though
- In .drone.yml change the ACR registry to whichever you are using.

### Create drone namespace
Create a namespace in kubernetes where our drone service will live.
- Assuming we've previously retrieved the config for kubectl (see **Install kubectl** in the Argo [README](https://github.com/nimbleapproach/argo-demo/blob/main/README.md))
- `kubectl create namespace drone`

### Create drone secrets in kubernetes
We'll use kubernetes secrets to store sensitive information (like the OAuth client info above) other methods are available, but this is pretty straightfroward.
- Before we create the secrets we need an additional secret which will be the key shared between the drone server and the drone runners, we can generate a key using `openssl rand -hex 16` on the command line, this will be our RPC secret, note it down
- `kubectl create secret generic drone-secrets --namespace=drone --from-literal=githubclientid=? --from-literal=githubclientsecret=? --from-literal=rpcsecret=?`
- Replace the question marks with the respective keys

### Install the drone server
- Copy the files in the *server* directory of this repo to a similar directory in th repo you are serving your installation from 
- There are a couple of entries in the deployment yaml you will likely want to change:
- *name: "DRONE_USER_CREATE" value: "username:paulpbrandon,admin:true"* Adds admin rights to the given user, which will give access to a few extra things, change the username as you wish, it will match your github account.
- *name: "DRONE_SERVER_HOST" value: "nimblepaultestdrone.uksouth.cloudapp.azure.com"* the host name for your drone server, change this too.
- In the service.yaml I have used the annotation *service.beta.kubernetes.io/azure-dns-label-name: nimblepaultestdrone* which is what generates the azure host name above, this is easiest for testing, but a proper name is likely to be used in the future
- Install the directory to Argo, assuming the argo-cli is installed you can use the following command:
- `argocd app create droneserver --repo git@github.com:youruser/yourrepo.git --path server --dest-server https://kubernetes.default.svc --dest-namespace drone`
- Or use the UI
- **N.B. Under no circumstances call the drone server *drone-server* it will mess with the environment variables and prevent the server from starting**

### Install drone runner(s)
The drone server will delegate building tasks to an available runner, so we will need to create at least one
- Copy the files in the *runner* directory of this repo to a similar directory in th repo you are serving your installation from 
- Change the *name: DRONE_RPC_HOST value: "nimblepaultestdrone.uksouth.cloudapp.azure.com"* entry accordingly in the deployment yaml
- Adjust the replicas entry if you want more the one
- Install the directory to Argo, assuming the argo-cli is installed you can use the following command:
- `argocd app create dronerunner --repo git@github.com:youruser/yourrepo.git --path runner --dest-server https://kubernetes.default.svc --dest-namespace drone`
- Or use the UI

### Create Service Principle for ACR
Do this to allow drone to log into your container registry and be able to push to it (change the name as you see fit).
- `az acr show --name paulsregistry80 --query "id" --output tsv` (to get the full id of the registry)
- `az ad sp create-for-rbac --name paulsregistryprinciple --scopes *outputfromabove* --role acrpush --query "password" --output tsv` (note this will return your generated password, so note it)
- `az ad sp list --display-name paulsregistryprinciple --query "[].appId" --output tsv` will show you the user id, note this too.
- The username and password will be required in the step below

### Set up a pipeline
- Assuming the server deployed successfully, navigate to the drone homepage e.g. https://nimblepaultestdrone.uksouth.cloudapp.azure.com
- Your should see a page with **You will be redirected to your source control management system to authenticate.** and a continue button.
- Click on continue, drone should authenticate against GitHub (you will see activity in the OAuth Apps page) and eventually you should see a list of repositories. Hopefully, you should see our test repository we created earlier, select it and press the *Activate* button. This should then create a webhook against that repository in GitHub (under settings for the repository)
- In Drone itself you will be taken to a settings page, as we'll be deploying to ACR the pipeline has been configured to get credentials from secrets.  So on the settings page select *Secrets* (The first link is secrets at the repo level, the second at the account level, have tested with the first but should work with the second and not have to keep reentering for all repos)
- Add entry for acr_pass (password from the step above)
- Add entry for acr_user (User ID from the step above)

*We may be able to get the secrets from elsewhere but not sure how at this point)*

### Build a pipeline 
- Within Drone, we can select the repo we want to build and select *New Build*, which naturally should start executing a new build
- Commiting or tagging in the repo should also trigger a build via the webhook, if this doesn't work it may come down to whether you're using https and if it is enabled, you can try it out with http instead)
- The behavour currently defined in the drone file is this:
1. Run test on any event to any branch (commit etc.)
2. If the event was a tag event run docker build and push the image to ACR (note it uses auto_tag which will generate the tag from the github tag)

*We'll likely want to look at different pipeline for different branches, environments and so on. I was interested in using GitHub environments to set deployment targets, however it requires an Enterprise account*

To address the above point, there is a second version of the helloworld app within the *kustomizeApp* folder in this repository that you may wish to try out. The pipeline in this file has steps for tag and promote, which will equate to dev and production for the purposes of this example. Additionally, the pipeline will update the the service definition associated with the application for Argo to then pick up and deploy. Steps as follows:
1. Run test on any event to any branch (commit etc.)
2. If the event was a tag event run docker build and push the image to ACR with SNAPSHOT suffix
3. If event was tag, update the image used by the dev deployment to the one we just built
4. If event was promote (done within drone), run docker build and push the image to ACR with github tag
5. If event was promote, update the image used by the prod deployment to the one we just built

## Things to look at
### TLS certificates
The is an error in the server logs `2022/07/15 21:21:05 http: TLS handshake error from 10.244.0.1:33125: EOF` though it currently does not appear to be affecting anything, I believe this might be to do with using the self cert option when setting up the server with https, and we should look at using the certs with the kubernetes cluster.

### Back drone with postgres
By default drone spins up an embedded SqlLite database which it mounts to /data. However, it does support adding proper databases. This may be beneficial for sorting out problems with runners, as the data will be readily accessible, it is also an alternative to persisting the data via the volume claim.

### Autoscaling
We would like to spin runners up and down depending on load. Drone has a Prometheus compatible metrics endpoint (https://docs.drone.io/server/metrics/) that has things like number of pending jobs. Ideally we could use these metrics in conjunction with autoscaling features within Kubernetes. First we would need to expose the metrics to Kubernetes. Apparently Azure has a Prometheus integration that does not require you to set it up, you just have to add a ConfigMap as per https://trstringer.com/collect-custom-aks-metrics/ (assuming monitoring is enabled). However, the metrics endpoint requires a token and I haven't found a way to add it via this route (yet, it seems to suggest needing to set tls config but not sure where). I tried adding the token as an access_token query param, which works in the browser, but remains unauthenticated when the metrics are collected. The logs for this are in the omsagent, example command `kubectl logs omsagent-rs-6c7ddbdf7b-jwtnh -n kube-system`. A fall back might be to add a proxy app that makes the call to metrics.
