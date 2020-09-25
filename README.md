# Running COBOL on OpenShift on IBM Z

In this code pattern, we will build a container with a simple hello world COBOL application using OpenShift, Buildah, and Podman. We will build a container locally, push it to the internal OpenShift on IBM Z registry, then run create a batch job that runs the COBOL application in a pod.

When you have completed this code pattern, you will understand how to:

* Build containers locally
* Push a container to the OpenShift internal registry
* Run the container as a pod in the OpenShift on IBM Z cluster

# Steps

1. [Clone the repository locally](#1-clone-the-repository-locally) 
2. [Create your project](#2-create-your-project)
3. [Build your COBOL container](#3-build-your-cobol-container)
4. [Push your COBOL container to the OpenShift on IBM Z internal registry](#4-push-your-cobol-container-to-the-openshift-on-ibm-z-internal-registry)
5. [Run a batch job on OpenShift on IBM Z](#5-Run-a-batch-job-on-OpenShift-on-IBM-Z)
6. [(Optional) Use a shell script to do the full installation](#6-use-a-shell-script-to-do-the-full-installation)

**NOTE**: If you simply want to get the COBOL application up and running on OpenShift on IBM Z quickly without going through each step manually, a shell script is included in this repository to do it for you. For directions on how to use it, skip to Step 6

### 1. Clone the repository locally

Clone down this `git` repository on your local server and change into the new directory.

```bash
$ git clone https://github.com/mmondics/kubernetes-cobol
Cloning into 'kubernetes-cobol'...
remote: Enumerating objects: 85, done.
remote: Counting objects: 100% (85/85), done.
remote: Compressing objects: 100% (76/76), done.
remote: Total 142 (delta 43), reused 6 (delta 2), pack-reused 57
Receiving objects: 100% (142/142), 41.37 KiB | 20.68 MiB/s, done.
Resolving deltas: 100% (65/65), done.

$ cd kubernetes-cobol/
```

### 2. Create your project

Next create a `project` in Red Hat OpenShift to store your container images. Log into OpenShift via the CLI, then run the following commands. 

**NOTE**: `<BRACKETS>` denote a variable that you should edit for your environment. Wherever you see text in `<BRACKETS>`, replace the text within the brackets and remove the brackets themselves e.g. `<OPENSHIFT_PROJECT>` will become something like `cobol-project`. 

```bash
$ oc login --server=<OPENSHIFT_API_URL> # replace with your OpenShift cluster API address. You can find it on the overview page in the OpenShift console.
$ oc new-project <OPENSHIFT_PROJECT>

```

### 3. Build your COBOL container

Build the container on your local workstation. First change into the `openshift/` directory. We are going to use the `project` that we created above, call the container `openshift-cobol`, and tag it as `v1.0.0`.

 **NOTE**: You will need your OpenShift Image Registry URL. You can find this by running `oc get routes -n openshift-image-registry`, or in the OpenShift console by navigating to Administrator -> Networking -> Routes and ensuring you're in the openshift-image-registry project.
```bash
$ cd openshift/
$ oc login <OPENSHIFT_API_URL> \
    --username=<OPENSHIFT_USERNAME> \
    --password=<OPENSHIFT_PASSWORD> \ # if you do not want to enter your password in plaintext, omit this flag and you will be prompted for your password and not displayed. 
    --insecure-skip-tls-verify=true
$ podman login \
    --username <OPENSHIFT_USERNAME> \
    --password $(oc whoami -t) \
    --tls-verify=false \
    <OPENSHIFT_REGISTRY_URL>
$ oc project <OPENSHIFT_PROJECT> # double check you're in the correct project
$ buildah build-using-dockerfile \
  -t <OPENSHIFT_REGISTRY_URL>/<OPENSHIFT_PROJECT>/openshift-cobol:v1.0.0 . # make sure to include the period at the end of the command. 
```

**NOTE**: The `buildah build-using-dockerfile` command above can take 2-10 minutes to complete, depending on your OpenShift cluster. The Dockerfile is downloading and extracting Open COBOL and compiling the `helloworld.cobol` package. 

Check that your container image has been built and tagged using the `podman images` command. You should see two images returned - the one you created, named `openshift-cobol` and another from docker.io named `clefos`.

```bash
$ podman images
REPOSITORY                                                                               TAG      IMAGE ID       CREATED         SIZE
<OPENSHIFT_REGISTRY_URL>/<OPENSHIFT_PROJECT>/openshift-cobol                             v1.0.0   4ca7207d6010   7 minutes ago   682 MB
docker.io/s390x/clefos                                                                   latest   865aa764e034   4 months ago    174 MB

```

### 4. Push your COBOL container to the OpenShift on IBM Z internal registry

Now you want to push the container image you just built into the internal OpenShift registry from where it can be deployed onto the cluster. You will use the `podman push` command to do so. 

```bash
$ podman push --tls-verify=false \
  <OPENSHIFT_REGISTRY_URL>/<OPENSHIFT_PROJECT>/openshift-cobol:v1.0.0
```
Check that the image was successfully pushed to the OpenShift registry by running the command `oc get is` to see the new ImageStream in your project.
```bash
$ oc get is
NAME              IMAGE REPOSITORY                                                                         TAGS     UPDATED
openshift-cobol   <OPENSHIFT_REGISTRY_URL>/<OPENSHIFT_PROJECT>/openshift-cobol                             v1.0.0   2 minutes ago
```

### 5. Run a batch job on OpenShift on IBM Z

Next change into the `job` directory, and open up the `job.yml`. You should see the `<OPENSHIFT_PROJECT>` and you need to edit it to your project name. After this you can create your batch job.

```bash
$ cd job/
$ vi job.yml   # replace <OPENSHIFT_PROJECT> with your project name
$ oc apply -f job.yml
job.batch/openshift-cobol-job created
```

This batch job has one goal - run a pod consisting of a container created from the COBOL image you built and  pushed to the OpenShift registry. 

Verify that your job was created using `oc get jobs`, then view the pod that the job created using `oc get pods`. 

```bash
$ oc get job
NAME                  COMPLETIONS   DURATION   AGE
openshift-cobol-job   1/1           16s        18s

$ oc get pods
NAME                                  READY     STATUS      RESTARTS   AGE
openshift-cobol-job-tcgrk             0/1       Completed   0          2m21s
```

Your batch job successfully created a pod from your container image that was compiled from the `helloworld.cobol` file during the build process. To see the Hello World! message from your containerized COBOL code, run the command `oc logs pod/<POD_NAME>`

**NOTE**: Pod names are randomly generated, so make sure to insert your own pod name. It will be in the form of `openshift-cobol-job-NNNNN` where the last 5 digits will be unique to your own deployment. 

```bash
$ oc logs pod/<POD_NAME>
Hello World!
```

Congratulations! You have just built a Hello World container image from COBOL, pushed it into an OpenShift on IBM Z cluster, and deployed the application using a batch job to create your pod. 

### 6. Use a shell script to do the full installation

**NOTE**: You do not need to complete this step if you completed steps 1-5. This is a simplified alternative set of directions for those who do not wish to complete the previous steps. 

A shell script is provided in this repository to do everything in steps 1-5, from logging in to OpenShift to building and pushing the COBOL container image, to deploying it in the OpenShift cluster using a batch job. 

Before running the shell script, you will need to edit the `env` and `job.yml` files to reflect your own OpenShift cluster. 

In the `/openshift` directory, edit the `env` file, changing the values for the OpenShift API URL, the route for your OpenShift internal registry, your OpenShift credentials, and the name of the project you wish to deploy the COBOL application into. 

In the `/openshift/job` directory, edit the `job.yml` file, changing the value for your OpenShift project. 

Back in the `/openshift` directory, run the shell script using the command `/.full-install.sh`. 

Once the script completes, you can run `oc get pods` to get your pod name, and then run `oc logs pod/<POD_NAME>` to return your Hello World! message. Refer to step 5 for more information. 

<!-- keep this -->
## License

This code pattern is licensed under the Apache License, Version 2. Separate third-party code objects invoked within this code pattern are licensed by their respective providers pursuant to their own separate licenses. Contributions are subject to the [Developer Certificate of Origin, Version 1.1](https://developercertificate.org/) and the [Apache License, Version 2](https://www.apache.org/licenses/LICENSE-2.0.txt).

[Apache License FAQ](https://www.apache.org/foundation/license-faq.html#WhatDoesItMEAN)
