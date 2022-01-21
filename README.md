# Kubeflow and the MPI Operator on OpenShift

This repository provides an example of using
[Kubeflow](https://www.kubeflow.org/) and its [MPI
operator](https://github.com/kubeflow/mpi-operator) on top of
[OpenShift](https://www.redhat.com/en/technologies/cloud-computing/openshift).
The examples specifically target running [OpenFOAM](https://openfoam.org/) CFD
simulations with CPUs and GPUs. It uses [OpenShift Data
Foundation](https://www.redhat.com/en/technologies/cloud-computing/openshift-data-foundation)
for providing RWX storage.

## Background
The growing adoption of Kubernetes provides a new opportunity to shed legacy HPC
infrastructures. Kubernetes is effectively a general purpose scheduling system
for containers. As many MPI-based workloads are already written on Linux, they
can be easily containerized. The Kubeflow project has an early-stage operator
that handles MPI applications. 

OpenFOAM is an application suite used for computational fluid dynamics (CFD)
analysis. It is capable of processing large jobs in parallel using MPI. These
jobs frequently involve large numbers of processors (CPUs). For an organization
with a large Kubernetes cluster at its disposal, making use of these large
processor pools to perform MPI jobs in their spare time seems logical. Further,
in cloud-based environments like AWS (where this example is designed to run),
organizations can make use of the autoscaling features inside Kubernetes to
simply create the capacity required to fulfill the MPI job's requirements.

Deeper descriptions of MPI, workers, processors, and etc. is outside of the
scope of this example. Knowledge of CFD and OpenFOAM is also outside of the
scope of this example. Some understanding of both MPI and OpenFOAM is assumed,
but not required. A solid grasp of Kubernetes and, to a degree, OpenShift, is
assumed.

## Base Requirements and Prerequisites
This example was constructed on an OpenShift 4.9 cluster using the Kubeflow MPI
Operator version 0.3. It also makes use of an OpenShift Data Foundation (ODF)
CephFS deployment to provide the RWX storage necessary for each of the MPI
workers to access the OpenFOAM data. 

OpenFOAM v9 was used along with some of its parallel processing tutorial
examples. Additionally, a more complicated real-world example of aerodynamic CFD
analysis of vehicles was graciously provided by Morlind Engineering. As of the
creation of this example, ODF does not have any non-replicated mode and, as
such, requires at least 3 OpenShift (Kubernetes) nodes to be able to be properly
installed.

The following subsections detail the steps to prepare to run the `damBreak`
example.

### OpenShift
OpenShift 4.9 was installed using the [Installer-Provisioned Infrastructure
(IPI)](https://docs.openshift.com/container-platform/4.9/installing/installing_aws/installing-aws-default.html)
method against an Amazon Web Services (AWS) environment using `m5a.4xlarge`
instance types for the OpenShift worker nodes.

It is assumed that you will also grab the `oc` command-line client, and that you
will be logged into your cluster as some user with `cluster-admin` privileges.

### OpenShift Data Foundation
OpenShift Data Foundation was installed via the OperatorHub within OpenShift as
a user with `cluster-admin` privileges:

* Click _Operators_ -> _OperatorHub_ in the left-hand navigation of the
  _Administrator_ perspective.
* Find the _OpenShift Data Foundation_ operator tile and click it.
* Click _Install_
* Leave all of the default options and click _Install_ again.

The Operator provides the capability for instaling the ODF solution. If you want
to learn more about Operators, you can learn more from the [Operator
Pattern](https://kubernetes.io/docs/concepts/extend-kubernetes/operator/)
documentation.

Once the Operator is installed:

* Click on _Operators_ -> _Installed Operators_ in the left-hand navigation of
  the _Administrator_ perspective.
* Find the `openshift-storage` project at the top in the drop-down menu.
* Click on the _OpenShift Data Foundation_ Operator.
* Click on the blue _Create StorageSystem_ button.
* Click _Next_ on the first screen.
* Select at least 3 nodes and click _Next_ on the second screen.
* Click _Next_ on the third screen.
* Finally, click _Create StorageSystem_ on the last screen.

Occasionally you may get a 404 error in the OpenShift web console if you have
not sufficiently refreshed the page (eg: Ctrl+F5) and certain content is either
cached or not cached. Go ahead and do a hard browser refresh at this time.

* Click on _Storage_ -> _OpenShift Data Foundation_ in the left-hand navigation of the _Administrator_ perspective.
* Click on the _Storage Systems_ tab.

You now will want to wait until the _StorageSystem_ status reports as ready. The
ODF operator is currently provisioning all of the resources it needs to be able
to provide a CephFS cluster on top of the `gp2` storage volumes (which
themselves are on top of AWS EC2 EBS).

### Kubeflow MPI Operator
The Kubeflow MPI Operator needs to be installed. The installation process also
creates a `CustomResourceDefinition` for an `mpijob` object, which is how you
will define the MPI job that you want the cluster to run.

You will want to clone the [MPI Operator
repository](https://github.com/kubeflow/mpi-operator.git) somewhere. From the
MPI Operator repository clone folder:

* `git checkout v0.3.0`
* `oc create -f deploy/v2beta1/mpi-operator.yaml`

This will create a namespace with all of the required elements to deploy the
operator. Wait for the pod for the MPI Operator to be deployed and ready before
continuing.

## MPI Job
The following sections detail getting the MPI data into the cluster and then
running the example MPI job. It is recommended that you deploy the following
assets into their own namespace (Project, in OpenShift parlance). For this
example we will refer to the `cfd` Project.

### CFD Project
Create a new Project in OpenShift called `cfd`. You can do this using the `oc`
CLI or the web console.

### Security Context Constraints (SCC)
OpenShift layers additional security features and defaults on top of vanilla
Kubernetes. One of these things is SCCs. You can learn about [Security Context
Constraints
here](https://docs.openshift.com/container-platform/4.9/authentication/managing-security-context-constraints.html).
By default, OpenShift does not allow containers to run as specific users/UIDs,
and it randomizes them. While OpenSSH (for MPI) and OpenFOAM can be made to work
with completely randomized UIDs, it's a lot of effort, and, for this example, it
was decided to relax the SCC defaults to allow `AnyUID`:

    oc adm policy add-scc-to-user anyuid -z default -n cfd

The above command allows the `default` ServiceAccount to use the `anyuid` SCC
when it deploys Pods. This means that our OpenFOAM pod, which wants to be user
`98765`, can be.

### Persistent Volume Claim
You will need some storage to attach the file manager and the CFD workers to. Be
sure to create the following file in the Project you created:

    oc create -f manifests/filemanager-pvc.yaml -n cfd

This PVC assumes that you used the default storage class names when you deployed
OpenShift Data Foundation.

Check the status of the PVC to make sure that it is successfully bound.

### Tiny File Manager
There are supporting manifests in the `manifests` folder for deploying a
PHP-based file manager program, [Tiny File
Manager](https://tinyfilemanager.github.io/), and a corresponding volume claim.
The `PersistentVolumeClaim` will grab a small amount of RWX storage, and you can
then upload the `damBreak` example from the `examples` folder in order to do
your pre-processing.

A [separate
repository](https://github.com/OpenShiftDemos/tinyfilemanager-php-ubi) has a
`Containerfile` which can be used to build the Tiny File Manager into a RHEL8
UBI-based Apache and PHP S2I image. Although Source-to-Image (S2I) was not used
to build the resulting container, the RHEL8 UBI Apache S2I image already has the
proper security modifications in order to easily be used in an OpenShift
environment. You can find more information about the [S2I image
here](https://github.com/sclorg/s2i-php-container). The image is also hosted on
Quay.io:
[https://quay.io/repository/openshiftdemos/tinyfilemanager-php-ubi](https://quay.io/repository/openshiftdemos/tinyfilemanager-php-ubi)

You can deploy the file manager with the following:

    oc create -f manifests/filemanager-assets.yaml -n cfd

This will also create a Service and a Route so that you can access the file
manager outside the cluster. Check the URL for the Route. Note that HTTPS is
_not_ enabled for this route. The effective URL to use is (make sure to check
your own base FQDN):

    http://filemanager-cfd.apps.cluster-fr5bx.fr5bx.sandbox1255.opentlc.com/tinyfilemanager.php

The default username and password for Tiny File Manager is used:

    admin / admin@123

* In the Tiny File Manager, click into the `storage` folder
* Click the `Upload` button at the top right.
* Drag the local `damBreak` folder (from this repository)
  into the file manager window to upload it recursively
  
That's it!

### OpenFOAM MPI Worker Container Image
Podman was used locally to build the OpenFOAM container image to go with the MPI
operator. You can find its `Containerfile` and supporting files in this
repository. The image is also currently being hosted on Quay.io:
[https://quay.io/repository/openshiftdemos/kubeflow-mpi-openfoam](https://quay.io/repository/openshiftdemos/kubeflow-mpi-openfoam)

OpenFOAM's CFD analysis process involves several steps. You can actually perform
some of the pre-processing steps inside a running container using `rsh` or
`exec`. First, deploy a Pod based on the OpenFOAM image that simply sleeps and
does nothing. This pod will also mount the same PersistentVolumeClaim so that it
can access the `damBreak` example.

    oc create -f manifests/foam-processor-pod.yaml -n cfd

Wait for the Pod to get running and then:

    oc rsh -n cfd foam-processor

You will get an `sh` prompt, at which point you want to run:

    $ bash

You should see something like:

    openfoam@foam-processor:~$

OpenFOAM includes a large quantity of `bash` functions, variables, paths,
libraries, and etc. It's important to start a `bash` session in order to pull
those things in. If you wanted to do the above two-step process in one step, you
could have done:

    oc exec -n cfd -it foam-processor -- bash

If you look at the directory structure inside the pod:

    ls -al

You will see something like the following:

    total 16
    drwxr-xr-x. 1 openfoam openfoam   21 Jan 21 16:47 .
    drwxr-xr-x. 1 root     root       22 Dec 16 12:51 ..
    -rw-r--r--. 1 openfoam openfoam  220 Feb 25  2020 .bash_logout
    -rw-r--r--. 1 openfoam openfoam 3804 Jan 17 19:38 .bashrc
    -rw-r--r--. 1 openfoam openfoam  807 Feb 25  2020 .profile
    -rw-r--r--. 1 openfoam openfoam   92 Jan 17 19:39 .sshd_config
    drwxrwxrwx. 3 root     root        1 Jan 21 16:35 storage

The `storage` folder is present, and if you do:

    ls -al storage/damBreak

You will see all the CFD assets. 

Go ahead and change directory:

    cd storage/damBreak

Then, run each of the following commands in this exact order exactly as
specified:

    blockMesh
    setFields
    decomposePar

The above sequence has broken up the CFD job into component parts that can be
run across 4 processors.

Go ahead and `exit` from your `bash` and/or `sh` shells back to your local
command prompt.

### OpenFOAM MPI Job
At this point you are ready to run your OpenFOAM MPI job. Take a look at the
`mpijob-dambreak-example.yaml` manifest to see the structure of an `mpijob`. The
MPI Operator repository provides more details. To paraphrase, our `mpijob` has 4
worker Replicas, and our `mpirun` command specifies 4 processors. Both the
Launcher and the Worker are both using our OpenFOAM image, and the rest of the
arguments to `mpirun` come from the OpenFOAM project. Go ahead and create the
`mpijob` manifest now:

    oc create -f manifests/mpijob-dambreak-example.yaml -n cfd

When you look at the Pods that are subsequently created, you will notice that
the launcher reports an `Error` state and ends up in a `CrashLoopBackoff`. This
is because of [this
issue](https://github.com/kubeflow/mpi-operator/issues/288#issuecomment-1011101805)
which is related to how OpenShift handles DNS resolution of service names.

Eventually the launcher should get into `Running` state. If you check its logs,
you will see a lot of this:

```
smoothSolver:  Solving for alpha.water, Initial residual = 0.00193684, Final residual = 3.40587e-09, No Iterations 3
Phase-1 volume fraction = 0.124752  Min(alpha.water) = -2.76751e-09  Max(alpha.water) = 1
MULES: Correcting alpha.water
MULES: Correcting alpha.water
Phase-1 volume fraction = 0.124752  Min(alpha.water) = -2.76751e-09  Max(alpha.water) = 1
DICPCG:  Solving for p_rgh, Initial residual = 0.0343767, Final residual = 0.0016133, No Iterations 5
time step continuity errors : sum local = 0.000480806, global = -4.75907e-08, cumulative = 7.68955e-05
DICPCG:  Solving for p_rgh, Initial residual = 0.00181566, Final residual = 8.38192e-05, No Iterations 33
time step continuity errors : sum local = 2.42908e-05, global = 3.04705e-06, cumulative = 7.99426e-05
DICPCG:  Solving for p_rgh, Initial residual = 0.000251337, Final residual = 8.59604e-08, No Iterations 109
time step continuity errors : sum local = 2.49879e-08, global = -9.43014e-10, cumulative = 7.99416e-05
ExecutionTime = 22.2 s  ClockTime = 23 s

Courant Number mean: 0.076648 max: 1.01265
Interface Courant Number mean: 0.00417282 max: 0.936663
deltaT = 0.0010575
Time = 0.11616
```

It's working!

With four processors on an m5a.4xlarge instance type, this job takes
approximately 650-750 seconds.