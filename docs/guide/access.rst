Access
------

NodeFabric nodes/instances should be accessed over SSH connection for management, configuration and manual bootstrapping purposes. 
There are also local and remote web-based status dashboards available - more details about these are presented in the "Management" chapter.


SSH login
+++++++++++++++++++++++

Node/instance SSH login credentials are activated through cloud-init method internally and it is highly recommended to use key-based authentication.
Exact details how you need to supply your SSH public key differ between target environments:

- in case of AWS EC2 you have to create your ssh keypair in EC2 console
- for Openstack you have to setup your ssh keypair through Horizon UI or CLI
- for plain hypervisors (ie KVM, Parallels Desktop, VirtualBox, VMWare) you have to supply your ssh public key as part of custom cloud-init.iso
- TODO: vmware vSphere 

We do supply prebuilt and downloadable cloud-init.iso which is available from here: http://downloads.sourceforge.net/project/opennode/NodeFabric/cloud-init.iso. It sets default user login password to a known value (rootpass). 

You should create your own cloud-init.iso with your custom ssh keys included - if using this method for user account activation.
Custom cloud-init.iso can be generated as follows (MacOSX and Linux instructions provided here):

.. code-block:: bash

   # create top-level iso folder
   mkdir -p cloud-init

   # skip cloud-init meta-data file contents for now
   echo > cloud-init/meta-data

   # set ssh key and password 
   cat << EOF > cloud-init/user-data
   #cloud-config
   password: rootpass 
   ssh_pwauth: True
   chpasswd: { expire: False }

   ssh_authorized_keys: 
     - ssh-rsa <paste_your_ssh_pubkey_here>
   EOF

   # generating cloud-init ISO on MacOSX
   hdiutil makehybrid -o cloud-init.iso -hfs -joliet -iso -default-volume-name cidata cloud-init/

   # generating cloud-init ISO on Linux
   genisoimage -output cloud-init.iso -volid cidata -joliet -rock cloud-init/


.. note:: Default username for ssh login is: "centos" (for downloadable images) OR "ec2-user" for AWS EC2 AMI based instances 

.. note:: Prebuilt downloadable cloud-init.iso sets default user login password to: rootpass

.. note:: NodeFabric 0.4.1 downloadable images do contain default root:rootpass account already 

The following shell commands might be helpful in order to connect to deployed NodeFabric instances:

.. code-block:: bash

    # Set node IP to connect to
    NODE_IP="10.211.55.100" # replace this example IP with yours
    
    # Set login username
    NODE_USER="centos" # OR ec2-user for AWS
    
    # Set to your login private key path
    KEY_PATH=".ssh/id_rsa"

    # Connect with your key
    ssh -i ${KEY_PATH} ${NODE_USER}@${NODE_IP}

.. note:: You can set root user password and switch to root user priviledged environment by running the following commands: 

.. code-block:: bash

   # setting root password
   sudo passwd

   # switching to root user environment
   su - root


Firewall ports
+++++++++++++++++++++++

NodeFabric open network ports can be divided into 3 separate access zones: localhost only, LAN only and WAN/remote access.
Enabling ICMP (ie ping) within LAN zone is highly recommended for diagnostic purposes. Management and internal dashboards access should be done over SSH connection (using port forwarding where necessary). Outgoing public internet connection is required for optional ATLAS cluster auto-join and remote dashboard services.

**Zone: localhost**

.. csv-table::
   :header: "Service", "port(s)", "proto", "comments"
   :widths: 80, 40, 30, 100

   "Consul CLI RPC", 8400, "tcp"
   "Consul HTTP API & UI", 8500, "tcp", "Access UI through ssh pf"
   "Consul DNS", 8600, "tcp/udp"
   "HAProxy UI", 48080, "tcp", "Access through ssh pf"

**Zone: LAN**

.. csv-table::
   :header: "Service", "port(s)", "proto", "comments"
   :widths: 80, 40, 30, 100

   "Consul RPC", 8300, "tcp"
   "Consul SERF", 8301, "tcp/udp"
   "MariaDB SQL", 3306, "tcp"
   "Galera SST", 4444, "tcp"
   "Galera WSREP", 4567, "tcp/udp"
   "Galera IST", 4568, "tcp"
   "Ceph MON", 6789, "tcp"
   "Ceph OSDs & MDS", 6800:7300, "tcp"

**Zone: WAN/remote access**

.. csv-table::
   :header: "Service", "port(s)", "proto", "comments"
   :widths: 80, 40, 30, 100

   "SSH", 22, "tcp", "Could be limited to LAN only"
   "Consul WAN gossip", 8302, "tcp/udp", "IF remote DCs are enabled"

