# OpenShift Image Mode & Lustre Client

Image mode for OpenShift allows you to easily extend the functionality of your base RHCOS image by layering additional images onto the base image. This layering does not modify the base RHCOS image. Instead, it creates a custom layered image that includes all RHCOS functionality and adds additional functionality to specific nodes in the cluster.

There are two methods for deploying a custom layered image onto your nodes:

* On-cluster image mode where we create a MachineOSConfig object that includes the Containerfile and other parameters. The build is performed on the cluster and the resulting custom layered image is automatically pushed to your repository and applied to the machine config pool that you specified in the MachineOSConfig object. The entire process is performed completely within your cluster.
  
* Out-of-cluster image mode where we create a Containerfile that references an OpenShift Container Platform image and the RPM that we want to apply, build the layered image in your own environment, and push the image to a repository. Then, in the cluster, we create a MachineConfig object for the targeted node pool that points to the new image. The Machine Config Operator overrides the base RHCOS image, as specified by the osImageURL value in the associated machine config, and boots the new image.

While I have written about out-of-cluster image mode before this example will focus on on-cluster image mode and specifically cover an example where I need the incorporate the Lustre client kernel drivers and packages into my OpenShift environment.

To get started we deployed a Single Node OpenShift environment running 4.20.8.   Note the process will not be any different if using multinode there will just be more nodes to apply the updated image to.

Next we need to create MachineOSConfig custom resource file that will define the additional components we need to add to RHCOS.  The following example shows that we will be doing the following:

* This MachineOSConfig will be built and applied to nodes in the master MachineOSConfig pool.  If we had workers in the worker pool we could change this and apply it there as well.  We can also apply it to custom pools as well.
* This MachineOSConfig will install EPEL, libyaml-devel and the four Lustre related client packages.  Dnf will ensure to pull in any additional packages.
* This MachineOSConfig has a renderedImagePushSpec pushing and pulling from Quay.io.  This could point to whichever registry where you want to store the image and then pull the image from.   A pull-secret for authorization is required.

~~~bash
$ cat  <<EOF > on-cluster-rhcos-layer-mc.yaml
apiVersion: machineconfiguration.openshift.io/v1 
kind: MachineOSConfig
metadata:
  name: worker 
spec:
  machineConfigPool:
    name: worker 
  containerFile: 
  - containerfileArch: NoArch 
    content: |-
      FROM configs AS final
      RUN  dnf install -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-9.noarch.rpm && \
        dnf install -y https://mirror.stream.centos.org/9-stream/CRB/x86_64/os/Packages/libyaml-devel-0.2.5-7.el9.x86_64.rpm && \
        dnf install -y https://downloads.whamcloud.com/public/lustre/lustre-2.15.7/el9.6/client/RPMS/x86_64/lustre-iokit-2.15.7-1.el9.x86_64.rpm \ 
                       https://downloads.whamcloud.com/public/lustre/lustre-2.15.7/el9.6/client/RPMS/x86_64/lustre-client-2.15.7-1.el9.x86_64.rpm \
                       https://downloads.whamcloud.com/public/lustre/lustre-2.15.7/el9.6/client/RPMS/x86_64/lustre-client-dkms-2.15.7-1.el9.noarch.rpm \
                       https://downloads.whamcloud.com/public/lustre/lustre-2.15.7/el9.6/client/RPMS/x86_64/kmod-lustre-client-2.15.7-1.el9.x86_64.rpm && \
        dnf clean all && \
        ostree container commit
  imageBuilder: 
    imageBuilderType: Job
  #baseImagePullSecret:
    #name: global-pull-secret-copy
  #renderedImagePushSpec: image-registry.openshift-image-registry.svc:5000/openshift/os-image:latest  
  renderedImagePushSpec: quay.io/redhat_emp1/ecosys-nvidia/os-image:latest
  renderedImagePushSecret: 
    name: bschmaus-secret
~~~

Once the MachineOSConfig custom resource file is generated we can create it on our cluster.

~~~bash
$ oc create -f on-cluster-rhcos-layer-mc.yaml 
machineosconfig.machineconfiguration.openshift.io/worker created
~~~

Once the MachineOSConfig has been created we can monitor it via the following command.

~~~bash
$ oc get machineosbuild
NAME                                      PREPARED   BUILDING   SUCCEEDED   INTERRUPTED   FAILED   AGE
worker-dc9806c4cbc6e9f76a9d8655f322eeb6   False      True       False       False         False    35s
~~~

We can also observe that a build-worker pod was created.

~~~bash
$ oc get pods -n openshift-machine-config-operator
NAME                                                  READY   STATUS      RESTARTS         AGE
build-worker-dc9806c4cbc6e9f76a9d8655f322eeb6-g5k6g   0/1     Init:0/1    0                33s
kube-rbac-proxy-crio-sno2.schmaustech.com             1/1     Running     9                44h
machine-config-controller-78b85fcd9c-h9gmn            2/2     Running     10               44h
machine-config-daemon-tlt8g                           2/2     Running     18 (4h58m ago)   44h
machine-config-nodes-crd-cleanup-29470933-l8jz2       0/1     Completed   0                44h
machine-config-nodes-crd-cleanup-29470952-xmlvn       0/1     Completed   0                44h
machine-config-operator-658ff78994-bpzpj              2/2     Running     10               44h
machine-config-server-mpwff                           1/1     Running     5                44h
machine-os-builder-65d7b4b97-xmn2n                    1/1     Running     0                44s
~~~

If we want to see more details on what is happening in the build-worker pod we can tail the logs of the image-build container inside the pod.  I am only showing the command to obtain the logs here because the output is quite long and verbose.  Further the build process takes awhile to run.

~~~bash
$ oc logs -f -n openshift-machine-config-operator build-worker-dc9806c4cbc6e9f76a9d8655f322eeb6-g5k6g -c image-build
~~~

~~~bash

~~~
