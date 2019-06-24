# tekton-example
Example implementation of a Tekton pipeline that deploys an Appsody project.

## Introduction
This repo contains an example of a Tekton pipeline that builds and deploys an application created with [Appsody](https://github.com/appsody/appsody) to a Kubernetes cluster. The application is deployed as a Knative Serving service. 

## Prerequisites and assumptions
This example can be put to fruition once you have fulfilled the following prerequisites:
1) You have access to a Kubernetes cluster where Knative and its prerequisites are configured. You can read about setting up Knative [here](https://knative.dev/docs/install/).
2) Your Kubernetes cluster has the [Tekton pipelines installed](https://github.com/tektoncd/pipeline/blob/master/docs/install.md).
3) You have created an application using the appsody CLI, and have checked in your code in a GitHub repository.
4) Your code repository includes a Knative Serving manifest file called `appsody-service.yaml`. We'll discuss this aspect more in detail later on.
5) Your Kubernetes cluster can access Docker Hub (it can pull and push images).

## Setting up the pipeline
This repo contains the manifests for the resources that you need to create on your cluster in order to run the Tekton pipeline for Appsody.

1) Since the Tekton pipeline needs to deploy to the cluster itself, you want to ensure that it runs under an identity that has cluster administrator privileges. 

For this reason, create a service account and the appropriate cluster role binding by issuing:
```
kubectl apply -f appsody-service-account.yaml
kubectl apply -f appsody-cluster-role-binding.yaml
```

2) Now, create the pipeline task and the pipeline definition. We have here a simple pipeline, with just a single task that performs the various steps of building and deploying the project:
```
kubectl apply -f appsody-build-task.yaml
kubectl apply -f appsody-build-pipeline.yaml
```
3) The pipeline requires the definition of two resources in order to operate:
* The definition docker image that is built and deployed by the pipeline itself
* The location of the GitHub project that contains your code

For this reason, you need to edit the `appsody-pipeline-resource.yaml`. Change the value of the Docker image url to match your settings:
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
4) Once you have edited the resources, apply them to your cluster:
```
kubectl apply -f appsody-pipeline-resources.yaml
```
The Tekton pipeline is now fully set up.

## A few words on the required deployment manifest
<TBD>

## Running the pipeline manually
The execution of a Tekton pipeline can be triggered automatically by a webhook that you can define on your GitHub project. However, that requires your Kubernetes cluster to be accessible on a public internet endpoint. For this reason, we provided a manual trigger (or PipelineRun resource) that you can use to kick off the pipeline on your cluster.

Run the following command:
```
kubectl apply -f appsody-pipeline-run.yaml
```
You will observe the pipeline being executed on your cluster.


