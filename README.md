# drone-demo
Drone is a Continuous Integration (CI) that we can use to build and push docker images ideally for use in a Kubernetes cluster. This project details getting some basic pipelines up and running, with the goal of pushing an image to a container registry. This should hopefully evolve over time as features are figured out.

## Pre-requisites
- As we are intending to use this in conjunction with Argo, then it is expected Argo has been set up too as per https://github.com/nimbleapproach/argo-demo, this should also ensure various things we need are also set up.
- Familiarity with kubectl

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

### Create drone namespace
Create a namespace in kubernetes where our drone service will live.
- Assuming we've previously retrieved the config for kubectl (see **Install kubectl** in the Argo [README](https://github.com/nimbleapproach/argo-demo/blob/main/README.md)
- `kubectl create drone`


### Create drone secrets in kubernetes
We'll use kubernetes secrets to store sensitive information (like the OAuth client info above) other methods are available, but this is pretty straightfroward.
- Before we create the secrets we need an additional secret which will be the key shared between the drone server and the drone runners, we can generate a key using `openssl rand -hex 16` on the command line, this will be our RPC secret, note it down




## Things to look at
### TLS certificates
The is an error in the server logs `2022/07/15 21:21:05 http: TLS handshake error from 10.244.0.1:33125: EOF` though it currently does not appear to be affecting anything, I believe this might be to do with using the self cert option when setting up the server with https, and we should look at using the certs with the kubernetes cluster.

### Autoscaling
We would like to spin runners up and down depending on load. Drone has a Prometheus compatible metrics endpoint (https://docs.drone.io/server/metrics/) that has things like number of pending jobs. Ideally we could use these metrics in conjunction with autoscaling features within Kubernetes. First we would need to expose the metrics to Kubernetes. Apparently Azure has a Prometheus integration that does not require you to set it up, you just have to add a ConfigMap as per https://trstringer.com/collect-custom-aks-metrics/ (assuming monitoring is enabled). However, the metrics endpoint requires a token and I haven't found a way to add it via this route (yet, it seems to suggest needing to set tls config but not sure where). I tried adding the token as an access_token query param, which works in the browser, but remains unauthenticated when the metrics are collected. The logs for this are in the omsagent, example command `kubectl logs omsagent-rs-6c7ddbdf7b-jwtnh -n kube-system`. A fall back might be to add a proxy app that makes the call to metrics.
