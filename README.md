# tekton-example
Example implementation of a Tekton pipeline that deploys an Appsody project.

## Introduction
This repo contains an example of a Tekton pipeline that builds and deploys an application created with [Appsody](https://github.com/appsody/appsody) to a Kubernetes cluster. The application is deployed as a Knative Serving service. 

## Prerequisites and assumptions
In order to run this example, the following prerequisites are required:
1) You have access to a Kubernetes cluster where Knative and its prerequisites are configured. You can read about setting up Knative [here](https://knative.dev/docs/install/).
1) Your Kubernetes cluster has the [Tekton pipelines installed](https://github.com/tektoncd/pipeline/blob/master/docs/install.md).
1) You have created an application using the appsody CLI, and your code is in a GitHub repository.
1) Your code repository includes a Knative Serving manifest file called `appsody-service.yaml`. We'll discuss this aspect in more detail later on.
1) Your Kubernetes cluster can access a Docker registry, such as Docker Hub (it can pull and push images). You must have a secret set up that contains valid credentials for authentication against your Docker registry.

This example and the artifacts that are included assume that you will be deploying the pipeline in the `default` namespace. If you wish to deploy it in your own namespace, you need to make the necessary changes (either append `-n <namespace>` to the `kubectl apply` commands, or edit the manifests to include a `namespace` definition).

## Setting up the pipeline
This repo contains the manifests for the resources that you need to create on your cluster in order to run the Tekton pipeline for Appsody.

1) Since the Tekton pipeline needs to deploy to the cluster itself, you want to ensure that it runs under an identity that has cluster administrator privileges.

    For this reason, create a service account and the appropriate cluster role binding by issuing:
    ```
    kubectl apply -f appsody-service-account.yaml
    kubectl apply -f appsody-cluster-role-binding.yaml
    ```
    The previous commands set up a service account called `appsody-sa` and grant the `cluster-admin` role to it. The pipeline you are going to create uses this service account. 

1) Make the Docker registry credentials available to the pipeline by adding your docker secret to the `appsody-sa` service account. This can be accomplished by editing the service account, using the following command:
    ```
    kubectl edit serviceaccount appsody-sa
    ```
    An editor opens up. Add your secret to the list of secrets, as shown in the example below: 
    ```
    ...
    secrets:
    - name: appsody-sa-token-ldzbq
    - name: my-docker-secret
    ```
    Save the changes. 

1) Now, create the pipeline task and the pipeline definition. We have a simple pipeline, with a single task that performs the various steps of building and deploying the project:
    ```
    kubectl apply -f appsody-build-task.yaml
    kubectl apply -f appsody-build-pipeline.yaml
    ```


1) The pipeline requires the definition of two resources in order to operate:
    * The definition of the Docker image that is built and deployed by the pipeline itself
    * The location of the GitHub project that contains your code

    For this reason, you need to edit the `appsody-pipeline-resources.yaml`. Change the value of the Docker image url to match your settings:
    ```
    ...
    spec:
      params:
      - name: url
        value: index.docker.io/chilantim/my-appsody-image
        type: image
    ```
    And change the definition of your GitHub project:
    ```
    ...
    spec:
      params:
      - name: revision
        value: master
      - name: url
        value: https://github.com/chilanti/appsody-test-build    type: git
    ```
1) Once you have edited the resources, apply them to your cluster:
  ```
  kubectl apply -f appsody-pipeline-resources.yaml
  ```
  The Tekton pipeline is now fully set up.
### Openshift considerations
If you are targeting **Openshift**, ensure you perform the following tasks.
    
  #### Make the Docker secret available to Kaniko
Edit the `appsody-build-task`, and set the path to the Docker config file for the Kaniko container. Issue the following command:
```
kubectl edit task appsody-build-task
```
and then add this environment variable to the `build-push-step` step:
```   
  env:
   - name: DOCKER_CONFIG
     value: /builder/home/.docker
``` 
The complete definition of the `build-push-step` should be as follows:
```
- name: build-push-step
    image: gcr.io/kaniko-project/executor
    command:
      - /kaniko/executor
    args:
      - --dockerfile=${inputs.params.pathToDockerFile}
      - --destination=${outputs.resources.docker-image.url}
      - --context=${inputs.params.pathToContext}
    env:
      - name: DOCKER_CONFIG
        value: /builder/home/.docker
```
  Note - this addition for Openshift is required for reasons explained in [this issue](https://github.com/appsody/tekton-example/issues/6).

  #### Allow Appsody to mount `docker.sock`
  After you create the `appsody-sa` service account, run the following command:
  ```
  oc adm policy add-scc-to-user hostmount-anyuid -z appsody-sa
  ```

  #### Create a persistent volume
  You need to create a persistent volume (PV), so that the pipeline can obtain a persistent volume claim when it runs. We have included an example of creating a PV in the `okd-pv.yaml` manifest. You can create it by running:
  ```
  kubectl apply -f okd-pv.yaml
  ```
## A few words on the required deployment manifest
As we mentioned earlier, the pipeline is designed to deploy your application to the Kubernetes cluster as a Knative Serving service. The pipeline expects a deployment manifest located within your project - specifically, it expects to run `kubectl apply` against a file named `appsody-service.yaml`. 

Here we provide an example of such a deployment manifest:
```
piVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: appsody-project
spec:
  runLatest:
    configuration:
      revisionTemplate:
        spec:
          container:
            image: mydockeraccount/appsody-project
            imagePullPolicy: Always
            ports:
            - containerPort: 3000

```
The file can be located anywhere within your project, since the pipeline will discover it. 

Notice that the image url must match the definition of the Docker image resource that you created for the pipeline. The `containerPort` must be set to the port number on which the server inside the Appsody stack is configured to listen.

One way to obtain a manifest file that has all the matching settings is to run the `appsody deploy` command, as described in [the Appsody documentation](https://appsody.dev/docs).

It must be noted, however, that the pipeline can work with any deployment manifest - not limited to Knative Serving services. Its current implementation applies whatever deployment manifest is contained in `appsody-service.yaml`. 

The file name can be modified by simply changing the relevant line in `appsody-build-pipeline.yaml`, as pointed out here:
```
      params:
      - name: appsody-deploy-file-name
        value: appsody-service.yaml
```
Also, if you wanted to retrieve a deployment manifest from a different repository, rather than assuming its presence in the application code repository, you could modify this section of `appsody-build-task.yaml`:
```
    - name: install-knative
      image: lachlanevenson/k8s-kubectl
      command: ['/bin/sh']
      args: ['-c', 'find /workspace/extracted -name ${YAMLFILE} -type f|xargs kubectl apply -f']
      env:
        - name: YAMLFILE
          value: ${inputs.params.appsody-deploy-file-name}
```
The implementation we have provided assumes the deployment manifest is in the `workspace\extracted` directory, which contains a clone of the source repository - but it could be adjusted to obtain that file from a different source. 

## Running the pipeline manually
The execution of a Tekton pipeline can be triggered automatically by a webhook that you can define on your GitHub project. However, that requires your Kubernetes cluster to be accessible on a public internet endpoint. For this reason, we provided a manual trigger (or PipelineRun resource) that you can use to kick off the pipeline on your cluster.

Run the following command:
```
kubectl apply -f appsody-pipeline-run.yaml
```
You will observe the pipeline being executed on your cluster.
