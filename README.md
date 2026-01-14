# OpenShift Image Mode & Lustre Client

Image mode for OpenShift allows you to easily extend the functionality of your base RHCOS image by layering additional images onto the base image. This layering does not modify the base RHCOS image. Instead, it creates a custom layered image that includes all RHCOS functionality and adds additional functionality to specific nodes in the cluster.

There are two methods for deploying a custom layered image onto your nodes:

* On-cluster image mode where we create a MachineOSConfig object that includes the Containerfile and other parameters. The build is performed on the cluster and the resulting custom layered image is automatically pushed to your repository and applied to the machine config pool that you specified in the MachineOSConfig object. The entire process is performed completely within your cluster.
  
* Out-of-cluster image mode where we create a Containerfile that references an OpenShift Container Platform image and the RPM that we want to apply, build the layered image in your own environment, and push the image to a repository. Then, in the cluster, we create a MachineConfig object for the targeted node pool that points to the new image. The Machine Config Operator overrides the base RHCOS image, as specified by the osImageURL value in the associated machine config, and boots the new image.

While I have written about out-of-cluster image mode before this example will focus on on-cluster image mode and specifically cover an example where I need the incorporate the Lustre client kernel drivers and packages into my OpenShift environment.

To get started we deployed a Single Node OpenShift environment running 4.20.8.   Note the process will not be any different if using multinode there will just be more nodes to apply the updated image to.

Next we need to create MachineOSConfig custom resource file that will define the additional components we need to add to RHCOS.  The following example shows that we will be doing the following:

* This image will be built and applied to nodes in the master MachineOSConfig pool.  If we had workers in the worker pool we could change this and apply it there as well.  We can also apply it to custom pools as well.
* This image will install additional packages:
  ** We need to install EPEL because dkms comes from there
  **
