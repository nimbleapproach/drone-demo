# Kustomized Helloworld Application

This version of the helloworld app is to demonstrate using drone (and Argo) to handle deployments to different environments.

As such this app uses an environment variable to write out a message, just so that we can prove that different environments are using different images and so on.

Secondly, this app contains a pipeline definition that will (at a high level):
1. Run test on any event to any branch (commit etc.), except on promote
2. If the event was a tag event run docker build and push the image to ACR with the same tag as the github tag
3. If event was tag, update the image used by the dev deployment to the one we just built
4. If event was promote, update the image used by the prod deployment to the one built in the tag step.

The pipeline updates the Argo deployment by cloning the Argo git repository, this will work by virtue of Drone being an OAuth app with GitHub, but it does mean that the application repo and the argo repo must be visible to the same account.

Once the argo repo is cloned it will update the relevant deployment file with the tag (using sed) for the image that has been built and then commit the change back.

The steps *gen_build_env_dev* and *gen_build_env_prod* act as a switch for setting which file and which replacement will be performed by the sed command depending on whether the event is tag or promote. There is also a *gen_build_env_base* step which applies to both, in this case it simply sets the image tag to use.

Once the change has been committed, Argo should detect the change, and if set to Auto-Sync it will deploy the change to Kubernetes for the affected deployment.

## Pipeline Template
Note this pipeline now uses a templated pipeline, the template is stored within the template directory of this repository.

To use a template, it must be added to drone, this can be done via the Drone UI, or assuming you have the drone [CLI](https://docs.drone.io/cli/install/) installed you can run:
- `drone template add --namespace nimbleapproach --name node_baseline.yaml --data @template/node_baseline.yaml`
- Note the name space usually matches the GitHub account you're using

The template contains a set of placeholders that will be provided from the *.drone.yml* file associated with you application.

This template requires the following parameters, most should be fairly self-explanatory:
- **argo_repo:** The repository where the Kubernetes resource files live, that Argo will be using to make deployments
- **deployment_file_dev:** The deployment (or Kustomize, or other) file that defines the image for the dev environment, in the argo repo
- **deployment_file_prod:** The deployment file that defines the image for the prod environment, in the argo repo
- **docker_registry:** The container registry where the docker container will live 
- **docker_container:** The name for the docker container

### Additional Notes
This pipeline uses the Organization secrets, docker_user and docker_pass to connect to ACR, which can be set via the UI or the drone CLI with:
- `drone orgsecret add nimbleapproach docker_user abcd`
*TODO look at using encrypted or external secrets*
- If cloning this app it will be a good idea to change the container name and set the versions to 0 in the deployment yaml, otherwise the current pipeline will fail when it finds no changes to commit, though it will actually work as the image required will now be present
