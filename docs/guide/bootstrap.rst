Bootstrap
---------

Each service that NodeFabric provides (ie Core Layer, MariaDB-Galera and Ceph currently) has to be initialized at first - which is a one-time operation.
However all services are later capable of repeatable (and non-destructive) automated bootstrapping -- even if they loose quorum (ie in case of full cluster/service nodes shutdown/reboot).

.. csv-table::
   :header: "Service", "auto-init (one-time)", "auto-bootstrap (repeatable)"
   :widths: 80, 80, 80

   "NF Core Layer", "yes (with ATLAS)", "yes"
   "MariaDB-Galera", "no", "yes"
   "Ceph MON", "no", "yes"
   "Ceph MDS", "no", "yes"


NF Core Layer
+++++++++++++++++++++++

.. note:: Each node must have its unique FQDN hostname set - otherwise nf-consul service container will refuse to start!

When NodeFabric nodes boot-up first time - they need to join and form the Consul cluster. For joining the cluster together each node must have it's own FQDN hostname set and it needs to know about the other nodes participating - ie how to connect with other nodes (ie having cluster hostmap). There are two supported methods for initializing cluster hostmap:

- by using remote auto-join mode with Hashicorp ATLAS public service (strictly optional but very convinient - hence recommended)
- by editing /etc/nodefabric/nodefabric.hostmap config file manually (on ALL nodes) 

Setup node FQDN hostname (IMPORTANT)
*************************************

Depending on target environment there are three different cases:

- in case of AWS node hostnames will be set by default already (using VPC LAN ip as a hostname) - optionally it is possible to supply custom hostname through user-data (ie NODENAME=node1.example.com)
- in case of Openstack please set VM hostname by supplying NODENAME=node1.example.com as part of user-data during VM launch
- in case of non-cloud deployments please login to node shell and set hostname manually - by following this recipe:

.. code-block:: bash

    # NB! You must update also HOSTNAME environment variable - as it is used in scripts!
    export HOSTNAME=node1.nf.int
    hostnamectl set-hostname $HOSTNAME

    # verify
    echo $HOSTNAME && hostnamectl

Activating remote auto-join mode
********************************

Hashicorp ATLAS service can be used for NF Core Layer remote auto-join. Main benefit here is that you don't need to know about node internal IPs for constructing the initial nodefabric hostmap - as this data will be collected and spread automagically by ATLAS remote service. ATLAS also adds remote web-based status dashboard as a bonus. Hashicorp does offer free-tier ATLAS service plans in order to get started.

For activating this remote auto-join mode within NodeFabric Host Image you have 2 possible options:

- either by supplying ATLAS_TOKEN and ATLAS_ENVNAME key-value pairs through cloud user-data at boot time (for each node)
- or by manually editing /etc/nodefabric/conf.d/nf-consul.conf file directly (after node has booted up) -- and providing ATLAS_TOKEN together with desired ATLAS_ENVNAME there (on ALL nodes)

Example manually edited /etc/nodefabric/conf.d/nf-consul.conf file should look like this (replace CONSUL_ATLAS_TOKEN and CONSUL_ATLAS_ENVNAME values with yours):

.. code-block:: bash

   ### CONSUL CONFIG ###
   CONSUL_INSTANCE="nf-consul"
   CONSUL_IMAGE="opennode/nf-consul"
   CONSUL_DATADIR="/var/lib/consul"
   CONSUL_CONFDIR="/etc/nodefabric/files.d/consul/config"
   CONSUL_EXECDIR="/etc/nodefabric/files.d/consul/scripts"
   CONSUL_BOOTSTRAP_EXPECT=3
   CONSUL_NODENAME="$( hostname )"
   CONSUL_BIND_IP="$HOST_PUBLIC_IP"
   CONSUL_BOOTSTRAP_HOSTS="$( cat /etc/nodefabric/nodefabric.hostmap 2>/dev/null | awk '{ print $1 }' )"
   CONSUL_BOOTSTRAP_HOSTS_CSV=$( echo $CONSUL_BOOTSTRAP_HOSTS | tr ' ' , )
   CONSUL_ATLAS_ENVNAME="jdunlop/my-cluster" # NB! Parameter format is: "atlas-user/atlas-env" as "jdunlop/testcluster"
   CONSUL_ATLAS_TOKEN="7ks0pfuyZI6Jgg.atlasv1.fMYK8ySzyEbozyel3T1vi2qR2MZ3lHyAtCrOy7sYDnuYdnohmDarvlVKj01bxPa8syb"
   CONSUL_SHARED_SECRET="" # Generate as: openssl rand -base64 16 

.. note:: You need to execute 'systemctl restart nf-consul' after manually editing nf-consul.conf for ATLAS token and environment name! 

Manual bootstrap procedure
******************************

If you don't want to use remote auto-join mode then you can simply supply initial cluster hostmap manually - by editing /etc/nodefabric/nodefabric.hostmap config file and providing LAN IP address and hostname for each node in standard hostsfile format (ie ipaddr fqdn shortname in every line). 

Example nodefabric.hostmap file would look like this:

.. code-block:: bash

   192.168.40.101 node01.nf.int node01
   192.168.40.102 node02.nf.int node02
   192.168.40.103 node03.nf.int node03

.. note:: You need to execute 'systemctl restart nf-consul' after manually editing /etc/nodefabric/nodefabric.hostmap config file!

After all nodes have been bootstrapped you can observe NodeFabric Core Layer status by running nodefabric-dashboard (or nodefabric-status) utility:

.. code-block:: bash
   
   [centos@ip-172-30-0-100 ~]$ sudo nodefabric-dashboard
   
   # or one-off version of it would be
   [centos@ip-172-30-0-100 ~]$ sudo nodefabric-status

.. figure:: ../images/screenshots/aws_nf_dash_init.png
    :alt: NodeFabric Core Layer status

Debug
***************

Consul eventlog can be observed on each cluster node by running nodefabric-monitor:

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nodefabric-monitor


Enabling MariaDB-Galera service
+++++++++++++++++++++++++++++++ 

MariaDB-Galera database cluster is packaged and delivered as nf-galera docker containers - which are already included into NodeFabric Host Image. It's service management commands are provided by nf-galera-ctl utility:

.. code-block:: bash

   [root@nf-dev1 ~]# nf-galera-ctl help


Enable DB nodes
****************

For MariaDB-Galera database service initialization you need to enable and start nf-galera containers across all cluster nodes. Do this by executing 'nf-galera-ctl enable' on a single cluster node:

.. note:: 'nf-galera-ctl enable' command is broadcasted across ALL cluster nodes (ie run it on single node only)

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-galera-ctl enable

Please observe MySQL service node statuses from nodefabric-dashboard. All nodes should turn red gradually - which indicates that particular service container is up but is not yet passing all the health-checks (yellow status means container not yet started). Global MySQL DB service should stay in “FAILED” status for now - as it is not yet bootstrapped:

.. figure:: ../images/screenshots/aws_nf_galera_enable.png
    :alt: MariaDB-Galera nodes enabled

Bootstrap DB cluster
********************

Once all DB service nodes reach "red/up/failed" status -- you can execute 'nf-galera-ctl bootstrap' command for dataset initialization and cluster bootstrap:  

.. note:: 'nf-galera-ctl bootstrap' command is broadcasted across ALL cluster nodes (so run it on single node only)

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-galera-ctl bootstrap

It might take up to couple of minutes normally - when DB node statuses should turn to green in nodefabric-dashboard - and global DB service status should reach into “RUNNING” state:   

.. figure:: ../images/screenshots/aws_nf_galera_bootstrapped.png
    :alt: MariaDB-Galera nodes bootstrapped

.. note:: After successful bootstrap database "root" user password is left empty and the account connectivity is limited to localhost


Debug
***************

For debugging purposes nf-galera-monitor command can be used:

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-galera-monitor


Enabling Ceph storage services
++++++++++++++++++++++++++++++++

There are 3 separate Ceph storage services that are currently included within NodeFabric Host Image:

- Ceph cluster (MON) service
- Ceph Remote Block Devices service (RBD) 
- Ceph distributed filesystem service (CephFS)

Ceph cluster monitor (MON) service is delivered as nf-ceph-mon docker containers - and it needs to be successfully initialized first - before any OSDs can join and before CephFS layer could be bootstrapped.

Object Storage Daemon software is actually included and run directly in the NodeFabric host OS context and for each Ceph data disk device there should be its own OSD daemon instance attached and running. You need to provide these dedicated block devices (min. 64GB per disk) to NodeFabric host for Ceph storage - additionally to default OS root disk. Multiple disks spreaded evenly across multiple NodeFabric hosts are recommended.

CephFS Metadata Service (ie MDS) is included as nf-ceph-mds docker container. It can be enabled and initialized after Ceph monitor cluster is running and initial number of OSDs (3) are joined and operational for storage pools. CephFS operates on top of its own dedicated Ceph RBD pools - which need to be created during bootstrap procedure.

nf-ceph-ctl, nf-ceph-disk and nf-ceph-fs utilities are used for various Ceph cluster related management tasks:

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-ctl help
   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-disk help
   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-fs help


Enable and bootstrap MON cluster
*************************************

For enabling and starting nf-ceph-mon containers across all cluster nodes please execute 'nf-ceph-ctl enable':

.. note:: 'nf-ceph-ctl enable' command is broadcasted to ALL cluster nodes - so execute on single node only

.. code-block:: bash
   
   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-ctl enable

Ceph MON service node statuses should gradually turn red in nodefabric-dashboard:

.. figure:: ../images/screenshots/aws_nf_ceph_enable.png
    :alt: Ceph MON nodes enabled

Once ALL Ceph MON nodes have been reaching UP status - you can issue 'nf-ceph-ctl bootstrap' for initializing Ceph cluster (one-time).
This bootstrap process generates and distributes initial Ceph cluster configuration and keys across all nodes.

.. note:: Run 'nf-ceph-ctl bootstrap' on single node only - as it is broadcasted command

.. code-block:: bash
   
   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-ctl bootstrap

Ceph MON service node statuses should be gradually reaching into OK state (expected bootstrap time should be normally less than a minute). Global Ceph MON service should reach into “RUNNING” state - as seen from the dashboard:

.. figure:: ../images/screenshots/aws_nf_ceph_bootstrapped.png
    :alt: Ceph MON nodes bootstrapped


Provide and initialize Ceph disks
*************************************

.. note:: Ceph disks have to be initialized on EACH node separately -- meaning that nf-ceph-disk commands DO NOT broadcast across cluster!

Please login to each NodeFabric host and list available block devices (that you have previously attached to this VM/host):

.. code-block:: bash
   
   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-disk list
   INFO: Listing block devices ...
   /dev/xvda :
    /dev/xvda1 other, xfs, mounted on /
   /dev/xvdb other, unknown 

Block devices with 'unknown' statuses are good candidates for Ceph disks :) In order to initialize particular block device as Ceph disk you have to run 'nf-ceph-disk init' command with full path to particular block device provided. 

.. note:: 'nf-ceph-disk init' WILL DESTROY ALL DATA ON SPECIFIED TARGET DISK! 

.. note:: The following command will produce some partx related error/warning messages in the output - which can be ignored

.. code-block:: bash
   
   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-disk init /dev/xvdb
   INFO: Initializing /dev/xvdb ...
   WARN: THIS WILL DESTROY ALL DATA ON /dev/xvdb!
   Are you sure you wish to continue (yes/no): yes
   Creating new GPT entries.
   GPT data structures destroyed! You may now partition the disk using fdisk or
   other utilities.
   The operation has completed successfully.
   partx: specified range <1:0> does not make sense
   The operation has completed successfully.
   partx: /dev/xvdb: error adding partition 2
   The operation has completed successfully.
   partx: /dev/xvdb: error adding partitions 1-2
   meta-data=/dev/xvdb1             isize=2048   agcount=4, agsize=720831 blks
            =                       sectsz=512   attr=2, projid32bit=1
            =                       crc=0        finobt=0 
   data     =                       bsize=4096   blocks=2883323, imaxpct=25
            =                       sunit=0      swidth=0 blks
   naming   =version 2              bsize=4096   ascii-ci=0 ftype=0
   log      =internal log           bsize=4096   blocks=2560, version=2
            =                       sectsz=512   sunit=0 blks, lazy-count=1
   realtime =none                   extsz=4096   blocks=0, rtextents=0
   The operation has completed successfully.
   partx: /dev/xvdb: error adding partitions 1-2
   INFO: /dev/xvdb initialized!

You can verify local OSD service status by issuing 'nf-ceph-disk status':

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-disk status

.. figure:: ../images/screenshots/aws_nf_ceph_disk_status.png
    :alt: Ceph OSD status

.. note:: Now repeat this process and initialize ALL Ceph disks on ALL cluster nodes!

Once you are finished with Ceph disks initialization on all nodes - you should see the following fragment in the nodefabric-dashboard Ceph Status section (look for osdmap status line):

.. figure:: ../images/screenshots/aws_nf_ceph_osdmap.png
    :alt: Ceph OSD MAP

Enable and bootstrap CephFS
*************************************

For enabling CephFS layer - a POSIX compliant distributed filesystem - you need to start Ceph Metadata Daemon containers first (command is broadcasted across cluster nodes):

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-fs enable

Observe global Ceph MDS Service reaching into "RUNNING" state in nodefabric-dashboard - before proceeding with CephFS bootstrap:

.. figure:: ../images/screenshots/aws_nf_ceph_fs_enable.png
    :alt: CephFS enabled

Once Ceph MDS service is running you can issue CephFS bootstrap command (execute on single node):

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-fs bootstrap

After that you should see mdsmap line in Ceph status section in nodefabric-dashboard:

.. note:: Currently Ceph MDS service is run in active-passive mode - as suggested by Ceph authors for the sake of the stability

.. figure:: ../images/screenshots/aws_nf_ceph_fs_mdsmap.png
    :alt: Ceph MDS Map


Now you can proceed and mount CephFS on each cluster node - if you desire to do so:

.. note:: This command is not broadcasted and enables only local /srv/cephfs mountpoint

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-fs mount

For checking global Ceph Metadata service status and local mountpoint on current node please run:

.. code-block:: bash

   [centos@ip-172-30-0-100 ~]$ sudo nf-ceph-fs status

.. figure:: ../images/screenshots/aws_nf_ceph_fs_status.png
    :alt: CephFS service status

