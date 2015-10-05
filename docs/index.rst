.. nodefabric-guide documentation master file, created by
   sphinx-quickstart on Sat Sep 26 14:34:14 2015.
   You can adapt this file completely to your liking, but it should at least
   contain the root `toctree` directive.

Welcome
=======

NodeFabric Host Image is a modular VM (or bare-metal) image that delivers hyperconverged MariaDB-Galera database and Ceph storage solution for highly available, self-healing and load-balanced cloud services.

Based on stable and supported RedHat Enterprise Linux operating system or CentOS Atomic Host - it provides highly available data backend layer for self-healing and load-balanced cloud services. 

Docker, Consul and HAProxy are used internally for coordinating and maintaining included data storage services. NodeFabric Host Image can run on any virtual or physical infrastructure: Amazon EC2 cloud, Openstack and VMware private clouds or directly on bare metal. 

**Features include:**

- prebuilt NodeFabric Host Image and optional remote cluster auto-join service 
- zero-­configuration data backend fabric deployment ­- just "Boot--and­-Go"
- self-contained and runs everywhere - AWS, Openstack, VMWare, KVM, bare-metal etc.
- very low infrastructure capabilities requirements for clustering -- it does not require multicast networking or node fencing
- clustered by design
- built-in service discovery and health monitoring

**Example use cases:**

- hyperconverged solution stack (Docker + database + shared FS + load-balancer)
- highly available turnkey database cluster
- virtual SAN with distributed filesystem support

More information about supported NodeFabric product can be found here:
http://opennodecloud.com/products/nodefabric.html


User Guide
----------

.. toctree::
   :maxdepth: 1

   changelog
   guide/intro

License
-------

NodeFabric is released under open-source Apache v2 license.
