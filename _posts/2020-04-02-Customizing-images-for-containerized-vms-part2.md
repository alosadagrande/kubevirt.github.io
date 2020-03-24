---
layout: post
author: Alberto Losada Grande
description: This second part deals with the containerization process of the custom images built in the previous blog post. It details the procedure to upload and store them into a container image registry and how can be deployed by developers and users.
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
title: Customizing images for containerized VMs (2/3)
pub-date: April 2
pub-year: 2020
---

<!-- TOC depthFrom:2 insertAnchor:false orderedList:false updateOnSave:true withLinks:true -->

- [Previously on customizing images for containerized VMs](#previously-on-customizing-images-for-containerized-vms)
- [Image containerization procedure](#image-containerization-procedure)
    - [Store the image in the container registry](#store-the-image-in-the-container-registry)
- [Create a VM from the container image](#create-a-vm-from-the-container-image)
- [Connect to the deployed custom-built VM](#connect-to-the-deployed-custom-built-vm)
    - [OKD web user interface](#okd-web-user-interface)
    - [Console serial or VNC using virtctl](#console-serial-or-vnc-using-virtctl)
    - [SSH using service nodePort](#ssh-using-service-nodeport)
    - [SSH using multus CNI](#ssh-using-multus-cni)
- [Summary](#summary)
- [References](#references)

<!-- /TOC -->


## Previously on customizing images for containerized VMs

In the previous article [Customizing images for containerized VMs, part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}) it was explained the vision of a software company, which wanted the Software Engineering team to be autonomous to deploy, run and destroy standardized VMs in the current Kubernetes infrastructure. On top of these custom-built virtual machines the necessary tests and validations need to be executed before commiting the code to the respective Git branch. 

Thus, developers can request a fresh standard environment to test their code and more important, use the same tooling (`kubectl`, `oc`) they utilize when running applications in the corporate OKD Kubernetes cluster. From the Systems Engineering department point of view, there is no longer the need to maintain two different infrastructures to run containerized workloads and virtual machines.

Throughout the previous blog post, the process of creating a *golden image* using tools such as _Builder Tool_ and _virt-customize_ was described. This blog post focuses on describing the containerization process of the custom-built image created in the previous article. Once the image is uploaded and stored in the OKD container registry our developers are able to deploy a standard environment _consistently_, based on the custom-built image created, where they can verify their code. 

## Image containerization procedure

The procedure to inject a *VirtualMachineInstance* disk into a container image is pretty well explained in [containerDisk Workflow example](https://kubevirt.io/user-guide/#/creation/disks-and-volumes?) from the official documentation. Notice that only _RAW_ and _QCOW2_ formats are supported and the disk is recommended to be placed into the /disk directory inside the container. Actually, it can be placed in other directories, but then, it must be explicitly configured when creating the *VirtualMachine*.

[Customizing images for containerized VMs, part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}) article ended up with 4 standardized images ready to be containerized. Since the process is the same for all of them, in order to keep it short, we are just going to describe the creation of a container image based on CentOS 8 QCOW2 images.

> info "Information"
> These were the four images built in [part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}): CentOS 8 with GNOME, CentOS 8 terminal only, CentOS 7 with GNOME and CentOS 7 terminal only.

```sh
$ cat Dockerfile
FROM scratch
ADD golden-devstation-centos8-disk-10G.qcow2 /disk/
```

```sh
$ cat Dockerfile
FROM scratch
ADD golden-devstation-centos8-disk-10G-gui.qcow2 /disk/
```

<br>

As it can be seen, the Dockerfile script is really simple. Additionally, remember to place the container image in the same folder as the Dockerfile. Below it is shown the building process. In our case, [podman](https://podman.io/) is in charge of executing the building task, however, [docker](https://docker.com) or [buildah](https://buildah.io/) could be used as well.

```sh
$ podman build . -t openshift/devstation-centos8:terminal
STEP 1: FROM scratch
STEP 2: ADD golden-devstation-centos8-disk-10G.qcow2 /disk/
STEP 3: COMMIT openshift/devstation-centos8:terminal
8a9e83db71f08995fa73699c4e5a2d331c61b393daa18aa0b63269dc10078467

$ podman build . -t openshift/devstation-centos8:gui
STEP 1: FROM scratch
STEP 2: ADD golden-devstation-centos8-disk-10G-gui.qcow2 /disk/
STEP 3: COMMIT openshift/devstation-centos8:gui
2a4ecc7bf9da91bcb5847fd1cf46f4cd10726a4ceae88815eb2a9ab38b316be4
```

<br>

The container images of a successful build are stored locally. In our case, saved in the _Builder Server_. Anyway, remember that they must be uploaded to the OKD container registry so that they are available to all the users.

```sh
$ podman images
REPOSITORY                               TAG        IMAGE ID       CREATED          SIZE
localhost/openshift/devstation-centos8   gui        2a4ecc7bf9da   3 minutes ago    5.72 GB
localhost/openshift/devstation-centos8   terminal   8a9e83db71f0   13 minutes ago   1.94 GB
```

<br>

### Store the image in the container registry

Before pushing the images to the corporate container registry, it must be verified that the OKD registry is available and reacheable outside the Kubernetes cluster network. Exposing the container registry allows any authenticated user to gain external access to push and pull images. [Exposing the secure registry](https://docs.openshift.com/container-platform/4.3/registry/securing-exposing-registry.html) consists basically on configuring a Route object and expose that Route in the OKD routers. Once done, external **authenticated** access is allowed.

```sh
$ oc get route -n openshift-image-registry
NAME            HOST/PORT                                                     PATH   SERVICES         PORT    TERMINATION   WILDCARD
default-route   default-route-openshift-image-registry.apps.okd.okdlabs.com          image-registry   <all>   reencrypt     None
```

> note "Note"
> In order to upload your containerized images to the OKD registry, the user must be authenticated and [authorized to execute the push action](https://docs.okd.io/latest/registry/securing-exposing-registry.html). The role that must be added to the OKD user is the _registry-editor_

In order to authenticate with the OKD container registry, podman is employed as explained in the [official documentation](https://docs.okd.io/latest/registry/accessing-the-registry.html).

```sh
$ oc login https://api.okd.okdlabs.com:6443 -u alosadag
The server uses a certificate signed by an unknown authority.
You can bypass the certificate check, but any data you send to the server could be intercepted by others.
Use insecure connections? (y/n): y

Authentication required for https://api.okd.okdlabs.com:6443 (openshift)
Username: alosadag
Password:
Login successful.

$ HOST=$(oc get route default-route -n openshift-image-registry --template='{{ .spec.host }}')
$ echo $HOST
default-route-openshift-image-registry.apps.okd.okdlabs.com

$  podman login -u $(oc whoami) -p $(oc whoami -t) --tls-verify=false $HOST
Login Succeeded!
```

<br>

Before pushing the images, adapt container images names so they can be uploaded to the proper OKD private registry. Since it is agreed that all developers must be able to pull the images into their Kubernetes namespaces, the images need to be pushed to the _openshift_ project. This project is special in a way that it is accessible from the rest of projects.

> info "Information"
> [Understanding containers, images and imagestreams](https://docs.okd.io/latest/openshift_images/images-understand.html) from OpenShift documentation deeply explains container image naming.

```sh
$ podman tag localhost/openshift/devstation-centos8:gui default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation:v8-terminal
$ podman push default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation:v8-terminal --tls-verify=false
```

```sh
$ podman tag localhost/openshift/devstation-centos:gui default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation:v8-gui
$ podman push default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation:v8-gui --tls-verify=false
Getting image source signatures
Copying blob 6b39f8837d66 [========>-----------------------------] 1.2GiB / 5.3GiB
Copying blob 6b39f8837d66 [========>-----------------------------] 1.2GiB / 5.3GiB
Copying blob 6b39f8837d66 [========>-----------------------------] 1.2GiB / 5.3GiB
Copying blob 6b39f8837d66 [========>-----------------------------] 1.2GiB / 5.3GiB
Copying blob 6b39f8837d66 [=============>------------------------] 1.9GiB / 5.3GiB
Copying blob 6b39f8837d66 [==============>-----------------------] 2.1GiB / 5.3GiB
Copying blob 6b39f8837d66 [===================>------------------] 2.7GiB / 5.3GiB
Copying blob 6b39f8837d66 [===================>------------------] 2.7GiB / 5.3GiB
Copying blob 6b39f8837d66 done
Copying config 2a4ecc7bf9 done
Writing manifest to image destination
Storing signatures
```

<br>

Verify that the images are stored correctly in the OKD container registry by checking the [imageStream](https://docs.okd.io/latest/openshift_images/image-streams-manage.html#working-with-imagestreams). As shown below, both images are uploaded effectively since the `devstation` imagestream contains two images with v8-gui and v8-terminal tags respectively.

```sh
$ oc describe imagestream devstation -n openshift
Name:			devstation
Namespace:		openshift
Created:		23 hours ago
Labels:			<none>
Annotations:		<none>
Image Repository:	default-route-openshift-image-registry.apps.okd.okdlabs.com/openshift/devstation
Image Lookup:		local=false
Unique Images:		2
Tags:			2

v8-gui
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/openshift/devstation@sha256:e301d935c1cb5a64d41df340d78e6162ddb0ede9b9b5df9c20df10d78f8fde0f
      2 hours ago

v8-terminal
  no spec tag

  * image-registry.openshift-image-registry.svc:5000/openshift/devstation@sha256:47c2ba0c463da84fa1569b7fb8552c07167f3464a9ce3b6e3f607207ba4cee65
```

<br>

At this point, the images are stored in a private registry and ready to be consumed by the developers.

> info "Information"
> In case you do not have a corporate private registry available, you can upload images to any free public container registry. Then, consume the container images from the public container registry instead. Just in case you want to use them or take a look, they were available from my [public container image repository at quay.io](https://quay.io/repository/alosadag/devstation?tab=tags)

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-03-26-Customizing-images-for-containerized-vms/okd_is_devstation.png"
      itemprop="contentUrl"
      data-size="1110x467"
    >
      <img
        src="/assets/2020-03-26-Customizing-images-for-containerized-vms/okd_is_devstation.png"
        itemprop="thumbnail"
        width="100%"
        alt="VM to VM"
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>


## Create a VM from the container image 

Now that the images are stored in the OKD container registry, it is time to create the *VirtualMachine* Kubernetes YAML definition that pulls the proper image and configures the virtual machine managed by KubeVirt. First, a new namespace is recommended to be created. Additionally, OKD is configured to allow members of the _A team_ to run their VMs and verifications in that namespace.

```sh
$ kubectl create ns dev-team-a
namespace/dev-team-a created
```
<br>

Below, it is shown the _VirtualMachine_ YAML manifest that deploys a CentOS 8 container image (without GNOME) as _containerDisk_ pulled from the internal registry: _https://image-registry.openshift-image-registry.svc:5000/openshift/devstation:v8-terminal_. Notice that the image is configured as a containerDisk volume.

> info "Information"
> More information regarding the containerization procedure and containerDisk can be found in [image containerization procedure from part 1](https://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#image-containerization-procedure)

Notice that the developers can request the adequate memory and CPU to run and test their applications. They are *resource* parameters from the *VirtualMachine* Custom Resource (CR). The YAML definition show below can be downloaded from [here](/assets/2020-04-02-Customizing-images-for-containerized-vms-2/vm-devstation-8.yaml). 

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

Below, the YAML manifest is applied in the proper namespace:

```sh
$ kubectl create -f vm-devstation-8.yaml -n dev-team-a
virtualmachine.kubevirt.io/devstation created
```
<br>

Next, several events are logged in the _dev-team-a_ namespace while the _VirtualMachineInstance_ is being started. See how the _VirtualMachine_ and _VirtualMachineInstance_ (VMI) are created. Additionally, observe how the virt-launcher and the custom-built image (devstation-8-terminal) are being pulled. Eventually, the _VirtualMachineInstance_ (devstation) is created and started:

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

At this stage, the virtual machine is ready to be used for testing the new code written by the developers. The next question that will arise from the users is... OK... **how can I connect to my VM and do the work?** 


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
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/centos8-terminal-okd.png"
      itemprop="contentUrl"
      data-size="1110x527"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/centos8-terminal-okd.png"
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
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/centos8-gui-okd.png"
      itemprop="contentUrl"
      data-size="1110x476"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/centos8-gui-okd.png"
        itemprop="thumbnail"
        width="100%"
        alt="CentOS 8 OKD console"
      />
    </a>
    <figcaption itemprop="caption description"></figcaption>
  </figure>
</div>

### Console serial or VNC using virtctl

The developers can use the _virtctl_ binary or the _virtctl_ plugin for [krew](https://krew.dev/) to connect from their laptop to the virtual machine running in the Kubernetes cluster. The installation is well documented in the [Krew quickstart guide](https://kubevirt.io/quickstart_minikube/#install-virtctl). Observe that _virtctl_ allows us to connect via serial or VNC console. 

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
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/virtctl-serial-console.png"
      itemprop="contentUrl"
      data-size="1110x488"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/virtctl-serial-console.png"
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
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/virtctl-vnc-console.png"
      itemprop="contentUrl"
      data-size="924x720"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/virtctl-vnc-console.png"
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

In order to connect through SSH from a workstation, a [service nodePort](https://kubernetes.io/docs/concepts/services-networking/service/#nodeport) is an option that can be used. A service nodePort is a service type within Kubernetes that exposes a service on each Node's IP at a static port. Thus, end users connect (ssh) to the static port of any Node in order to access to the desired VM. There is specific information in KubeVirt documentation about [how to expose a VMI](https://kubevirt.io/user-guide/#/usage/network-service-integration?id=expose-virtualmachineinstance-as-a-nodeport-service) in case you need further detail.

In the first place, a nodePort service that maps to the _VirtualMachineInstance_ SSH port (tcp/22) must be created and applied. The YAML service manifest shown below can be also downloaded from [here](/assets/2020-04-02-Customizing-images-for-containerized-vms-2/vm-svc-ssh-nodeport.yaml)

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
> OKD comes with Multus out of the box. A proper CNI configuration is just needed as detailed in the [CNI plug-in](https://docs.okd.io/latest/networking/multiple_networks/understanding-multiple-networks.html#additional-networks-provided) OpenShift documentation. In our environment, the [host-device](https://docs.okd.io/latest/networking/multiple_networks/configuring-host-device.html#configuring-host-device) plug-in was configured. 

As depicted in the diagram below, OKD Nodes were configured with two interfaces: eth0 and eth1, which correspond to service and management interfaces respectively. Eth1 (management interface) was configured to be used by the VMs running on top of Kubernetes to acquire an IP address (dhcp or static) from the corporate management network.

<div class="my-gallery" itemscope itemtype="http://schema.org/ImageGallery">
  <figure
    itemprop="associatedMedia"
    itemscope
    itemtype="http://schema.org/ImageObject"
  >
    <a
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/multus-diagram.png"
      itemprop="contentUrl"
      data-size="937x720"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/multus-diagram.png"
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
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/multus-okd-dialog.png"
      itemprop="contentUrl"
      data-size="1110x431"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/multus-okd-dialog.png"
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
      href="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/multus_okd.png"
      itemprop="contentUrl"
      data-size="1110x376"
    >
      <img
        src="/assets/2020-04-02-Customizing-images-for-containerized-vms-2/multus_okd.png"
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

## Summary

In this article, it has been explained how a custom-built QCOW2 image can be containerized to be consumed by KubeVirt and finally deployed into our Kubernetes cluster. In any case, this is not the end for the developers, it is just the start point. Once they are capable of deploying the developer environment (VM), they need to learn how it can be accessed. In this blog post, it has been discussed about different choices such as connecting from the OKD web user interface, exposing the SSH service using a nodePort service type, using virtctl command-line or connecting to an exposed interface of the VM formerly configured by Multus.


In the next and last article of this series [Customizing images for containerized VMs, part 3]({% post_url 2020-04-09-Customizing-images-for-containerized-vms-part3 %}) will describe how developers can adapt and adjust to their needs the standard VM deployed based on the custom-built image created in [Customizing images for containerized VMs, part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}). Actually, end users could need extra disk devices to be attached at start time depending on the application deployed. Additionally, could require automating tasks such as the configuration of an application during VM creation. These topics will be commented in detail in the next blog post of this series.


## References

* [Customizing images for containerized VMs, part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %})
* [OpenShift 4.3 official documentation](https://docs.openshift.com/container-platform/4.3/welcome/index.html)
* [OKD official documentation](https://docs.okd.io/latest/welcome/index.html)



