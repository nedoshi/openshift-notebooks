# How to deploy Jupyter Notebook #

Pre-req - You already have an OpenShift cluster. The following guides through a step by step procedure in deploying Jupyter Notebook in OpenShift.

### What will we do in this lab?
In this lab, you’ll go through a set of tasks that will help you understand the concepts of deploying and using containers.

Some of the things you’ll be going through:

- Create a minimal Jupyter notebook image using the Source-to-Image (S2I) build process. The image can be built in OpenShift, separately using the s2i tool, or using a docker build.
- Deploy a custom notebook image, you can use the s2i-minimal-notebook:3.6 image as an S2I builder

You’ll be doing the majority of the labs using the OpenShift CLI, but you can also accomplish them using the OpenShift web console.

The Jupyter Project provides a number of images for notebooks on Docker Hub. These are:

* base-notebook
* r-notebook
* minimal-notebook
* scipy-notebook
* tensorflow-notebook
* datascience-notebook
* pyspark-notebook
* all-spark-notebook

The GitHub repository used to create these is:

* https://github.com/jupyter/docker-stacks

Building the Minimal Notebook
-----------------------------

Instead of using the pre-built version of the minimal notebook, you can build the minimal notebook from source code. You may want to do this where you need it to use a RHEL base image included with your OpenShift cluster, instead of CentOS. Do be aware though that certain third party system packages may not be available for RHEL if you need to extend the image. One known example of this is image/video processing libraries, which although they may be able to be added to a CentOS base image, do not work with RHEL.

In order to build the minimal notebook image from source code in your OpenShift cluster use the command:

```
oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/jupyter-notebooks/master/build-configs/s2i-minimal-notebook.json
```

This will create a build configuration in your OpenShift project to build the minimal notebook image using the Python 3.6 S2I builder included with your OpenShift cluster. You can watch the progress of the build by running:

```
oc logs --follow bc/s2i-minimal-notebook-py36
```

A tagged image ``s2i-minimal-notebook:3.6`` should be created in your project. Since it uses the same image name as when loading the image using the image stream, referencing the image on quay.io, only do one or the other. Don't try to both load the image stream, and build the minimal notebook from source code.

Deploying the Minimal Notebook
------------------------------

To deploy the minimal notebook image run the following commands:

```
oc new-app s2i-minimal-notebook:3.6 --name minimal-notebook \
    --env JUPYTER_NOTEBOOK_PASSWORD=mypassword
```

The ``JUPYTER_NOTEBOOK_PASSWORD`` environment variable will allow you to access the notebook instance with a known password.

Deployment should be quick if you build the minimal notebook from source code. If you used the image stream, the first deployment may be slow as the image will need to be pulled down from quay.io. You can monitor progress of the deployment if necessary by running:

```
oc rollout status dc/minimal-notebook
```

Because the notebook instance is not exposed to the public network by default, you will need to expose it. To do this, and ensure that access is over a secure connection run:

```
oc create route edge minimal-notebook --service minimal-notebook \
    --insecure-policy Redirect
```

To see the hostname which is assigned to the notebook instance, run:

```
oc get route/minimal-notebook
```

Access the hostname shown using your browser and enter the password you used above.

To delete the notebook instance when done, run:

```
oc delete all --selector app=minimal-notebook
```

Creating Custom Notebook Images
-------------------------------

To create custom notebooks images, you can use the ``s2i-minimal-notebook:3.6`` image as an S2I builder. This repository contains two examples for extending the minimal notebook. These can be found in:

* [scipy-notebook](./scipy-notebook)
* [tensorflow-notebook](./tensorflow-notebook)

These are intended to mimic the images of the same name available from the Jupyter project.

In the directories you will find a ``requirements.txt`` file listing the additional Python packages that need to be installed from PyPi. You will also find a ``.s2i/bin/assemble`` script which will be triggered by the S2I build process, and which installs further packages and extensions.

To use the S2I build process to create a custom image, you can then run the command:

```
oc new-build --name custom-notebook \
  --image-stream s2i-minimal-notebook:3.6 \
  --code https://github.com/jupyter-on-openshift/jupyter-notebooks \
  --context-dir scipy-notebook
```

If any build of a custom image fails because the default memory limit on builds in your OpenShift cluster is too small, you can increase the limit by running:

```
oc patch bc/custom-notebook \
  --patch '{"spec":{"resources":{"limits":{"memory":"1Gi"}}}}'
```

and start a new build by running:

```
oc start-build bc/custom-notebook
```

If using the custom notebook image with JupyterHub running in OpenShift, you may also need to set the image lookup policy on the image stream created.

```
oc set image-lookup is/custom-notebook
```

This is necessary so that the image stream reference in the pod definition created by JupyterHub will be able to resolve the name to that of the image stream.

For the ``scipy-notebook`` and ``tensorflow-notebook`` examples provided, if you wish to use the images, instead of running the above commands, after you have loaded the image stream for, or built the minimal notebook image, you can instead run the commands:

```
oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/jupyter-notebooks/master/build-configs/s2i-scipy-notebook.json
oc create -f https://raw.githubusercontent.com/jupyter-on-openshift/jupyter-notebooks/master/build-configs/s2i-tensorflow-notebook.json
```

When creating a custom notebook image, the directory in the Git repository the S2I build is run against can contain a ``requirements.txt`` file listing the Python package to be installed in the custom notebook image. Any other files in the directory will also be copied into the image. When the notebook instance is started from the image, those files will then be present in your workspace.
