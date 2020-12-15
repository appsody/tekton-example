## Development of Appsody as a standalone project has ended, but the core technologies of Appsody have been merged with odo to create odo 2.0! See our [blog post](https://appsody.dev/blogs/DevelopmentEnded) for more details!

# tekton-example
Example implementation of a Tekton pipeline that deploys an Appsody project.

## Introduction
This repo contains an example of a Tekton pipeline that builds and deploys an application created with [Appsody](https://github.com/appsody/appsody) to a Kubernetes cluster. The application is deployed via the [Appsody Operator](https://github.com/appsody/appsody-operator), which you will need to install prior to running the pipeline.

## Prerequisites and assumptions
In order to run this example, the following prerequisites are required:
1) You have access to a Kubernetes cluster that has the [Tekton pipelines installed](https://github.com/tektoncd/pipeline/blob/master/docs/install.md).
1) You have created an application using the appsody CLI, and your code is in a GitHub repository.
1) Your Kubernetes cluster can access a Docker registry, such as Docker Hub (it can pull and push images). You must have a [secret](https://kubernetes.io/docs/tasks/configure-pod-container/pull-image-private-registry/#registry-secret-existing-credentials) set up that contains valid credentials for authentication against your Docker registry.
1) You will need to install the appsody operator in the namespace your application will be deployed to. [Appsody Operator Install](https://appsody.dev/docs/using-appsody/building-and-deploying/#using-the-appsody-operator-commands)

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
    kubectl apply -f appsody-build-push-deploy.yaml
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
        value: https://github.com/chilanti/appsody-test-build    
        type: git
    ```
1) Once you have edited the resources, apply them to your cluster:
  ```
  kubectl apply -f appsody-pipeline-resources.yaml
  ```
  The Tekton pipeline is now fully set up.
### Openshift considerations
If you are targeting **Openshift**, ensure you perform the following task.
    
  #### Create a persistent volume
  You need to create a persistent volume (PV), so that the pipeline can obtain a persistent volume claim when it runs. We have included an example of creating a PV in the `okd-pv.yaml` manifest. You can create it by running:
  ```
  kubectl apply -f okd-pv.yaml
  ```

## A few words on the required deployment manifest
The pipeline is designed to deploy your application to the Kubernetes cluster using a deployment manifest. The build step generates a deployment manifest as part of the `appsody build` command and stores it in your project folder in the shared workspace. This deployment manifest named `app-deploy.yaml` is use to run `kubectl apply` to deploy your application.

Here we provide an example of such a deployment manifest:
```
apiVersion: appsody.dev/v1beta1
kind: AppsodyApplication
metadata:
  name: appsody-test-build
spec:
  # Add fields here
  version: 1.0.0
  applicationImage: appsody-test-build 
  stack: nodejs-express
  service:
    type: NodePort
    port: 3000
    annotations:
      prometheus.io/scrape: 'true'
  readinessProbe:
    failureThreshold: 12
    httpGet:
      path: /ready
      port: 3000
    initialDelaySeconds: 5
    periodSeconds: 2
    timeoutSeconds: 1
  livenessProbe:
    failureThreshold: 12
    httpGet:
      path: /live
      port: 3000
    initialDelaySeconds: 5
    periodSeconds: 2
  expose: true
```
The file can be located anywhere within your project, since the pipeline will discover it. 

Notice that the image url must match the definition of the Docker image resource that you created for the pipeline. The `containerPort` must be set to the port number on which the server inside the Appsody stack is configured to listen.

The file name can be modified by simply changing the relevant line in `appsody-build-pipeline.yaml`, as pointed out here:
```
      params:
      - name: appsody-deploy-file-name
        value: app-deploy.yaml
```
Also, if you wanted to retrieve a deployment manifest from a different repository, rather than assuming its presence in the application code repository, you could modify this section of `appsody-build-push-deploy.yaml`:
```
    - name: deploy-image
      image: kabanero/kabanero-utils
      command: ['/bin/sh']
      args: ['-c', 'cd /workspace/$gitsource && kubectl apply -f $(YAMLFILE)']
      env:
        - name: gitsource
          value: git-source
        - name: YAMLFILE
          value: $(inputs.params.app-deploy-file-name)
```
The implementation we have provided assumes the deployment manifest is in the `workspace/extracted` directory, which contains a clone of the source repository - but it could be adjusted to obtain that file from a different source. 

## Running the pipeline manually
The execution of a Tekton pipeline can be triggered automatically by a webhook that you can define on your GitHub project. However, that requires your Kubernetes cluster to be accessible on a public internet endpoint. For this reason, we provided a manual trigger (or PipelineRun resource) that you can use to kick off the pipeline on your cluster.

Run the following command:
```
kubectl apply -f appsody-pipeline-run.yaml
```
You will observe the pipeline being executed on your cluster.
