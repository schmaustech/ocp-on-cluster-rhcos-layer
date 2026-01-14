# OpenShift Image Mode & Lustre Client

Image mode for OpenShift allows you to easily extend the functionality of your base RHCOS image by layering additional images onto the base image. This layering does not modify the base RHCOS image. Instead, it creates a custom layered image that includes all RHCOS functionality and adds additional functionality to specific nodes in the cluster.

There are two methods for deploying a custom layered image onto your nodes:

* On-cluster image mode where we create a MachineOSConfig object that includes the Containerfile and other parameters. The build is performed on the cluster and the resulting custom layered image is automatically pushed to your repository and applied to the machine config pool that you specified in the MachineOSConfig object. The entire process is performed completely within your cluster.
  
* Out-of-cluster image mode where we create a Containerfile that references an OpenShift Container Platform image and the RPM that we want to apply, build the layered image in your own environment, and push the image to a repository. Then, in the cluster, we create a MachineConfig object for the targeted node pool that points to the new image. The Machine Config Operator overrides the base RHCOS image, as specified by the osImageURL value in the associated machine config, and boots the new image.

While I have written about out-of-cluster image mode before this example will focus on on-cluster image mode and specifically cover an example where I need the incorporate the Lustre client kernel drivers and packages into my OpenShift environment.

To get started we deployed a Single Node OpenShift environment running 4.20.8.   Note the process will not be any different if using multinode there will just be more nodes to apply the updated image to.

Next we need to generate our secrets that can be used in the builder process.  First let's set some environment variables for our internal registry, the user, the namespace and the token creation.

~~~bash
$ export REGISTRY=image-registry.openshift-image-registry.svc:5000
$ export REGISTRY_USER=builder
$ export REGISTRY_NAMESPACE=openshift-machine-config-operator
$ export TOKEN=$(oc create token $REGISTRY_USER -n $REGISTRY_NAMESPACE --duration=$((900*24))h)
~~~

Next let's create the push-secret using the variables we set in the openshift-machine-config-operator namespace.

~~~bash
$ oc create secret docker-registry push-secret -n openshift-machine-config-operator --docker-server=$REGISTRY --docker-username=$REGISTRY_USER --docker-password=$TOKEN
secret/push-secret created
~~~

Now we need to extract the push secret and the clusters global pull-secret.

~~~bash
$ oc extract secret/push-secret -n openshift-machine-config-operator --to=- > push-secret.json
# .dockerconfigjson

$ oc extract secret/pull-secret -n openshift-config --to=- > pull-secret.json
# .dockerconfigjson
~~~

We will now take the push-secret and global pull secret into one merged secret.

~~~bash
$ jq -s '.[0] * .[1]' pull-secret.json push-secret.json > pull-and-push-secret.json
~~~

This new merged secret needs to be create as well in the openshift-machine-config-operator namespace.

~~~bash
$ oc create secret generic pull-and-push-secret -n openshift-machine-config-operator --from-file=.dockerconfigjson=pull-and-push-secret.json --type=kubernetes.io/dockerconfigjson
secret/pull-and-push-secret created
~~~

~~~bash
$ oc get secrets -n openshift-machine-config-operator |grep push
pull-and-push-secret                        kubernetes.io/dockerconfigjson        1      10s
push-secret                                 kubernetes.io/dockerconfigjson        1      114s
~~~

Now we need to create MachineOSConfig custom resource file that will define the additional components we need to add to RHCOS.  The following example shows that we will be doing the following:

* This MachineOSConfig will be built and applied to nodes in the master MachineOSConfig pool.  If we had workers in the worker pool we could change this and apply it there as well.  We can also apply it to custom pools as well.
* This MachineOSConfig will install EPEL, libyaml-devel and the four Lustre related client packages.  Dnf will ensure to pull in any additional packages.
* This MachineOSConfig has a renderedImagePushSpec pushing and pulling to internal registry of the OCP cluster.  This could point to whichever registry where you want to store the image and then pull the image from.
* We also have our secrets that we created before defined in this file.

~~~bash
$ cat  <<EOF > on-cluster-rhcos-layer-mc.yaml
apiVersion: machineconfiguration.openshift.io/v1 
kind: MachineOSConfig
metadata:
  name: master 
spec:
  machineConfigPool:
    name: master 
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
  baseImagePullSecret: 
    name: pull-and-push-secret
  renderedImagePushSecret: 
    name: push-secret
  renderedImagePushSpec: image-registry.openshift-image-registry.svc:5000/openshift-machine-config-operator/os-image:latest
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
master-afc1942c842a324aa66271cbf5fcb0d8   False      True       False       False         False    16s
~~~

We can also observe that a build-worker pod was created.

~~~bash
$ oc get pods -n openshift-machine-config-operator
NAME                                                  READY   STATUS      RESTARTS         AGE
build-master-afc1942c842a324aa66271cbf5fcb0d8-fprgj   0/1     Init:0/1    0                29s
kube-rbac-proxy-crio-sno2.schmaustech.com             1/1     Running     9                46h
machine-config-controller-78b85fcd9c-h9gmn            2/2     Running     10               45h
machine-config-daemon-tlt8g                           2/2     Running     18 (6h37m ago)   45h
machine-config-nodes-crd-cleanup-29470933-l8jz2       0/1     Completed   0                46h
machine-config-nodes-crd-cleanup-29470952-xmlvn       0/1     Completed   0                45h
machine-config-operator-658ff78994-bpzpj              2/2     Running     10               46h
machine-config-server-mpwff                           1/1     Running     5                45h
machine-os-builder-65d7b4b97-m97lw                    1/1     Running     0                41s
~~~

If we want to see more details on what is happening in the build-worker pod we can tail the logs of the image-build container inside the pod.  I am only showing the command to obtain the logs here because the output is quite long and verbose.  Further the build process takes awhile to run.

~~~bash
$ oc logs -f -n openshift-machine-config-operator build-master-afc1942c842a324aa66271cbf5fcb0d8-fprgj -c image-build
~~~

When the build finishes the image will get pushed to the registry defined in the MachineOSConfig.  The logs will have a reference in there like this at the end.

~~~bash
+ buildah push --storage-driver vfs --authfile=/tmp/final-image-push-creds/config.json --digestfile=/tmp/done/digestfile --cert-dir /var/run/secrets/kubernetes.io/serviceaccount image-registry.openshift-image-registry.svc:5000/openshift-machine-config-operator/os-image:master-afc1942c842a324aa66271cbf5fcb0d8
Getting image source signatures
Copying blob sha256:3a1265f127cd4df9ca7e05bf29ad06af47b49cff0215defce94c32eceee772bc
Copying blob sha256:d87a18a2396ee3eb656b6237ac1fa64072dd750dde5aef660aff53e52c156f56
(...)
Copying blob sha256:1d82edb13736f9bbad861d8f95cae0abfe5d572225f9d33d326e602ecc5db5fb
Copying blob sha256:eb199ffe5f75bd36c537582e9cf5fa5638d55b8145f7dcd3cfc6b28699b2568d
Copying config sha256:3d835eb02f08fe48d26d9b97ebcf0e190c401df2619d45cce1a94b0845d7f4e2
Writing manifest to image destination
~~~

At that point if we look at the machineosbuild output we will see the image moved to succeeded.

~~~bash
$ oc get machineosbuild
NAME                                      PREPARED   BUILDING   SUCCEEDED   INTERRUPTED   FAILED   AGE
master-afc1942c842a324aa66271cbf5fcb0d8   False      False      True        False         False    24m
~~~

And we will see that the machine config pool is now in an updating state.   At this point the image that was build is getting applied to the system and it will reboot.

~~~bash
$ oc get mcp
NAME     CONFIG                                             UPDATED   UPDATING   DEGRADED   MACHINECOUNT   READYMACHINECOUNT   UPDATEDMACHINECOUNT   DEGRADEDMACHINECOUNT   AGE
master   rendered-master-2d9e26ad2557d1859aadc76634a4f1a5   False     True       False      1              0                   0                     0                      46h
worker   rendered-worker-36ba71179b413c7b7abc3e477e7367d5   True      False      False      0              0                   0                     0                      46h
~~~

Once the node or nodes if multicluster reboot we should be able to open a debug pod on them and validate that are kernel modules and client were installed properly.  First let's open a debug prompt.

~~~bash
$ oc debug node/sno2.schmaustech.com
Starting pod/sno2schmaustechcom-debug-xcvdh ...
To use host binaries, run `chroot /host`. Instead, if you need to access host namespaces, run `nsenter -a -t 1`.
Pod IP: 192.168.0.204
If you don't see a command prompt, try pressing enter.
sh-5.1# chroot /host
sh-5.1# 
~~~

Next let's confirm the Lustre rpm packages are present.

~~~bash
sh-5.1# rpm -qa|grep lustre
kmod-lustre-client-2.15.7-1.el9.x86_64
lustre-client-dkms-2.15.7-1.el9.noarch
lustre-client-2.15.7-1.el9.x86_64
lustre-iokit-2.15.7-1.el9.x86_64
~~~

The packages are there now let's see if the Lustre kernel module is loaded.  It might not be because my understanding is that it requires a process to request it first.  If its not there we can manually load it.

~~~bash
sh-5.1# lsmod|grep lustre
sh-5.1# modprobe lustre
sh-5.1# lsmod|grep lustre
lustre               1155072  0
lmv                   233472  1 lustre
mdc                   315392  1 lustre
lov                   385024  2 mdc,lustre
ptlrpc               1662976  7 fld,osc,fid,lov,mdc,lmv,lustre
obdclass             3571712  8 fld,osc,fid,ptlrpc,lov,mdc,lmv,lustre
lnet                  884736  6 osc,obdclass,ptlrpc,ksocklnd,lmv,lustre
libcfs                262144  11 fld,lnet,osc,fid,obdclass,ptlrpc,ksocklnd,lov,mdc,lmv,lustre
~~~~

We can see our image has been updated and contains the necessary packages.   Hopefully this provides an example of how to add 3rd party drivers and packages to an OpenShift environment.
