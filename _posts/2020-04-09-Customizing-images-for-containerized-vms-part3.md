---
layout: post
author: Alberto Losada Grande
description: This third part focuses on adjust the custom-built containerized image to end users and business needs. Deploying and accessing to the VMs is just the beginning since our users need to know how a standard VM can be adapted to their specific requirements.
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
title: Customizing images for containerized VMs (3/3)
pub-date: April 9
pub-year: 2020
---

<!-- TOC depthFrom:2 insertAnchor:false orderedList:false updateOnSave:true withLinks:true -->

- [Previously on customizing images for containerized VMs](#previously-on-customizing-images-for-containerized-vms)
- [Add disks to the custom-built VM](#add-disks-to-the-custom-built-vm)
    - [emptyDisk (ephemeral)](#emptydisk-ephemeral)
    - [hostDisk (persistent)](#hostdisk-persistent)
    - [Other options](#other-options)
- [Adapt the VM far beyond with cloud-init](#adapt-the-vm-far-beyond-with-cloud-init)
- [Summary](#summary)
- [References](#references)

<!-- /TOC -->


## Previously on customizing images for containerized VMs

In the previous article [Customizing images for containerized VMs, part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}) it was explained the vision of a software company, which wanted the Software Engineering team to be autonomous to deploy, run and destroy standardized VMs in the current Kubernetes infrastructure. On top of these custom-built virtual machines the necessary tests and validations needs to be executed before commiting the code to the respective Git branch.  From the Systems Engineering department point of view, there is no longer the need to maintain two different infrastructures to run containerized workloads and virtual machines.

Throughout the second blog post [Customizing images for containerized VMs part 2]({% post_url 2020-04-02-Customizing-images-for-containerized-vms-part2 %}), the process of containerizing the custom-built QCOW2 image was detailed. Additionally, the reader could follow how KubeVirt handles the deployment of a containerized VM into the corporate OKD Kubernetes cluster and the different options end users have to access the VM and do their work.

During this article, the custom-built container images stored in the OKD container registry are going to be deployed to show how they can be adapted to our specific needs. Actually, there are some situations where an application requires much storage than it is configured in the container image (10 GB) where KubeVirt's _emptyDisk_ and _hostDisk_ features come in handy. In our specific use case, there is a need from the Quality Assurance team to automate the deployment of the applications that needs to be tested by the team. Automating the deployment and configuration of the application at VM start-up allow the team to focus on their verification tasks. Certainly it opens the door to automate the whole quality process.

At this point where the tasks detailed in [part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}) and [part 2]({% post_url 2020-04-02-Customizing-images-for-containerized-vms-part2 %}) were achieved, the following container images are stored in the openshift namespace, so they are available to all authenticated OKD Kubernetes users. 

```sh
$ oc get imagestream devstation -n openshift
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
> In case you did not build the custom container images as detailed in the [Customizing images for containerized VMs part 2]({% post_url 2020-04-02-Customizing-images-for-containerized-vms-part2 %}) you still can download them from my [public container image repository](https://quay.io/repository/alosadag/devstation?tab=tags) at quay.io. 

Whether you want some hands-on experience while reading this write up, you can just copy the public container images from quay.io container registry to your private container registry using tools like `skopeo`:

> info "Information"
> The process of copying images to OKD container registry was explained in the section [store the image in the container registry](https://kubevirt.io/2020/Customizing-images-for-containerized-vms-part2.html#store-the-image-in-the-container-registry) from the previous article [Customizing images for containerized VMs, part 2]({% post_url 2020-04-02-Customizing-images-for-containerized-vms-part2 %})

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


## Add disks to the custom-built VM

Presumably, there is a chance that users need more disk space for the VMs deployed from the custom-built images. In [Customizing images for containerized VMs part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %}), QCOW2 custom-built images were [already expanded](http://kubevirt.io/2020/Customizing-images-for-containerized-vms.html#image-creation-with-builder-tool) to 10 GB size in order to meet the Software Engineering team demands. Despite that, an extra disk can be desirable in cases where there is a requirement to install a large number of packages or import massive data.


### emptyDisk (ephemeral)

In cases where there is a need for extra storage, the developers can take advantage of the [emptyDisk](https://kubevirt.io/user-guide/#/creation/disks-and-volumes?id=emptydisk) feature. It adds an extra sparse QCOW2  **ephemeral** disk to the VM as long as it exists. This means that the data stored in the emptyDisk will survive guest side reboots, but not VM re-creation.

> info "Information"
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

A [YAML VirtualMachine manifest](/assets/2020-03-18-Customizing-images-for-containerized-vms-2/vm-devstation-8-hostdisk.yaml) of running a devstation with a _hostDisk_ attached is presented below. 

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

The result of applying the YAML manifest is stated below. Observer that the VM (devstation-8-hostdisk) was deployed with an extra block device of 1Gi (dev/vdc).
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

As it has been explained along these series of blog posts, the company does not need to persist the VM. However, there are some cases where it is necessary to copy some libraries or files from the internal shared fileserver or viceversa. In those cases, a shared volume can be mounted in our VM running on top of OKD Kubernetes. Below, it is shown how a 50G shared volume from the corporate NFS server is mounted inside the VM (devstation-8-gui):

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

<br>
All the tasks configured in the cloud-init section are executed at VM start-up. It is possible to see them when connecting to the VM serial console:

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

* [Customizing images for containerized VMs, part 1]({% post_url 2020-03-26-Customizing-images-for-containerized-vms %})
* [OpenShift 4.3 official documentation](https://docs.openshift.com/container-platform/4.3/welcome/index.html)1






