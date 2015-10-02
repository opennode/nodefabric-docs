Troubleshoot
------------

Database cluster not auto-bootstrapping after full shutdown
++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++

In case of database cluster bootstrap problems you can re-run 'nf-galera-ctl bootstrap' - as it is designed to be re-run safe. It does not re-initialize dataset once it already exists -- it only recovers last GTID and transforms node with the latest dataset as primary node.

.. code-block:: bash

    sudo nf-galera-ctl bootstrap
    sudo nf-galera-monitor

Ceph OSD does not activate after node reboot
+++++++++++++++++++++++++++++++++++++++++++++

Symptoms:

.. code-block:: bash

   # Problem symptom #1: OSD mount is shown but OSD systemd service entry is missing 
   [root@nf-dev2 ~]# sudo nf-ceph-disk status

   INFO: Listing OSD services ...


   INFO: Listing OSD mounts ...

   var-lib-ceph-osd-ceph\x2d2.mount - /var/lib/ceph/osd/ceph-2
     Loaded: loaded (/proc/self/mountinfo)
     Active: active (mounted) since Wed 2015-09-30 12:34:16 GST; 6min ago
      Where: /var/lib/ceph/osd/ceph-2
       What: /dev/sdb1
   
   # Problem symptom #2: Ceph disk listing will complain over filesystem corruption 
   [root@nf-dev2 ~]# sudo nf-ceph-disk list
   INFO: Listing block devices ...
   mount: mount /dev/sdb1 on /var/lib/ceph/tmp/mnt.RuWU_R failed: Structure needs cleaning
   WARNING:ceph-disk:Old blkid does not support ID_PART_ENTRY_* fields, trying sgdisk; may not correctly identify ceph volumes with dmcrypt
   /dev/sda :
    /dev/sda1 other, xfs, mounted on /boot
    /dev/sda2 other, LVM2_member
   /dev/sdb :
   mount: mount /dev/sdb1 on /var/lib/ceph/tmp/mnt.SGq2oW failed: Structure needs cleaning
    /dev/sdb1 ceph data, unprepared
    /dev/sdb2 ceph journal
   /dev/sr0 other, unknown

Fixes:

.. code-block:: bash

   # Repairing filesystem
   [root@nf-dev2 ~]# sudo xfs_repair /dev/sdb1
   Phase 1 - find and verify superblock...
   Phase 2 - using internal log
           - zero log...
   * ERROR: mismatched uuid in log
   *            SB : 1cb2ae7d-5765-46c8-a217-03c1b4a6cfde
   *            log: 9df2630e-5e8f-4455-9c72-c0b27764bace
           - scan filesystem freespace and inode maps...
           - found root inode chunk
   Phase 3 - for each AG...
           - scan and clear agi unlinked lists...
           - process known inodes and perform inode discovery...
           - agno = 0
           - agno = 1
           - agno = 2
           - agno = 3
           - process newly discovered inodes...
   Phase 4 - check for duplicate blocks...
           - setting up duplicate extent list...
           - check for inodes claiming duplicate blocks...
           - agno = 0
           - agno = 1
           - agno = 2
           - agno = 3
   Phase 5 - rebuild AG headers and trees...
           - reset superblock...
   Phase 6 - check inode connectivity...
           - resetting contents of realtime bitmap and summary inodes
           - traversing filesystem ...
           - traversal finished ...
           - moving disconnected inodes to lost+found ...
   Phase 7 - verify and correct link counts...
   done

   # Re-activate OSD (note that you need to re-activate partition - not disk device!)
   [root@nf-dev2 ~]# sudo nf-ceph-disk activate /dev/sdb1
   INFO: Activating /dev/sdb1 ...
   === osd.1 ===
   create-or-move updated item name 'osd.1' weight 0.06 at location {host=nf-dev2,root=default} to crush map
   Starting Ceph osd.1 on nf-dev2...
   Running as unit run-6098.service.
   INFO: /dev/sdb1 activated!




