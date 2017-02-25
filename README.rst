================================
Object Storage (OpenStack Swift)
================================

General information about Openstack Swift can be found at:

    - https://wiki.openstack.org/wiki/Swift

This repository provides the following:

    - `Bill of Materials <https://github.com/open-power-ref-design/standalone-swift/blob/master/swift.pdf>`_
    - `Deployment configuration file for a small size cluster <https://github.com/open-power-ref-design/standalone-swift/blob/master/small-config.yml>`_
    - `Deployment configuration file for a medium size cluster <https://github.com/open-power-ref-design/standalone-swift/blob/master/medium-config.yml>`_
    - `Deployment configuration file for a large size cluster <https://github.com/open-power-ref-design/standalone-swift/blob/master/large-config.yml>`_

The Bill of Materials document provides a description and representation of Database
as a Service that is tuned for OpenPOWER servers.  It provides information
such as model numbers and feature codes to simplify the ordering process
and it provides racking and cabling rules for the preferred layout of
servers, switches, and cables.

The Deployment configuration file provides a mapping of servers and switches
to software for the purposes of deployment.  Each server is mapped to a set
of OpenStack based software roles constituting the control plane, compute
plane, and storage plane.  Each role is defined in terms of operating system
based resources such as users and networks that need to be configured
to satisfy that role.

The Deployment configuration file needs to be edited so that it reflects the
configuration that is to be installed.  This is mostly a matter of making sure
that the numbers of servers represented match the number of servers to be
installed and that IP addresses are allocated so that the installation is
properly integrated into the data center.

The installation process is split into two parts::

    > Bare metal installation of Linux
    > Installation of OpenStack, Swift, and Operational Management

The Deployment configuration file is fed into the bare metal installation
process which is performed by cluster-genesis.  The operating system is loaded
and configured as specified in the configuration file.  Users, networks, and
switches are configured during this step.  The last step is to invoke a small
script that installs OpenStack, Swift, et al. which is orchestrated by os-services.

More properly, in the bare metal installation step, only the installation tools
for OpenStack and Swift were installed, not the actual services.  The next step
is to configure these tools, so that they install the actual services in a
prescribed manner so that they fit properly in the data center.  The two
projects are os-services and ceph-services.  See the README files of each project
to determine what is required here.

The final step is to invoke cluster-create.sh in the os-services
repository to install and configure the cluster.  os-services orchestrates
the installation process of OpenStack, Swift, and Operational Management
which are loaded into the first controller that is setup by cluster-genesis.

The OpenStack dashboard may be reached through your browser:

https://<ipaddr or hostname of an OpenStack control node>

This recipe also includes an operational management console which is
integrated into the OpenStack dashboard.  It monitors the cloud infrastructure
and shows metrics relates to the capacity, utilization, and health of the
cloud infrastructure.  It may also be configured to generate alerts when
components fail.  It is provided through the opsmgr repository.

.. Hint::
   Only os-services must be configured before invoking create-cluster.  For
   more info, see related projects below.

   Passwords may be found in /etc/openstack_deploy/user_secrets*.yml on
   the first OpenStack controller node.

Getting Started
---------------

The toolkit runs on an Ubuntu 16.04 OpenPOWER server or VM that is connected
to the internet and management switch in the cluster to be configured.

#. Read `swift.pdf <https://github.com/open-power-ref-design/standalone-swift/blob/master/swift.pdf>`_

#. Choose a small, medium, or large deployment configuration file

#. Rack and cable hardware as indicated

#. Get a local copy of this repository::

   $ git clone git://github.com/open-power-ref-design/standalone-swift
   $ cd standalone-swift
   $ TAG=$(git describe --tags $(git rev-list --tags --max-count=1))
   $ git checkout $TAG
   $ CFG=$(pwd)/<your-chosen-config>.yml

#. Edit the configuration file:

   Instructions for editing the file are included in the file itself.

   General information may be found in the
   `Cluster Genesis User Guide <http://cluster-genesis.readthedocs.io/en/latest/>`_

   Swift specific instructions can be found below in the
   section titled Swift Installation and Customization.

#. Validate the configuration file::

   $ git clone git://github.com/open-power-ref-design/os-services
   $ cd os-services
   $ git checkout $TAG
   $ ./scripts/validate_config.py --file $CFG

#. Place the configuration file::

   $ git clone git://github.com/open-power-ref-design-toolkit/cluster-genesis
   $ cd cluster-genesis
   $ git checkout 1.1.0
   $ cp $CFG config.yml

#. Invoke cluster-genesis to perform the bare metal installation process:

   Instructions may be found in the Cluster Genesis User Guide identified above.

#. Wait for cluster-genesis to complete, ~3 hours:

#. Edit the OpenStack and Swift configuration files:

   Instructions may be found below in the section titled
   Swift Installation and Customization.

#. Invoke the toolkit again to complete the installation::

   $ ./scripts/create-cluster

   Note this command is invoked on the first OpenStack controller node.  The commands
   listed above are invoked on the deployer node.  When cluster-genesis completes,
   it displays on the screen instructions for invoking the command above.


Swift Installation and Customization
------------------------------------

Prior to activating cluster-genesis, the following parameters can be customized:

The ``node-templates`` section of config.yml contains a
swift-metadata template for metadata nodes and a swift-object
template for object nodes.  In cases where metadata and object
rings are converged on the same host, only the swift-object
template is present.

Under either swift-metadata or swift-object, the domain-settings
allow devices for each ring to be selected either by pci path or
by individual disk names.  The pci path (e.g. /dev/disk/by-path/pci-0000:01)
will be expanded to include all individual disks on that path.  The
individual disk names (e.g. /dev/sdx) are of course not expanded.

The disk containing the / filesystem will always be avoided.

The account, container, and object device lists cannot partially
overlap.  The lists must either be identical or mutually exclusive.

Here is an example where all rings use the devices on path
pci-0004:03 as well as /dev/sdz.

 .. code-block:: yaml

    node-templates:
        swift-object:
            domain-settings:
                account-ring-devices:
                    - /dev/disk/by-path/pci-0004:03
                    - /dev/sdz
                container-ring-devices:
                    - /dev/disk/by-path/pci-0004:03
                    - /dev/sdz
                object-ring-devices:
                    - /dev/disk/by-path/pci-0004:03
                    - /dev/sdz


The following parameters can be customized
prior to the create cluster phase:


* ``/etc/openstack_deploy/openstack_user_config.yml`` (optional)

     .. code-block:: yaml

          swift:
            mount_point: /srv/node
            part_power: 8
            storage_network: br-storage
            storage_policies:
            - policy:
                default: 'True'
                index: 0
                name: default


  The default settings (which are shown above) include a 3x replication
  policy for the object ring.  The account and container rings do not
  need to be specified and will use 3x replication.

  The description of each setting that can be changed is shown in
  /etc/openstack_deploy/conf.d/swift.yml.example.

  For example, the default storage policy could be changed to use
  erasure coding:

     .. code-block:: yaml

        storage_policies:
        - policy:
            default: 'True'
            index: 0
            name: default
            policy_type: erasure_coding
            ec_type: jerasure_rs_vand
            ec_num_data_fragments: 10
            ec_num_parity_fragments: 4
            ec_object_segment_size: 1048576


  Here is an example using multiple storage policies, where the default
  storage policy named 'default' uses 3x replication and an additional storage
  policy named 'ec10-4' uses erasure coding:

     .. code-block:: yaml


        storage_policies:
        - policy:
            default: 'True'
            index: 0
            name: default
        - policy:
            index: 1
            name: ec10-4
            policy_type: erasure_coding
            ec_type: jerasure_rs_vand
            ec_num_data_fragments: 10
            ec_num_parity_fragments: 4
            ec_object_segment_size: 1048576

  The swift_hosts section of openstack_user_config.yml shows
  which rings reside on a particular set of drives within each
  host.  This is initially based on the settings provided by
  config.yml prior to the bootstrap phase.  For example:

     .. code-block:: yaml


      swift_hosts:
        swift-object-1:
          container_vars:
            swift_vars:
              drives:
              - groups:
                 - default
                name: disk1
              - groups:
                - default
                name: disk2
              ...

              - groups:
                - default
                name: disk7
              - groups:
                - account
                - container
                name: meta1
              - groups:
                - account
                - container
                name: meta2
              - groups:
                - account
                - container
                name: meta6

* ``/etc/openstack_deploy/user_secrets.yml`` (optional)

  This contains passwords which are generated during the create-cluster phase.
  Any fields that are manually filled in after the bootstrap-cluster phase will
  not be touched by the automatic password generator during the create-cluster
  phase.

Advanced Customization
----------------------

The config.yml file which is used as input to cluster-genesis
allows the devices used by Swift rings to be specified as part of
the ``node-templates`` section.  The cluster-genesis code gathers
inventory information from each node and uses that to populate
a ``nodes`` section of its output inventory file,
/var/oprc/inventory.yml.  For situations where heterogenous hardware
is used, it may be necessary for some hosts to override the devices list
specified in the ``node-templates`` section.

Under normal circumstances, when the cluster-genesis project is activated
it will automatically invoke the bootstrap-cluster.sh that is provided
by the os-services project.  In order to perform the advanced customization
steps described below, you will need to prevent that from happening
so that you have time to modify /var/oprc/inventory.yml.

To customize the disks and devices for the Swift rings on a per-node
basis, modify config.yml to remove the call to boostrap-cluster.sh
before initiating cluster-genesis. After cluster-genesis completes,
modify /var/oprc/inventory.yml on the first controller node as
discussed below and then invoke bootstrap-cluster.sh.

The settings in the node-templates section apply to all nodes in the
corresponding nodes section of /var/oprc/inventory.yml unless an
individual node sets domain-settings to override the template.

Here is an example where node 192.168.16.112 specifies different
devices to override the node-templates section shown above.

    .. code-block:: yaml

        nodes:
            swift-object:
            -   ipv4-pxe: 192.168.16.112
                domain-settings:
                    account-ring-devices:
                        - /dev/sdx
                        - /dev/sdy
                        - /dev/sdz
                    container-ring-devices:
                        - /dev/sdx
                        - /dev/sdy
                        - /dev/sdz
                    object-ring-devices:
                        - /dev/sdx
                        - /dev/sdy
                        - /dev/sdz

Verifying an install
--------------------

After successful installation, verify that Swift services are running correctly.

* Check for the existence of a utility container using ``lxc-ls -f`` on the
  controller nodes.

* Attach the utility container using ``lxc-attach -n <container name>``

* Source the environment file::

  $ source /root/openrc

* Run some sample OpenStack Swift commands and ensure they run
  without any errors::

  $ swift list
  $ swift stat
  $ swift post <containerName>
  $ swift list <containerName>
  $ swift stat <containerName>
  $ swift upload <containerName> <filename>
  $ swift download <containerName> <filename>

* Find the public endpoint URL for the OpenStack Keystone
  identity service, so that it can be used to access Swift
  from remote hosts::

  $ openstack catalog list

Using OpenStack Swift
---------------------

Further information on using the OpenStack Swift client can be found at:

http://docs.openstack.org/user-guide/managing-openstack-object-storage-with-swift-cli.html

Administration for OpenStack Swift
----------------------------------

The OpenStack Ansible playbooks can be used to perform administrative
tasks in the cluster.  The playbooks are found on the first OpenStack
controller node in::

  /opt/openstack-ansible/playbooks

The Swift role for OpenStack Ansible is found in::

  /etc/ansible/roles/os_swift

The settings used by these playbooks are in::

  /etc/openstack_deploy/openstack_user_config.yml

For example, changes to the ring configuration could be made
in openstack_user_config.yml.  Then to refresh Swift services, rebuild
the rings, and push these changes out to the cluster::

  $ cd /opt/openstack-ansible/playbooks
  $ openstack-ansible os-swift-sync.yml --skip-tags swift-key,swift-key-distribute

Related projects
----------------

Recipes for OpenPOWER servers are located here:

    - `Recipe directory <https://github.com/open-power-ref-design/>`_

Here, you will find several OpenStack based recipes:

    - `Private cloud w/ and w/o Swift Object Storage <https://github.com/open-power-ref-design/private-compute-cloud/blob/master/README.rst>`_
    - `DBaaS <https://github.com/open-power-ref-design/dbaas/blob/master/README.rst>`_
    - `Standalone Ceph Clusters <https://github.com/open-power-ref-design/standalone-ceph/blob/master/README.rst>`_

The following projects provides services that are used as major building blocks in
recipes:

    - `cluster-genesis <https://github.com/open-power-ref-design-toolkit/cluster-genesis>`_
    - `os-services <https://github.com/open-power-ref-design-toolkit/os-services>`_
    - `ceph-services <https://github.com/open-power-ref-design-toolkit/ceph-services>`_
    - `opsmgr <https://github.com/open-power-ref-design-toolkit/opsmgr>`_

