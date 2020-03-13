---
layout: post
author: Alberto Losada Grande
description: This second part focuses on deploying custom-built containerized images into our Kubernetes cluster. However, this is just the beginning since our users need to understand how VMs can be accessed, how extra disks can be added or how standardize VMs can be adapted to their needs with cloud-init. 
navbar_active: Blogs
category: news
tags:
  [
    "kubevirt",
    "kubernetes",
    "virtual machine",
    "okd",
    "openshift",
    "containerDisk",
    "dockerfile",
    "registry",
    "quay.io",
  ]
comments: true
title: Customizing images for containerized VMs (2/2)
pub-date: March 18
pub-year: 2020
---

<!-- TOC depthFrom:2 insertAnchor:false orderedList:false updateOnSave:true withLinks:true -->

- [Previously on customizing images for containerized VMs](#previously-on-customizing-images-for-containerized-vms)
- [Create a VM from the container image](#create-a-vm-from-the-container-image)
- [Connect to the deployed custom-built VM](#connect-to-the-deployed-custom-built-vm)
    - [OKD web user interface](#okd-web-user-interface)
    - [Console serial or VNC using virtctl](#console-serial-or-vnc-using-virtctl)
    - [SSH using service nodePort](#ssh-using-service-nodeport)
    - [SSH using multus CNI](#ssh-using-multus-cni)
- [Add disks to the custom-built VM](#add-disks-to-the-custom-built-vm)
    - [emptyDisk (ephemeral)](#emptydisk-ephemeral)
    - [hostDisk (persistent)](#hostdisk-persistent)
    - [Other options](#other-options)
- [Adapt the VM far beyond with cloud-init](#adapt-the-vm-far-beyond-with-cloud-init)
- [Summary](#summary)
- [References](#references)

<!-- /TOC -->


## Previously on customizing images for containerized VMs

In the previous article [Customizing images for containerized VMs, part 1]({% post_url 2020-03-11-Customizing-images-for-containerized-vms %}) it was explained the vision of a software company, which wanted the Software Engineering team to be autonomous to deploy, run and destroy standardized VMs in the current Kubernetes infrastructure. On top of these custom-built virtual machines the necessary tests and validations needs to be executed before commiting the code to the respective Git branch. 

Thus, developers can request a fresh standard environment to test their code and more important, using the same tooling (`kubectl`, `oc`) they utilize when running applications in the corporate OKD Kubernetes cluster. From the Systems Engineering department point of view, there is no longer the need to maintain two different infrastructures to run containerized workloads and virtual machines.

Throughout the previous blog post, the process of creating a *golden image* using tools such as _Builder Tool_ and _virt-customize_ was described. Once the custom-built image was ready, it was containerized, uploaded and stored into the private OKD container registry.

At that point, the following container images are stored in the openshift namespace, so they are available to all authenticated OKD Kubernetes users. 

```sh
$ oc get is devstation -n openshift
NAME         IMAGE REPOSITORY                                                                   TAGS                                    UPDATED
devstation   default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation   v8-terminal,v7-gui,v7-terminal,v8-gui   7 hours ago
```

> info "Information"
> In the previous command, the [imageStream](https://docs.openshift.com/container-platform/4.3/openshift_images/image-streams-manage.html) called devstation consists of 4 container images tagged as: **v8-terminal,v7-gui,v7-terminal,v8-gui** which corresponds to the different custom-built images containerized. In the next command, more information is shown regarding each container image.

```sh
$ oc describe imagestream devstation -n openshift
Name:			devstation
Namespace:		openshift
Created:		6 days ago
Labels:			<none>
Annotations:		<none>
Image Repository:	default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation
Image Lookup:		local=false
Unique Images:		5
Tags:			4

v7-gui
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/openshift/devstation@sha256:05e75ff0ca2206360a7132885fe10adb294e128bd0db658d2062d081140246ed
      2 days ago

v7-terminal
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/openshift/devstation@sha256:fba5fa23284414d456573b58326211ba849e3d1d7885200e0586e3d674b0d485
      4 days ago

v8-gui
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/openshift/devstation@sha256:e301d935c1cb5a64d41df340d78e6162ddb0ede9b9b5df9c20df10d78f8fde0f
      6 days ago

v8-terminal
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/openshift/devstation@sha256:2455f3168e18d9ae5d1bdc09a10db601e5c26859411ae1752a95abe64981502c
      7 hours ago
```

> note "Note"
> In case you did not build the custom images as detailed in the [Customizing images for containerized VMs part 1]({% post_url 2020-03-11-Customizing-images-for-containerized-vms %}) you still can download them from my [public container image repository](https://quay.io/repository/alosadag/devstation?tab=tags) at quay.io. 

During this article, the custom-built container images stored in the OKD container registry are going to be used to deploy virtual machines in Kubernetes. Whether you want some hands-on experience while reading this write up, you can just copy the public container images from quay.io container registry to your private container registry using tools like `skopeo`:

> info "Information"
> The process of copying images to OKD container registry was explained in the section [store the image in the container registry](https://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#store-the-image-in-the-container-registry) from the previous article [Customizing images for containerized VMs, part 1]({% post_url 2020-03-11-Customizing-images-for-containerized-vms %})

```sh
$ podman login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false default-route-openshift-image-registry.apps.okd.okdlabs.com
Login Succeeded!

$ podman login quay.io
Existing credentials are valid. Already logged in to quay.io

$ skopeo copy --dest-tls-verify=false docker://quay.io/alosadag/devstation:v7-terminal docker://default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation:v7-terminal
Getting image source signatures
Copying blob f51fdc2c219a [=>------------------------------------] 23.1MiB / 558.8MiB
Copying blob f51fdc2c219a [==>-----------------------------------] 36.8MiB / 558.8MiB
Copying blob f51fdc2c219a [=====>--------------------------------] 89.3MiB / 558.8MiB
Copying blob f51fdc2c219a [===========>--------------------------] 180.7MiB / 558.8MiB
Copying blob f51fdc2c219a [==================================>---] 517.2MiB / 558.8MiB
Copying blob f51fdc2c219a done  
Copying config 9109717b83 done  
Writing manifest to image destination
Storing signatures
```
<br>

Other valid option is just **replace the image name from the YAML *VirtualMachine* manifest** to pull it directly from the quay.io, which is a public container image registry, instead of the private OKD.


## Create a VM from the container image 

Once the images are stored in the OKD container registry, it is time to create the *VirtualMachine* Kubernetes YAML definition that pulls the proper image and configures the virtual machine managed by KubeVirt. First, a new namespace is recommended to be created. OKD is configured to allow members of the _A team_ to run their VMs and verifications in that namespace.

```sh
$ kubectl create ns dev-team-a
namespace/dev-team-a created
```
<br>

Below, it is shown the _VirtualMachine_ YAML manifest that deploys a CentOS 8 container image (without GNOME) as _containerDisk_ pulled from the internal registry: https://image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-terminal. Notice that the image is configured as a containerDisk volume.

> info "Information"
> More information regarding the containerization procedure and containerDisk can be found in [image containerization procedure from part 1](https://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#image-containerization-procedure)

Notice that developers can request the adequate memory and cpu to run and test their applications. They are *resource* parameters from the *VirtualMachine* Custom Resource (CR). The YAML definition show below can be downloaded from [here](/assets/2020-03-18-Customizing-images-for-containerized-vms-2/vm-devstation-8.yaml). 

```yaml
apiVerson: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: devstation
spec:
  running: true 
  template:
    metadata:
      labels: 
        kubevirt.io/size: small
        kubevirt.io/domain: devstation
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: default
            bridge: {}
        resources:
          requests:
            cpu: 1
            memory: 1Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-terminal
            imagePullPolicy: IfNotPresent
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: | 
              #cloud-config
```
<br>

Now, the YAML manifest is applied in the proper namespace:

```sh
$ kubectl create -f vm-devstation-8.yaml -n dev-team-a
virtualmachine.kubevirt.io/devstation created
```
<br>

Below are shown some events logged in the dev-team-a namespace while the _VirtualMachineInstance_ is being created. See how the _VirtualMachine_ and _VirtualMachineInstance_ (VMI) are created. Observe how the virt-launcher and the custom-built image (devstation-8-terminal) are being pulled. Eventually, the _VirtualMachineInstance_ (devstation) created and started:

```sh
oc get events -w
LAST SEEN   TYPE     REASON             OBJECT                               MESSAGE
33s         Normal   SuccessfulCreate   virtualmachine/devstation            Started the virtual machine by creating the new virtual machine instance devstation
33s         Normal   SuccessfulCreate   virtualmachineinstance/devstation    Created virtual machine pod virt-launcher-devstation-btq6n
<unknown>   Normal   Scheduled          pod/virt-launcher-devstation-btq6n   Successfully assigned dev-team-a/virt-launcher-devstation-btq6n to okd-worker-0.okd.okdlabs.com
24s         Normal   Pulled             pod/virt-launcher-devstation-btq6n   Container image "index.docker.io/kubevirt/virt-launcher@sha256:170469602559abbfba57d99c54d74458fbc233b068beaad4f1f201002a5c8770" already present on machine
24s         Normal   Created            pod/virt-launcher-devstation-btq6n   Created container compute
24s         Normal   Started            pod/virt-launcher-devstation-btq6n   Started container compute
24s         Normal   Pulling            pod/virt-launcher-devstation-btq6n   Pulling image "image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-terminal"
2s          Normal   Pulled             pod/virt-launcher-devstation-btq6n   Successfully pulled image "image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-terminal"
2s          Normal   Created            pod/virt-launcher-devstation-btq6n   Created container volumecontainerdisk
2s          Normal   Started            pod/virt-launcher-devstation-btq6n   Started container volumecontainerdisk
1s          Normal   Created            virtualmachineinstance/devstation    VirtualMachineInstance defined.
1s          Normal   Started            virtualmachineinstance/devstation    VirtualMachineInstance started.
2s          Normal   Created            virtualmachineinstance/devstation    VirtualMachineInstance defined.
2s          Normal   Created            virtualmachineinstance/devstation    VirtualMachineInstance defined.
```

<br>

> note "Note"
> The first time the custom-built image is pulled to a Node it can take a few minutes since the image must be copied. The second time it is pulled to the same Node the process is much faster since the [imagePullPolicy](https://kubernetes.io/docs/concepts/containers/images/) is configured to **IfNotPresent**. In general, custom-built VM images are much heavier than regular container images. This directly affects to the time needed to copy the image from the container image registry to the Node. 

Lastly, verify that the _VirtualMachineInstance_ (VMI) is running successfully:

```sh
$ kubectl get vmi -o wide
NAME             AGE     PHASE     IP               NODENAME                       LIVE-MIGRATABLE
devstation       19h     Running   10.135.0.49/23   okd-worker-2.okd.okdlabs.com   False
```

<br>

At this stage, the virtual machine is ready to be used for testing and verifying the new code written by the developers. The next question that will arise from the users is... OK... so how can I connect to the VM and do the work? There are multiple options that are presented one by one in the next section.


## Connect to the deployed custom-built VM

KubeVirt permits several ways to connect and work with VMs running on top of Kubernetes. Below are shown some of the most common used alternatives.

### OKD web user interface

Probably, this is the most straightforward. Since OKD deploys a friendly console, it is possible to connect using VNC or serial console directly from the web user interface. 

> info "Information"
> More information on how to deploy the OKD web console in a native Kubernetes cluster can be found in [Managing KubeVirt with Openshift Web Console]({% post_url 2020-01-24-OKD-web-console-install %})


<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/centos8-terminal-okd.png"
      itemprop="contentUrl"
      data-size="1110x527"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/centos8-terminal-okd.png"
        itemprop="thumbnail"
        width="100%"
        alt="CentOS 8 OKD console"
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

<br>

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/centos8-gui-okd.png"
      itemprop="contentUrl"
      data-size="1110x476"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/centos8-gui-okd.png"
        itemprop="thumbnail"
        width="100%"
        alt="CentOS 8 OKD console"
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

### Console serial or VNC using virtctl

The developers can use the _virtctl_ binary or the _virtctl_ plugin for [krew](https://krew.dev/) to connect from their laptop to the virtual machine running in the Kubernetes cluster. The installation is well documented in the [quickstart guide](https://kubevirt.io/quickstart_minikube/#install-virtctl). Observe that _virtctl_ allows us to connect via serial or VNC console. 

```sh
$ kubectl virt console --help
Connect to a console of a virtual machine instance.

Usage:
  kubectl virt console (VMI) [flags]

Examples:
  # Connect to the console on VirtualMachineInstance 'myvmi':
  kubectl virt console myvmi
  # Configure one minute timeout (default 5 minutes)
  kubectl virt console --timeout=1 myvmi

Flags:
  -h, --help          help for console
      --timeout int   The number of minutes to wait for the virtual machine instance to be ready. (default 5)
```

```sh
$ kubectl virt vnc --help
Open a vnc connection to a virtual machine instance.

Usage:
  kubectl virt vnc (VMI) [flags]

Examples:
  # Connect to 'testvmi' via remote-viewer:\n"
  kubectl virt vnc testvmi
```
<br>

Below, it is depicted a serial console and a VNC connection established between a user's workstation and the CentOS 8 VM (running GNOME) using _virtctl_:

```sh
$ kubectl virt console devstation-8-gui
```
<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/virtctl-serial-console.png"
      itemprop="contentUrl"
      data-size="1110x488"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/virtctl-serial-console.png"
        itemprop="thumbnail"
        width="100%"
        alt="CentOS 8 OKD console"
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

> note "Note"
> Notice that the disk is actually 10 GB and the root filesystem was successfully expanded to 9 GB as described in the [image creation with Builder Tool section from part 1](http://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#image-creation-with-builder-tool)


```sh
$ kubectl virt vnc devstation-8-gui
```

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/virtctl-vnc-console.png"
      itemprop="contentUrl"
      data-size="924x720"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/virtctl-vnc-console.png"
        itemprop="thumbnail"
        width="90%"
        alt="CentOS 8 OKD console"
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

### SSH using service nodePort

All custom-built container images were shipped with the *OpenSSH* server installed and configured to start-up at boot time. Hence, probably SSH is the most common way to access to the VM in case there is no need for using the graphical user interface.

In order to connect through SSH from a workstation, a [service nodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) is an option that can be used. A service nodePort is a service type within Kubernetes that exposes a service on each Node's IP at a static port. Thus, end users SSH to the static port of any Node in order to access to the desired VM. There is specific information in KubeVirt documentation about [how to expose a VMI](https://kubevirt.io/user-guide/#/usage/network-service-integration?id=expose-virtualmachineinstance-as-a-nodeport-service) in case you need further detail.

In the first place, a nodePort service that maps to the _VirtualMachineInstance_ SSH port (tcp/22) must be created and applied. The YAML service manifest shown below can be also downloaded from [here](/assets/2020-03-18-Customizing-images-for-containerized-vms-2/vm-svc-ssh-nodeport.yaml)

```yaml
apiVersion: v1
kind: Service
metadata:
  name: ssh-np-8-gui
spec:
  externalTrafficPolicy: Cluster
  ports:
  - name: nodeport
    nodePort: 30000
    port: 27017
    protocol: TCP
    targetPort: 22
  selector:   # Verify the selector matches the proper VMI
    kubevirt.io/domain: devstation-8-gui
  type: NodePort
```
<br>

After applying the service, verify the proper endpoints are detected:

```sh
oc get svc -o wide
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)           AGE     SELECTOR
ssh-np-8-gui       NodePort    172.30.14.216    <none>        27017:30000/TCP   2m45s   kubevirt.io/domain=devstation-8-gui
```

```sh
oc get endpoints devstation-8-ssh
NAME               ENDPOINTS         AGE
devstation-8-ssh   10.133.1.108:22   3d22h
``` 

> warning "Warning"
> Double check that the service selector matches the proper VMI. In the example above, the *selector* proxies the connection requests from the user to any pod inside the dev-team-a namespace that its "kubevirt.io/domain" label is equal to "devstation-8-gui".

Ultimately, connect to any Node of the cluster at port tcp/3000. The connection must be redirected to the SSH service running in the *devstation-8-gui* VM.

```sh
$ ssh alosadag@192.168.17.106 -p 30000 
The authenticity of host [192.168.17.106]:30000 ([192.168.17.106]:30000) can't be established.
ECDSA key fingerprint is SHA256:ABeejDf7+Z4fqVWsO85jTRIB6bjypA7XM1PdvAP/01Q.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added [192.168.17.106]:30000 (ECDSA) to the list of known hosts.
Activate the web console with: systemctl enable --now cockpit.socket

[alosadag@devstation-8-gui ~]$ 
```

> note "Note"
> Notice that we have connected using public key authentication with a user called *alosadag* instead of developer or sysadmin. This is because we have injected a specific configuration in the virtual machine (devstation-8-gui) manifest utilizing cloud-init. In the following section, [Adapt the VM far beyond with cloud-init](#adapt-the-vm-far-beyond-with-cloud-init), it is further detailed how to customize the VM at boot time using cloud-init.

Finally, it is worth mentioning that *virtctl* is able to create service nodePort from the command-line. Take a look at the **expose** option.

```sh
$ virtctl expose --help
Looks up a virtual machine instance, virtual machine or virtual machine instance replica set by name and use its selector as the selector for a new service on the specified port.
A virtual machine instance replica set will be exposed as a service only if its selector is convertible to a selector that service supports, i.e. when the selector contains only the matchLabels component.

Examples:
  # Expose SSH to a virtual machine instance called 'myvm' on each node via a NodePort service:
  virtctl expose vmi myvm --port=22 --name=myvm-ssh --type=NodePort
```

<br>

### SSH using multus CNI

Another alternative to connect to the virtual machines running in the OKD Kubernetes cluster is using [Multus CNI](https://github.com/intel/multus-cni). Multus CNI is a container network interface (CNI) plugin for Kubernetes that enables attaching multiple network interfaces to pods. This can be accomplished because Multus acts as a "meta-plugin", a CNI plugin that can call multiple other CNI plugins.

> info "Information"
> This topic was covered previously in [Attaching to Multiple Networks](https://kubevirt.io/2018/attaching-to-multiple-networks.html) blog post.

The idea is configuring the VMs with two network interfaces: the default interface, which is connected to the pod SDN as usual, and a new interface connected directly to the corporate network through Node's physical interface. This second interface gets an IP address directly from the corporate network. Thus, users can connect via SSH to the VM's second interface as they usually do with ordinary virtual or physical servers.

> info "Information"
> OKD comes with Multus out of the box. A proper CNI configuration is just needed as detailed in the [CNI plug-in](https://docs.openshift.com/container-platform/4.2/networking/multiple_networks/understanding-multiple-networks.html#additional-networks-provided) OpenShift documentation. In our environment, the [host-device](https://docs.openshift.com/container-platform/4.3/networking/multiple_networks/configuring-host-device.html#configuring-host-device) plug-in was configured. 

As depicted in the diagram below, OKD Nodes were configured with two interfaces: eth0 and eth1, which correspond to service and management interfaces respectively. Eth1 (management interface) was configured to be used by the VMs running on top of Kubernetes to acquire an IP address (dhcp or static) from the corporate management network.

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/multus-diagram.png"
      itemprop="contentUrl"
      data-size="937x720"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/multus-diagram.png"
        itemprop="thumbnail"
        width="70%"
        alt=""
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

> info "Information"
> The host-device Container Network Interface (CNI) plug-in allows you to move the specified network device from the host’s network namespace into the Pod’s network namespace.

Taking into account that management network configures `192.168.18.0/24` addressing, the new interface added to the VM from the OKD user interface must obtain an IP from this network range. 

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/multus-okd-dialog.png"
      itemprop="contentUrl"
      data-size="1110x431"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/multus-okd-dialog.png"
        itemprop="thumbnail"
        width="100%"
        alt=""
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

> note "Note"
> Multus `kubevirt-blog` network shown in the picture above is configured accordingly to host-device documentation.

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/multus_okd.png"
      itemprop="contentUrl"
      data-size="1110x376"
    >
      <img
        src="/assets/2020-03-18-Customizing-images-for-containerized-vms-2/multus_okd.png"
        itemprop="thumbnail"
        width="100%"
        alt=""
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

<br>

At that point, the developers are capable of connecting (SSH) to the management interface of the VM running in the cluster, since it is directly connected to the corporate network through Node's interface. In the following snippet, we are connected to the VM (devstation-8-hostdisk) through serial console. The IPs of all network interfaces are shown:

```sh
[alosadag@sonnelicht ]$ kubectl virt console devstation-8-hostdisk
...
$ ifconfig eth1
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.18.158  netmask 255.255.255.0  broadcast 192.168.18.255
$ ifconfig eth0
eth0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1450
        inet 10.135.0.79  netmask 255.255.254.0  broadcast 10.135.1.255
```

> note "Note"
> Notice that eth0 IP belongs to pod SDN network while eth1 IP address is part of the management corporate network (192.168.18.0/24).

Eventually, below it is confirmed that a developer from her workstation (sonnelicht) can connect to the second interface of the virtual machine (192.168.18.158 / devstation-8-hostdisk). Worth mentioning that the network team must ensure that users are allowed to connect from their current network to the management network.

```sh

[alosadag@sonnelicht ]$ ifconfig enp0s31f6
enp0s31f6: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.20.251  netmask 255.255.255.0  broadcast 192.168.20.255

[alosadag@sonnelicht ]$ ssh root@192.168.18.158 
The authenticity of host '192.168.18.158 (192.168.18.158)' can't be established.
ECDSA key fingerprint is SHA256:xL8wCu/JNlzz6WS6htMTF87iBgHKwmy9Lx4RjL9ihZs.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '192.168.18.158' (ECDSA) to the list of known hosts.
Last login: Mon Mar  9 17:16:45 2020 from gateway

[root@devstation-8-hostdisk ~]# ifconfig eth1
eth1: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
        inet 192.168.18.158  netmask 255.255.255.0  broadcast 192.168.18.255
``` 

<br>

## Add disks to the custom-built VM

Presumably, there is a chance that users need more disk space for the VMs deployed from the custom-built images. In the former article, QCOW2 custom-built images were [already expanded](http://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#image-creation-with-builder-tool) to 10 GB size in order to meet the Software Engineering team demands. Despite that, an extra disk can be desirable in cases where there is a requirement to install a large number of packages or import massive data.


### emptyDisk (ephemeral)

In cases where there is a need for extra storage, the developers can take advantage of the [emptyDisk](https://kubevirt.io/user-guide/#/creation/disks-and-volumes?id=emptydisk) feature. It adds an extra sparse QCOW2  **ephemeral** disk to the VM as long as it exists. This means that the data stored in the emptyDisk will survive guest side reboots, but not VM re-creation.

> info “Information”
> Capacity parameter of emptyDisk must be specified in the YAML manifest.

Below, there is an example of YAML manifest where an extra emptyDisk of 10 GB was requested along with the usual containerDisk. The YAML manifest can be downloaded from [here](/assets/2020-03-18-Customizing-images-for-containerized-vms-2/vm-devstation-8-extradisk.yaml).

```sh
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: devstation-8-extradisk
spec:
  running: true
  template:
    metadata:
      labels:
        kubevirt.io/size: small
        kubevirt.io/domain: devstation-8-extradisk
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
            - name: emptydisk  # emptyDisk definition
              disk:
                bus: virtio
          interfaces:
          - name: default
            bridge: {}
        resources:
          requests:
            cpu: 1
            memory: 1Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-gui
            imagePullPolicy: IfNotPresent
        - name: emptydisk
          emptyDisk:
            capacity: "10Gi"  # capacity requested for the ephemeral emptyDisk
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: |
              #cloud-config
              users:
                - name: alosadag
                  ssh-authorized-keys:
                    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCu6ebpjHClLxurBN3csEZNbIo3493cqFZGCnpicdeipIWVcU+Hdvy0/CIiZ8AQvcNsaC44zC9PYlHnoFWIfxrGvPvZikUC/qv4xmzNZa60Ype4M3tyMUXlP+vbDu3sOgeIWhyiyvV7NN2DGYF9mi0eKV5u/awS2B/pL6KXPjXjwVZNFB9aur9DkVClgAHc+18pI11RMMuUK4oOPsdopvECgZoWK/zuXrCJQFmqMqlArgveOhGU1FGUpeG63lsf38Xe2+9EXhhcMMuLcYN2BxOwdyRcHt0uflI3SagufUHbBA0vLF4XeTaoU+X2NFM5ToQDdP5Aa+s0mKgGTR9SpknT ansible-generated on eko1.cloud.lab.eng.bos.redhat.com
                  lock_passwd: false
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  groups: users, wheel

```

<br>

Once the VMI is deployed successfully, it is time to connect and verify there is actually an extra block device of the requested capacity.

```sh
[alosadag@sonnelicht $] kubectl virt console devstation-8-extradisk
Successfully connected to devstation-8-extradisk console. The escape sequence is ^]

CentOS Linux 8 (Core)
Kernel 4.18.0-147.5.1.el8_1.x86_64 on an x86_64

Activate the web console with: systemctl enable --now cockpit.socket

devstation-8-extradisk login: developer
Password: 

[root@devstation-8-extradisk ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0   10G  0 disk 
├─vda1 253:1    0    1G  0 part /boot
└─vda2 253:2    0    9G  0 part /
vdb    253:16   0  366K  0 disk 
vdc    253:32   0   10G  0 disk 

[root@devstation-8-extradisk ~]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        475M     0  475M   0% /dev
tmpfs           490M     0  490M   0% /dev/shm
tmpfs           490M   14M  477M   3% /run
tmpfs           490M     0  490M   0% /sys/fs/cgroup
/dev/vda2       9.0G  4.7G  4.4G  52% /
/dev/vda1       976M   45M  865M   5% /boot
tmpfs            98M  7.3M   91M   8% /run/user/42
tmpfs            98M  4.0K   98M   1% /run/user/1001
```

> note "Note"
> Observe that /dev/vda contains the root file system, /dev/vdc is the extra disk requested and /dev/vdb contains the cloud-init configuration referenced as cloudinitdisk. See that the cloudinitdisk (dev/vdc) contains the cloud-init data applied to the VM at boot time.
> ```sh
>[root@devstation-8-extradisk ~]# mount /dev/vdb /mnt/
>mount: /mnt: WARNING: device write-protected, mounted read-only.
>[root@devstation-8-extradisk ~]# ls -l /mnt/
>total 2
>-rw-r--r--. 1 root root  99 Mar  6 17:24 meta-data
>-rw-r--r--. 1 root root 591 Mar  6 17:24 user-data
>```

<br>

### hostDisk (persistent)

While *emptyDisk* presents an ephemeral extra block device to the VM, there might be cases where users need persistent extra disk. In thoses cases a [hostDisk](https://kubevirt.io/user-guide/#/creation/disks-and-volumes?id=hostdisk) can be requested. 

> info "Information"
> *hostDisk* provides the ability to create or use a disk image obtained from the local storage of a single Node. It is a similar concept to *hostPath* in Kubernetes.

It is important to appreciate that the *hostDisk* uses local Node storage and it does not replicate to all the other Nodes of the cluster. So, the data is local to the Node where the VM is running.

> warning "Warning"
> If you request a _hostDisk_ make sure the VM attached to the disk always runs in the same Node where the _hostDisk_ is located.

A [YAML VirtualMachine manifest](/assets/2020-03-18-Customizing-images-for-containerized-vms-2/vm-devstation-8-hostdisk.yaml) example of running a devstation with a _hostDisk_ attached is presented below. 

```sh
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: devstation-8-hostdisk
spec:
  running: true 
  template:
    metadata:
      labels: 
        kubevirt.io/size: small
        kubevirt.io/domain: devstation-8-hostdisk
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
            - name: host-disk # hostDisk definition
              disk:
                bus: virtio
          interfaces:
          - name: default
            bridge: {}
        resources:
          requests:
            cpu: 1
            memory: 1Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: image-registry.openshift-image-registry.svc:5000/openshift/devstation:v7-terminal
            imagePullPolicy: IfNotPresent
        - name: host-disk
          hostDisk:
            capacity: "1Gi"  # host-disk capacity set to 1 Gi
            path: /mnt/disk.img  # local path to the Node running the VM where the disk is stored
            type: DiskOrCreate  # create disk if does not exist
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: | 
```

<br>

The result of applying the YAML manifest is stated below. Observer that the VM (devstation-8-hostdisk) was deployed with an extra block device of 1 Gi (dev/vdc).
```sh
oc get vmi -o wide
NAME               AGE     PHASE     IP                NODENAME                       LIVE-MIGRATABLE
devstation-8-hostdisk   3h55m   Running   10.133.1.108/23   okd-worker-0.okd.okdlabs.com   False
```

```sh
[alosadag@sonnelicht $] kubectl virt console devstation-8-hostdisk
Successfully connected to devstation-8-hostdisk console. The escape sequence is ^]

CentOS Linux 8 (Core)
Kernel 4.18.0-147.5.1.el8_1.x86_64 on an x86_64

Activate the web console with: systemctl enable --now cockpit.socket

devstation-8-hostdisk login: developer
Password: 

[developer@devstation-8-hostdisk ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vda    253:0    0   10G  0 disk 
├─vda1 253:1    0    1G  0 part /boot
└─vda2 253:2    0    9G  0 part /
vdb    253:16   0  366K  0 disk 
vdc    253:32   0    1G  0 disk 
```

<br>

Furthermore, observe that it has been created an extra-disk of 1 Gi in the okd-worker-0.okd.okdlabs.com worker. That is the location (/mnt/data.img) configured and the Node where the VMI is running.

```sh
[root@okd-worker-0 ~]# ls -l /mnt/
total 4
-rw-r--r--. 1 107 107 1073741824 Mar  9 10:06 disk.img
[root@okd-worker-0 ~]# ls -lh /mnt/
total 4.0K
-rw-r--r--. 1 107 107 1.0G Mar  9 10:06 disk.img
```

<br>


### Other options

AS it has been explained along these series of two blog posts, the company does not need to persist the VM. However, there are some cases where it is necessary to copy some libraries or files from the internal shared fileserver or viceversa. In those cases, a shared volume can be mounted in our VM running on top of OKD Kubernetes. Below, it is shown how a 50G shared volume from the corporate NFS server is mounted inside the VM (devstation-8-gui):

```sh
[root@devstation-8-gui ~]# mount -t nfs nfsserver:/srv/nfs4/ /mnt
[root@devstation-8-gui ~]# df -h
Filesystem            Size  Used Avail Use% Mounted on
devtmpfs              979M     0  979M   0% /dev
tmpfs                 994M     0  994M   0% /dev/shm
tmpfs                 994M   18M  977M   2% /run
tmpfs                 994M     0  994M   0% /sys/fs/cgroup
/dev/vda2             9.0G  4.7G  4.4G  52% /
/dev/vda1             976M   45M  865M   5% /boot
tmpfs                 199M  7.3M  192M   4% /run/user/42
tmpfs                 199M  4.0K  199M   1% /run/user/1001
nfsserver:/srv/nfs4/   50G  390M   50G   1% /mnt

[root@devstation-8-gui ~]# ls -l /mnt
total 0
drwxr-xr-x. 2 root root 6 Mar  6 23:11 apps
```

> note "Note"
> Notice that you are running real VMs, so you have access to tools like nfs-clients or any other packages available in your regular repositories. 

There are other options if you need to make your VM persistent in case of re-creation. Those ones require configuring persistent storage to be managed by the Kubernetes cluster, such as [persistentVolumeClaim](https://kubevirt.io/user-guide/#/creation/disks-and-volumes?id=persistentvolumeclaim) or [dataVolume](https://kubevirt.io/user-guide/#/creation/disks-and-volumes?id=datavolume). More information on how to import our custom QCOW2 image in our cluster and make the changes to persist can be found in [CDI-Datavolumes blog post](https://kubevirt.io/2018/CDI-DataVolumes.html).

## Adapt the VM far beyond with cloud-init

Apart from the basic [cloud-init](https://cloudinit.readthedocs.io/en/latest/) functionality, in the current scenario there is an additional demand from Q&A department to automate the deployment of a fresh custom-built image along with the application to be reviewed.

> note "Note"
> cloud-init package was included during tailoring of the golden image in [Image tailoring with virt-customize section, from part 1](http://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#image-tailoring-with-virt-customize)
 
An [example](/assets/2020-03-18-Customizing-images-for-containerized-vms-2/vm-devstation-8-php.yaml) of an automated installation and configuration of a PHP application during VM start-up is shown below. More examples of cloud-init configuration are found at [cloud-init documentation](https://cloudinit.readthedocs.io/en/latest/topics/examples.html).

```yaml
apiVersion: kubevirt.io/v1alpha3
kind: VirtualMachine
metadata:
  name: qa-php-app1
spec:
  running: true 
  template:
    metadata:
      labels: 
        kubevirt.io/size: small
        kubevirt.io/domain: qa-php-app1
    spec:
      domain:
        devices:
          disks:
            - name: containerdisk
              disk:
                bus: virtio
            - name: cloudinitdisk
              disk:
                bus: virtio
          interfaces:
          - name: default
            bridge: {}
        resources:
          requests:
            cpu: 1
            memory: 2Gi
      networks:
      - name: default
        pod: {}
      volumes:
        - name: containerdisk
          containerDisk:
            image: image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-gui
            imagePullPolicy: IfNotPresent
        - name: cloudinitdisk
          cloudInitNoCloud:
            userData: | 
              #cloud-config
              users:
                - name: qa
                  ssh-authorized-keys:
                    - ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQCu6ebpjHClLxurBN3csEZNbIo3493cqFZGCnpicdeipIWVcU+Hdvy0/CIiZ8AQvcNsaC44zC9PYlHnoFWIfxrGvPvZikUC/qv4xmzNZa60Ype4M3tyMUXlP+vbDu3sOgeIWhyiyvV7NN2DGYF9mi0eKV5u/awS2B/pL6KXPjXjwVZNFB9aur9DkVClgAHc+18pI11RMMuUK4oOPsdopvECgZoWK/zuXrCJQFmqMqlArgveOhGU1FGUpeG63lsf38Xe2+9EXhhcMMuLcYN2BxOwdyRcHt0uflI3SagufUHbBA0vLF4XeTaoU+X2NFM5ToQDdP5Aa+s0mKgGTR9SpknT ansible-generated on eko1.cloud.lab.eng.bos.redhat.com 
                  lock_passwd: false
                  sudo: ALL=(ALL) NOPASSWD:ALL
                  groups: users, wheel
              packages:
                - git
                - php-cli
                - php-zip
                - wget
                - unzip
                - php-json
                - curl
              runcmd:
                - "export HOME=/root && curl -sS https://getcomposer.org/installer | php"
                - mv composer.phar /usr/bin/composer
                - chmod +x /usr/bin/composer
                - git clone https://github.com/PHP-DI/demo.git /var/www/html/demo
                - cd /var/www/html/demo && /usr/bin/composer install -vvv
                - sed -i 's_DocumentRoot "/var/www/html"_DocumentRoot "/var/www/html/demo/web/"_' /etc/httpd/conf/httpd.conf
                - systemctl restart httpd
              write_files:
                - content: |
                    1. Verify accessing to Apache root is showing the app
                    2. Verify that you can create new items
                    3 .....
                  path: /var/www/html/index.html
                  permissions: "0644"
```

<br>

Following this manifest and the specific cloud-init section, Q&A is able to:

* Add a specific *qa* user with administrator privileges. The user is configured to be accesses through SSH with public key authentication.
* Install the packages required by the php-app1 on top of the custom-built VM.
* Configure and run the application by cloning the source code and deploying it. The web server, which is already in place, is also configured.
* Include some files to the custom-built containerized image that help QA members to obtain the test-cases indispensable to run. 

```sh
$ oc create -f vm-devstation-8-php.yml 
virtualmachine.kubevirt.io/qa-php-app1 created
```

All the tasks configured in the cloud-init section can be viewed connecting to the serial console:

```sh
$ kubectl virt console qa-php-app1 
...
[  961.962827] cloud-init[1685]: Package php-cli-7.2.11-2.module_el8.1.0+209+03b9a8ff.x86_64 is already installed.
[  962.034708] cloud-init[1685]: Package wget-1.19.5-8.el8_1.1.x86_64 is already installed.
[  962.039763] cloud-init[1685]: Package unzip-6.0-41.el8.x86_64 is already installed.
[  962.050719] cloud-init[1685]: Package curl-7.61.1-11.el8.x86_64 is already installed.
[  964.546032] cloud-init[1685]: Dependencies resolved.
[  964.879004] cloud-init[1685]: ================================================================================
[  964.884630] cloud-init[1685]:  Package          Arch   Version                                Repo       Size
[  964.893672] cloud-init[1685]: ================================================================================
[  964.908842] cloud-init[1685]: Installing:
[  964.920010] cloud-init[1685]:  git              x86_64 2.18.2-1.el8_1                         AppStream 186 k
[  964.929811] cloud-init[1685]:  php-json         x86_64 7.2.11-2.module_el8.1.0+209+03b9a8ff   AppStream  73 k
[  964.938800] cloud-init[1685]:  php-pecl-zip     x86_64 1.15.3-1.module_el8.1.0+209+03b9a8ff   AppStream  51 k
[  964.947680] cloud-init[1685]: Installing dependencies:
[  964.955839] cloud-init[1685]:  git-core         x86_64 2.18.2-1.el8_1                         AppStream 4.8 M
[  964.960304] cloud-init[1685]:  git-core-doc     noarch 2.18.2-1.el8_1                         AppStream 2.3 M
[  964.971474] cloud-init[1685]:  libzip           x86_64 1.5.1-2.module_el8.1.0+209+03b9a8ff    AppStream  63 k
[  964.980428] cloud-init[1685]:  perl-Error       noarch 1:0.17025-2.el8                        AppStream  46 k
[  964.988807] cloud-init[1685]:  perl-Git         noarch 2.18.2-1.el8_1                         AppStream  77 k
[  964.996988] cloud-init[1685]:  perl-TermReadKey x86_64 2.37-7.el8                             AppStream  40 k
[  965.007575] cloud-init[1685]: Transaction Summary
[  965.017653] cloud-init[1685]: ================================================================================
[  965.026894] cloud-init[1685]: Install  9 Packages
[  965.050452] cloud-init[1685]: Total download size: 7.6 M
[  965.072707] cloud-init[1685]: Installed size: 43 M
[  965.108310] cloud-init[1685]: Downloading Packages:
[ 1007.877951] cloud-init[1685]: [MIRROR] git-2.18.2-1.el8_1.x86_64.rpm: Curl error (28): Timeout was reached for http://mirrors.seas.harvard.edu/centos/8.1.1911/AppStream/x86_64/os/Packages/git-2.18.2-1.el8_1.x86_64.rpm [Operation too slow. Less than 1000 bytes/sec transferred the last 30 seconds]
....
[ 1221.904206] cloud-init[2191]: All settings correct for using Composer
[ 1222.014342] cloud-init[2191]: Downloading...
[ 1224.481017] cloud-init[2191]: Composer (version 1.10.0) successfully installed to: //composer.phar
[ 1224.501585] cloud-init[2191]: Use it: php composer.phar
[ 1224.810570] cloud-init[2191]: Cloning into '/var/www/html/demo'...
[ 1229.232157] cloud-init[2191]: Reading ./composer.json
[ 1229.281799] cloud-init[2191]: Loading config file ./composer.json
[ 1229.374687] cloud-init[2191]: Checked CA file /etc/pki/ca-trust/extracted/pem/tls-ca-bundle.pem: valid
[ 1229.523000] cloud-init[2191]: Executing command (/var/www/html/demo): git branch --no-color --no-abbrev -v
[ 1230.202824] cloud-init[2191]: Failed to initialize global composer: Composer could not find the config file: /root/.composer/composer.json
[ 1230.209904] cloud-init[2191]: To initialize a project, please create a composer.json file as described in the https://getcomposer.org/ "Getting Started" section
[ 1230.262812] cloud-init[2191]: Running 1.10.0 (2020-03-10 14:08:05) with PHP 7.2.11 on Linux / 4.18.0-147.5.1.el8_1.x86_64
[ 1230.996433] cloud-init[2191]: Loading composer repositories with package information
...
ci-info: +++++++++++++++++++++++++++Authorized keys from /home/alosadag/.ssh/authorized_keys for user alosadag+++++++++++++++++++++++++++
ci-info: +---------+-------------------------------------------------+---------+--------------------------------------------------------+
ci-info: | Keytype |                Fingerprint (md5)                | Options |                        Comment                         |
ci-info: +---------+-------------------------------------------------+---------+--------------------------------------------------------+
ci-info: | ssh-rsa | 2f:3e:40:6c:e1:4f:0f:69:3d:c5:dc:ec:2d:22:8c:60 |    -    | ansible-generated on eko1.cloud.lab.eng.bos.redhat.com |
ci-info: +---------+-------------------------------------------------+---------+--------------------------------------------------------+
<14>Mar 10 16:19:35 ec2: 
<14>Mar 10 16:19:35 ec2: #############################################################
<14>Mar 10 16:19:35 ec2: -----BEGIN SSH HOST KEY FINGERPRINTS-----
<14>Mar 10 16:19:35 ec2: 256 SHA256:sTH9QQaaWvPBDaZxGvy8GXOwPqhCvmtVuGPU1O4PoIk no comment (ECDSA)
<14>Mar 10 16:19:35 ec2: 256 SHA256:DWXt+NbgCxjsEBvjSQ5H73JTO9TSlhuFdX06vCUvVc0 no comment (ED25519)
<14>Mar 10 16:19:35 ec2: 3072 SHA256:zwyYqMAstIbqFGPrVVZdZq1hnAr6HSUT1P4FehP23vM no comment (RSA)
<14>Mar 10 16:19:35 ec2: -----END SSH HOST KEY FINGERPRINTS-----
<14>Mar 10 16:19:35 ec2: #############################################################
-----BEGIN SSH HOST KEY KEYS-----
ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBObcueck5ULJqY7itWSTlwSHzeOoVTOK/FtvwGZi+SSVbglkLfE9PqpzgNJiSzIOoTs8pmqc+/794+Xq6StzP4U= 
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIEWsPRspY5q319gjsiGZSevHIcjZAPcLBxzQNW1mzw+c 
ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCsqsdMnZzWfN0nVL7MW80tzcIp4gwDEz8xC5XhGt8w7REnEOqSc+OGH6JsOiT3nJgH1MY5hha/NtpR3Ix4F9DQEk+enQLImx2u1sHxRUL0BzGjtu8Eg1X1MtWRU1jyvI7nsnx/osCgTODEhyz3Fj2NldJhWRWeSklXsa5lUuFybOOKhditpBV8GIQ+ovnkwWtN9dKFE4ecwlgGubJawrsN3iw2cU6XRV0siH8UjDC6On4ToY2ZMCnB4roh3Uw/CxAIyunXKxoz1Rx1RT91CBQppvfd3BNwluWXkif/QvgvpDdCNN91brZ8vklVzPNne0IfsWK5wbJEE5GgH32LAl6qSAIDG0aZkPuN9auxoEGOBERd+Avktxv/0QqeFpjci306bmH6z7TGxmAkd4hFcUQa8cbKhe3XuZnkXuUdHjVNnTnQtVvANfhNSt4lwG7HmBPfNsmZimVJ8RpwwZPKK6s0oD/og3SVn7+h57slIeGqiOZVJgmJ3OezJSbphIRmCfs= 
-----END SSH HOST KEY KEYS-----
[ 1344.046458] cloud-init[2191]: Cloud-init v. 18.5 running 'modules:final' at Tue, 10 Mar 2020 15:17:29 +0000. Up 1216.75 seconds.
```

<br>

> info "Information"
> It is important to observe that users have the ability to add missing data or code to the VM when starting-up. It is worth mentioning that this may make the boot process longer since cloud-init needs to fetch the data first.

Realize that Q&A or even the developers can take advantage of KubeVirt coupled with cloud-init to adapt a standardized image to the test or application needs.


## Summary

In this article, it has been explained how a custom-built containerized image can be deployed into our Kubernetes cluster. In any case, this is not the end for the developers, it is just the start point. Once they are capable of deploying the developer environment (VM), they need to learn how it can be accessed. In this blog post, it has been discussed about different choices such as connecting from the OKD web user interface, exposing the SSH service using a nodePort service type, using virtctl command-line or connecting to an exposed interface of the VM formerly configured by Multus.

Lastly, a brief indication of how a developer or Q&A member can adapt the golden container image was described. This adjustment could mean simply add disks to the VM or more complex changes leveraging the cloud-init capabilities.

In conclusion, these series of blog posts have presented a real use case where a company benefits from KubeVirt to create standard environments. They are used by developers to test their code before committing and also by the Q&A team to verify the proper behaviour of the application on clean environments. And even more relevant. both teams are autonomous to create the expected infrastructure to do their job.


## References

* [Customizing images for containerized VMs, part 1]({% post_url 2020-03-11-Customizing-images-for-containerized-vms %})
* [OpenShift 4.3 official documentation](https://docs.openshift.com/container-platform/4.3/welcome/index.html)1






