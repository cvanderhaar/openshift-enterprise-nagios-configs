openshift-enterprise-nagios-configs
===================================

Some configuration for doing simple Nagios monitoring of Openshift Enterprise.

#Selinux Module & Context

##ose-node policy

In the `selinux` folder of this repo is some source for an `ose-node`
selinux module. This sets up our `nagios_openshift_plugin_exec_t` type
and tells selinux that it's cool.

To install this policy, run the following command from the directory
`ose-node.te` lives in:
```
make -f /usr/share/selinux/devel/Makefile ose-node.pp; semodule -i ose-node.pp
```

Note: Using RHEL 6.5 here (just incase something doesn't work)

##Plugin files & context

Plugin files should be dropped in `/usr/lib64/nagios/plugins/extra/`
and will need their selinux type set.

```
chcon -t nagios_openshift_plugin_exec_t /usr/lib64/nagios/plugins/extra/*
```

#Sudoers

Some of these check commands need to run with sudo so we've given nrpe
permission to do so in cases where there's little risk.

You'll need to make sure that sudoers is configured to not require a
tty.

##Node

Add the following to `/etc/sudoers`

```
nrpe    ALL=(ALL) NOPASSWD: /usr/sbin/oo-accept-node
```

##Broker

Add the following to `/etc/sudoers`.

```
nrpe    ALL=(ALL) NOPASSWD: /usr/sbin/oo-accept-broker
nrpe    ALL=(ALL) NOPASSWD: /usr/sbin/oo-admin-ctl-district -b
```

#Plugins

##check_broker_accept_status (bash)

Requires configuring sudoers. See Sudoers section above.

Checks for `PASS` result from `oo-accept-broker -v`, returns verbose output.

###Nrpe config entry

```
command[check_broker_accept_status]=/usr/lib64/nagios/plugins/extra/check_broker_accept_status
```

###Service definition

```
define service {
    use                         slow-service
    service_description         BROKER-ACCEPT-STATUS
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_broker_accept_status
    }
```

##check_node_accept_status (bash)

Requires configuring sudoers. See Sudoers section above.

Checks for `PASS` result from `oo-accept-node -v`, returns verbose output.

###Nrpe config entry

```
command[check_node_accept_status]=/usr/lib64/nagios/plugins/extra/check_node_accept_status
```

###Service definition

```
define service {
    use                         slow-service
    service_description         ACCEPT-NODE-STATUS
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_node_accept_status
    }
```

##check_system_capacity (python)

Requires configuring sudoers. See Sudoers section above.

Depends: python-simplejson

Checks capacity output of `oo-admin-ctl-district`. Like this:

```
{...
 "available_capacity"=>5978,
 "max_capacity"=>6000,
 ...}

```

Returns CRITICAL if available capacity under 25%.

###Nrpe config entry

```
command[check_system_capacity]=/usr/lib64/nagios/plugins/extra/check_system_capacity
```

###Service definition

```
define service {
    use                         generic-service
    service_description         SYSTEM-CAPACITY
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_system_capacity
    }

```

##check_selinux_status (ruby)

Ensures selinux is enabled.

###Nrpe config entry

```
command[check_selinux_status]=/usr/lib64/nagios/plugins/extra/check_selinux_status --warning permissive --critical disabled
```

###Service definition

```
define service {
    use                         generic-service
    service_description         SELINUX-STATUS
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_selinux_status
    }
```

##check_mcollective

Ensure `mcollectived` running w/ generic `check_procs` call.

###Nrpe config entry

```
command[check_mcollective]=/usr/lib64/nagios/plugins/check_procs -C ruby -a mcollectived -c 1:1
```

###Service definition

```
define service {
    use                         generic-service
    service_description         MCOLLECTIVE
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_mcollective
    }
```

##check_cgroups

Ensure `cgrulesengd` running w/ generic `check_procs` call.

###Nrpe config entry

```
command[check_cgroups]=/usr/lib64/nagios/plugins/check_procs -C cgrulesengd -a cgred -c 1:1
```

###Service definition

```
define service {
    use                         generic-service
    service_description         CGROUPS
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUPS]
    check_command               check_nrpe_command!check_cgroups
    }
```

##check_mongo_rs_status (ruby)

Requires configuring `$config` section of the plugin script to set
host, port, etc.

Depends: rubygems, rubygem-mongo

Checks mongo replicaset status. EX: OK: All nodes (total: 3) are up and healthy

###Nrpe config entry

```
command[check_mongo_rs_status]=/usr/lib64/nagios/plugins/extra/check_mongo_rs_status
```

###Service definition

```
define service {
    use                         generic-service
    service_description         MONGO-RS-STATUS
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_mongo_rs_status
    }

```

##check_mongo_rs_timestamp (ruby)

Requires configuring `$config` section of the plugin script to set
host, port, etc.

Depends: rubygems, rubygem-mongo

Checks mongo replicaset timestamp. EX. OK: All nodes (total: 3) are in sync [1406318347]

###Nrpe config entry

```
command[check_mongo_rs_timestamp]=/usr/lib64/nagios/plugins/extra/check_mongo_rs_timestamp
```

###Service definition

```
define service {
    use                         generic-service
    service_description         MONGO-RS-TIMESTAMP
    host_name                   [HOSTNAME]
    contact_groups              [CONTACT-GROUP]
    check_command               check_nrpe_command!check_mongo_rs_timestamp
    }

```

#Service definitions

The `oo-accept-*` commands can take a while to run so we don't run
them as often and have a `slow-service` that gets checked on a 5 min
interval rather than 3.

```
define service {
    name                            generic-service
    active_checks_enabled           1
    passive_checks_enabled          1
    parallelize_check               1
    obsess_over_service             1
    check_freshness                 0
    notifications_enabled           1
    event_handler_enabled           1
    flap_detection_enabled          1
    process_perf_data               1
    retain_status_information       1
    retain_nonstatus_information    1
    is_volatile                     0
    check_period                    24x7
    max_check_attempts              3
    normal_check_interval           3
    retry_check_interval            1
    notification_interval           60
    notification_period             24x7
}

define service {
    name                            slow-service
    active_checks_enabled           1
    passive_checks_enabled          1
    parallelize_check               1
    obsess_over_service             1
    check_freshness                 0
    notifications_enabled           1
    event_handler_enabled           1
    flap_detection_enabled          1
    process_perf_data               1
    retain_status_information       1
    retain_nonstatus_information    1
    is_volatile                     0
    check_period                    24x7
    max_check_attempts              3
    normal_check_interval           5
    retry_check_interval            3
    notification_interval           60
    notification_period             24x7
}
```

#Nrpe Config

Again, since we have some slow running service definitions for those
pesky slow check commands, we have an amped up connection and command
timeout.

`/etc/nagios/nrpe.cfg` tweaks:

```
# COMMAND TIMEOUT
# This specifies the maximum number of seconds that the NRPE daemon will
# allow plugins to finish executing before killing them off.

command_timeout=240

# CONNECTION TIMEOUT
# This specifies the maximum number of seconds that the NRPE daemon will
# wait for a connection to be established before exiting. This is sometimes
# seen where a network problem stops the SSL being established even though
# all network sessions are connected. This causes the nrpe daemons to
# accumulate, eating system resources. Do not set this too low.

connection_timeout=300
```
