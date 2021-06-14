---
title: "Building a Ceph-powered Cloud"
date: "2020-05-28"
author: "admin"
tags: 
  - "planet"
---

_Deploying a containerized Red Hat Ceph Storage 4 cluster for Red Hat Open Stack Platform 16_

_with John Fulton (Red Hat) and Gregory Charot (Red Hat)_

Ceph is the most popular storage backend for OpenStack by a wide margin, as has been reported by the OpenStack Foundation’s [survey](https://www.openstack.org/analytics) every year since its inception. In the latest survey, conducted during the Summer of 2019, Ceph outclassed other options by an even greater margin than it did in the past, with a 75% adoption rate.

[![image2_31.png](images/4puqLC7cBrf19i1K9XAnVh0xspap_small.png)](https://svbtleusercontent.com/4puqLC7cBrf19i1K9XAnVh0xspap.png)

Red Hat’s default method for installing OpenStack is with a tool called [director](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/director_installation_and_usage/chap-introduction). This tool can configure OpenStack to connect with an existing Ceph cluster, or it can deploy a new Ceph cluster as part of the process to create a new OpenStack private cloud. Because director integrates ceph-ansible, the same deployment options described in our previous [post](https://www.redhat.com/en/blog/deploying-containerized-red-hat-ceph-storage-4-cluster-using-ceph-ansible) dedicated to it remain available from director. This post will examine these integrated capabilities, including new features for greater flexibility which were introduced with the latest versions.

# Director in a nutshell [#](#director-in-a-nutshell_1)

Director can deploy an OpenStack private cloud backed by Red Hat Ceph Storage if it is provided the necessary configuration information to do so. Files defining the following configuration are required input:

1. Hardware definition
2. OpenStack end state definition
3. Ceph cluster end state definition or information necessary to connect to an existing Ceph Storage cluster

Director requires a single dedicated system called the undercloud which is used to deploy and manage one or more OpenStack clusters, which are referred to as overclouds. To deploy OpenStack with director, we run commands like openstack overcloud deploy on the undercloud system. Director will then deploy an OpenStack cloud using its own defaults, excepting any default settings which were overridden. By default, only one controller and one compute node are deployed, but those defaults can be easily changed to configure as many compute nodes as you have or provision other types of nodes, including Ceph storage nodes running Ceph’s Object Storage Daemon (OSD). The [blog we dedicated to ceph-ansible](https://www.redhat.com/en/blog/deploying-containerized-red-hat-ceph-storage-4-cluster-using-ceph-ansible) recommends designating a “host, or virtual machine, as the ansible ‘controller’ or administration host — providing a separate management plane.” When deploying OpenStack with Ceph, this single host is the undercloud.

The hardware definition input file should contain [IPMI](https://www.intel.com/content/www/us/en/products/docs/servers/ipmi/ipmi-second-gen-interface-spec-v2-rev1-1.html) information, which is used by director’s [baremetal tooling](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/director_installation_and_usage/creating-a-basic-overcloud-with-cli-tools) to power-cycle the servers, PXE boot them, and install Red Hat Enterprise Linux on them. [Ironic](https://wiki.openstack.org/wiki/Ironic), the upstream project providing these tools, can also wipe the disks before deployment; this should be done to ensure the disks are ready to be initialized as storage for Ceph OSDs. To enable this option set `clean_nodes=True` in the `undercloud.conf` file.

The OpenStack end state definition is defined in [Heat](https://wiki.openstack.org/wiki/Heat) environment files. We enable different features in the overcloud by including environment files with the `-e` operator. For example, the next image shows the commands to deploy one controller and one compute and trigger the ceph-ansible client role to configure them to use to an [existing Ceph RBD storage pool](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/integrating_an_overcloud_with_an_existing_red_hat_ceph_cluster/index):

[![image3_23_0.png](images/rHV8yZduL4GnKKgute2cRk0xspap_small.png)](https://svbtleusercontent.com/rHV8yZduL4GnKKgute2cRk0xspap.png)

The `environments` directory can be found in `/usr/share/openstack-tripleo-heat-templates/` on the undercloud. Because new versions of these files are shipped with updates it’s easier to override the default values contained in these files, with a separate overrides file, than modify them directly. In this example, the `ceph-ansible-external.yaml` file tells director to use the ceph-ansible client role because the pre-existing Ceph cluster is external to it.

The `~/containers-prepare-parameters.yaml` file is generated by the `openstack tripleo container image prepare` command, which is run when the undercloud is [being configured](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/director_installation_and_usage/preparing-for-director-installation). Because the OpenStack services themselves run in containers, this file is used to define where the registry of container images is located. When Ceph is [deployed by director](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/deploying_an_overcloud_with_containerized_red_hat_ceph/index), this file specifies where to get the Ceph Storage container images.

The `~/my-overrides.yaml` file stores configuration information about the Ceph cluster, including its FSID, the IP addresses or hostnames of the monitor nodes, and a CephX keyring which has been configured to access the RBD pools for Nova, Cinder, and Glance, OpenStack’s respective Compute, Volume, and Image services. If these pools on the existing Ceph cluster are not named VMs, volumes, and images respectively, these defaults can also be overridden. An example of `~/my-overrides.yaml` may contain something like this:

```
parameter_defaults:
  # The cluster FSID
  CephClusterFSID: '4b5c8c0a-ff60-454b-a1b4-9747aa737d19'
  # The CephX user auth key
  CephClientKey: 'AQDLOh1VgEp6FRAAFzT7Zw+Y9V6JJExQAsRnRQ=='
  # The list of Ceph monitors
  CephExternalMonHost: '172.16.1.7, 172.16.1.8, 172.16.1.9'
```

This assumes that the administrator of the Ceph cluster has run a command like this example, and shared the resulting client key:

```
$ ceph auth add client.openstack mon 'allow r' osd 'allow class-read object_prefix rbd_children, allow rwx pool=vms, allow rwx pool=volumes, allow rwx pool=images'
```

In Red Hat OpenStack platform 13 and earlier Heat orchestrated the entire deployment, but Heat’s role has since changed. While it is still the authoritative source of what the configuration should be, it no longer has a role in how that configuration is applied.

Instead, director outputs Ansible playbooks. The operator has the option to either run those playbooks directly with the `ansible-playbook` command, or letting director run the playooks automatically after they are generated. The latter is the default so that a single command may still be used to deploy an overcloud, as was described in [James’ blog](https://www.redhat.com/en/blog/greater-control-red-hat-openstack-platform-deployment-ansible-integration). Either way, the config-download directory (which defaults to `/var/lib/mistral/config-download-latest`) will contain a `ceph-ansible` directory that director will create and it will look like the `ceph-ansible` directory described in the [blog we dedicated to ceph-ansible](https://www.redhat.com/en/blog/deploying-containerized-red-hat-ceph-storage-4-cluster-using-ceph-ansible).

It will also contain a `ceph_ansible_command.log` file with the output of each ansible-playbook run.

# Hyperconvergence [#](#hyperconvergence_1)

So far we’ve covered how to deploy what we could call the default cloud (one OpenStack Controller and one OpenStack Compute) and have them use an existing Ceph cluster by triggering the ceph-ansible client role.

Next, we will deploy a new OpenStack controller running co-located on the same host with the Ceph Monitor (MON) and Manager (MGR) daemons, and a new OpenStack Compute node, which also runs a co-located Ceph OSD daemon. In this case, rather than using an existing Ceph cluster external to OpenStack, we are creating a new one and running it on the same hardware.

This kind of service co-location is referred to as hyperconverged infrastructure, and a variation of this example may be used to deploy Red Hat [Hyperconverged Infrastructure for Cloud](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/hyperconverged_infrastructure_guide/index), which is based on the same underlying Ceph and OpenStack technologies.

Director uses the concept of roles to define which set of services are going to be configured in each server. A predefined role called ComputeHCI has all of the services necessary to run a hyperconverged Nova compute and Ceph OSD server. To have director create a roles file with the Controller and ComputeHCI roles which will be used in the next example run the following:

```
openstack overcloud roles generate Controller ComputeHCI -o ~/my-roles.yaml
```

Because the ComputeHCI role is not deployed by default, (it’s default node count is zero), we need to override the default node counts by creating the file `~/my-counts.yaml` with the following contents:

```
parameter_defaults:
  ControllerCount: 1
  ComputeHCICount: 1
```

This example is deploying one Controller and one ComputeHCI node. Technically we don’t need to specify ControllerCount since it already defaults to 1. In a production deployment you would have at least three of each node type for six total nodes. This ensures there are the required three Ceph monitors and three OSD servers, but this is just a simple example to get you started.

# All your options are belong to us [#](#all-your-options-are-belong-to-us_1)

In other words, anything ceph-ansible can do, OpenStack director can do as well — and using the same underlying mechanism, so operators don’t have to learn two distinct ways to accomplish the same task. The [ceph-ansible blog](https://www.redhat.com/en/blog/deploying-containerized-red-hat-ceph-storage-4-cluster-using-ceph-ansible) described how example variables can define which Ansible roles will be assigned to each host. For example, the `group_vars/osds.yml` file has a device list which refers to each disk to be configured as an OSD store. When using director, the very same variables can be configured from Heat environment files, as in this example:

```
parameter_defaults:
  CephAnsibleExtraConfig:
    dmcrypt: true
  CephAnsibleDisksConfig:
    devices:
      - /dev/sdb
      - /dev/sdc
```

Variables normally set in `ceph-ansible/group_vars` can be overridden under the CephAnsibleExtraConfig key. In addition to overriding any arbitrary ceph-ansible variable, you may also add lines to any section of `/etc/ceph/ceph.conf` using `CephConfigOverrides` like this:

```
parameter_defaults:
  CephConfigOverrides:
    global:
      max_open_files: 131072
    osd:
      osd_scrub_during_recovery: false
```

If the above were saved to `~/my-overrides.yaml` with the role and count information already discussed in the previous section, the following would deploy a new OpenStack cloud along with its very own hyperconverged Ceph cluster.

[![image1_44.png](images/r2ch4BangMNNNcFY13YxP90xspap_small.png)](https://svbtleusercontent.com/r2ch4BangMNNNcFY13YxP90xspap.png)

# Controlling which Playbooks are run [#](#controlling-which-playbooks-are-run_1)

Director reasserts the configuration definition on the private cloud each time the deployment command is run. This is useful if you want to change one setting on all of the nodes in your private cloud. This type of operation is also known as a stack update.

Because the `site-container.yml` ceph-ansible playbook is idempotent, and because a Ceph cluster can restart an OSD without a service interruption, this reassertion does not cause any problems. However, if you are confident that the Ceph configuration doesn’t need to be changed during a stack update and would rather not take the extra time to reassert the Ceph configuration, then the Ceph services within Heat can be “no op’d” as described in our knowledge base articles [Update Overcloud without re-configuring Ceph clients](https://access.redhat.com/solutions/4939291) or [Update Overcloud without re-configuring Ceph server](https://access.redhat.com/solutions/4963041).

It’s also possible to do the inverse and run a stack update which only changes the Ceph configuration and doesn’t change any OpenStack configuration; this is achieved by using Ansible tags. [James’ blog](https://www.redhat.com/en/blog/greater-control-red-hat-openstack-platform-deployment-ansible-integration) covers how to apply director configuration by running the ansible-playbook command directly and how to pass tags. If you pass `--tags external_deploy_steps` to the same command, then only the “external deploy scripts” run, which for the ceph-ansible integration, means run the Ansible which generates the ceph-ansible inventory and then executes ceph-ansible.

Other tasks besides Ceph configuration fall under the external deploy scripts framework like the container image preparation mentioned earlier so you might wish to combine `—tags` and `--skip-tags` like this `--tags external_deploy_steps --skip-tags container_image_prepare`. Finally, if you’re at the point where you’re running `ansible-playbook` and don’t want to “no-op” in Heat, then you can also use `--skip-tags run_ceph_ansible` to have Ansible apply OpenStack updates but not re-run ceph-ansible.

# Lifecycle Management [#](#lifecycle-management_1)

As described above, director can help you deploy and manage both OpenStack Platform and Ceph Storage from a single unfined deployment and management system. But what about updates and upgrades?

When deploying Ceph Storage directly from OpenStack Platform’s director, the two products’ major versions are aligned to ensure full compatibility and support alignment over time. If you choose to deploy OpenStack Platform 13, you would get a Ceph Storage 3 cluster, with OpenStack Platform16 you get a Ceph Storage 4 cluster.

Director can manage both minor updates in order to get the latest Ceph Storage updates as well as major upgrades. Major upgrades are performed at the same time as OpenStack Platform upgrades, for example if you upgrade Red Hat OpenStack Platform from version 13 to 16, director will also upgrade Ceph Storage from version 3 to 4.

The process uses the same philosophy, employing ceph-ansible under the hood to configure Ceph Storage while exposing a single management tool to the operator, which will manage any other necessary configuration changes, such as operating system upgrades.

# Just getting started [#](#just-getting-started_1)

OpenStack director is a remarkably powerful tool which can be used to completely control the integration of cloud infrastructure — and we have barely scratched the surface of what is possible to accomplish with it.

Director can replace the default OpenStack Object Storage service ([Swift](https://wiki.openstack.org/wiki/Swift)) with the compatible Ceph Object Gateway (RGW) in a single, streamlined command. The other services that normally use Swift to store their configuration will start using the Ceph Object Gateway instead [without further configuration](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html/deploying_an_overcloud_with_containerized_red_hat_ceph/index), most notably the Keystone identity service.

Director can similarly configure the OpenStack File service ([Manila](https://wiki.openstack.org/wiki/Manila)) with a [CephFS](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.0/html-single/cephfs_via_nfs_back_end_guide_for_the_shared_file_system_service/index) backend. The result is an OpenStack cluster where storage needs are delivered from a single, scalable Ceph Storage cluster. This can simplify operational management by de-duplicating not just operational management activities, but capacity planning, backups and the rest that is required to maintain multiple storage silos. Just one more reason why modern scale-out compute needs to be paired with scale-out storage.

# What Next? [#](#what-next_1)

The integration between OpenStack Platform and Ceph Storage is constantly being enhanced, evolving with additions that expose brand new Ceph Storage features such as the Manager Dashboard, but also (and even more importantly) new ways to deploy it.

The upcoming OpenStack Platform 16.1 release will allow you to deploy multiple Ceph Storage clusters per OpenStack Platform deployment. This is a highly awaited feature necessary to support deployments at the edge of the network in OpenStack Platform’s Distributed Compute Nodes architecture with Ceph Storage providing storage at the edge. Support for deploying multiple Ceph clusters allows OpenStack Platform to deploy a dedicated Ceph Storage cluster at each remote edge site to bring the storage close to the workload without any added network latency or cluster stretching.

This functionality can also be used for fielding hybrid private cloud deployments in configurations where the operator chooses to deploy a separate Ceph cluster in each Availability Zone.

# Do try this at home [#](#do-try-this-at-home_1)

Red Hat Ceph Storage 4.1 [is available from Red Hat’s website](https://access.redhat.com/downloads/content/281/ver=4/rhel---8/4.0/x86_64/product-software). Try out your hand at distributed storage today!

Cross-posted to the [Red Hat Blog](https://www.redhat.com/en/blog/building-ceph-powered-cloud-deploying-containerized-red-hat-ceph-storage-4-cluster-red-hat-open-stack-platform-16?source=blogchannel&channel=blog/channel/red-hat-storage).

Source: Federico Lucifredi ([Building a Ceph-powered Cloud](https://f2.svbtle.com/building-a-ceph-powered-cloud))