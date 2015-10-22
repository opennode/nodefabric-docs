CHANGELOG
=========

0.4.3 (Oct 22, 2015)
----------------------
- FEATURE: Added support for MariaDB configuration files data volume
- FEATURE: Added nodefabric-update utility for system update
- IMPROVEMENT: Switched to non-blocking XtraBackup SST method for MariaDB-Galera (was rsync)
- IMPROVEMENT: Changed internal load-balanced MariaDB-Galera service endpoint to active-backup mode (in order to avoid potential multi-master deadlock problems with co-located apps using internal mysql service endpoint)
- IMPROVEMENT: nf-galera-ctl database management subcommands supporting root password input (when required)
- IMPROVEMENT: Better nf-galera service check script - in order to prevent node status flapping
- BUGFIX: Fixed nf-galera-ctl password change/update replication issue
- EXPERIMENTAL: Implemented initial support for (fixed) 5-node clusters

0.4.2 (Sep 29, 2015)
----------------------
- FEATURE: Added database management subcommands to nf-galera-ctl utility
- IMPROVEMENT: Database root account is now initialized with empty password and access is limited to localhost
- IMPROVEMENT: Ensuring that Ceph RBD and CephFS modules get autoloaded on boot
- IMPROVEMENT: Removing nf-galera service wait time

0.4.1 (Aug 21, 2015)
----------------------
- Initial release
