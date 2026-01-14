# OpenShift Image Mode & Lustre Client

Image mode for OpenShift allows you to easily extend the functionality of your base RHCOS image by layering additional images onto the base image. This layering does not modify the base RHCOS image. Instead, it creates a custom layered image that includes all RHCOS functionality and adds additional functionality to specific nodes in the cluster.

There are two methods for deploying a custom layered image onto your nodes:

* On-cluster image mode where we create a MachineOSConfig object that includes the Containerfile and other parameters. The build is performed on the cluster and the resulting custom layered image is automatically pushed to your repository and applied to the machine config pool that you specified in the MachineOSConfig object. The entire process is performed completely within your cluster.
  
* Out-of-cluster image mode where we create a Containerfile that references an OpenShift Container Platform image and the RPM that we want to apply, build the layered image in your own environment, and push the image to a repository. Then, in the cluster, we create a MachineConfig object for the targeted node pool that points to the new image. The Machine Config Operator overrides the base RHCOS image, as specified by the osImageURL value in the associated machine config, and boots the new image.

While I have written about out-of-cluster image mode before this example will focus on on-cluster image mode and specifically cover an example where I need the incorporate the Lustre client kernel drivers and packages into my OpenShift environment.

