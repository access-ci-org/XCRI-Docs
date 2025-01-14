# OpenStack Operations and Maintenance

### Useful Additions and Operations

---

#### Floating IP Addresses

Adding in and configuring a subnet of floating IPs can be extremely useful for your users if they would like to have a persistent IP address.  It's also good for administrators as this allows VMs that need to be accessible but not necessarily publicly routable to not use a public IPv4 address, which are becoming increasingly scarce, and simply have them on an internal network.  To set this up, we need to make sure our provider network has the value `router:external` set to `External`.  An easy way to check this with the following:
``` shell
openstack network list --provider-network-type flat
```
And checking the network that shows up from that with:
``` shell
openstack network show <network name>
```

Once that's set, you should make sure that you have a subnet attached to that network that contains the public facing IP address subnet that you have available.  If you need to create the subnet, you can do that with the following:
``` shell
openstack subnet create --network <network name> --allocation-pool start=<starting IP address>,end=<ending IP address> --dns-nameserver <dns resolver> --gateway <network gateway> --subnet-range <IP address subnet> <subnet name>
```
This will attach the newly created subnet to the network that's created and also allow users to allocate a floating IP from that range to their VMs.


#### Auto-allocated Network

[Automatic allocation of network topologies](https://docs.openstack.org/neutron/latest/admin/config-auto-allocation.html) can save a large amount of time for the administrators and/or support personnel by having users simply create a network, router, and subnet with one line, instead of the gratuitously arduous way of creating them one by one with the CLI.  This also supports popular additional interfaces like Exosphere ([link](https://gitlab.com/exosphere/exosphere/#to-use-with-an-openstack-cloud)).  The setup is relatively simple for how much work it cuts out for your users.  

The first step is to ensure you have a default public network setup.  To check this, run:
``` shell
openstack network list --external
```
Then if you have any networks listed, verify the `is_default` value when shown with:
``` shell
openstack network show <external network name>
```
If it is the default network, you can move on to the next step.  If not, set it to default with:
``` shell
openstack network set <external network name> --default
```

Now, we need a default subnet pool that the auto-allocated network will use. You can create this with:
``` shell
openstack subnet pool create --share --default --pool-prefix 192.168.0.0/16 --default-prefix-length 25 shared-default
```
This will create a default subnet pool using the `192.168.0.0/16` subnet, each carving off a /25 subnet off.  Each subnet will allow 128 different VMs on the network and also allow you to be able to carve off 512 subnets.  You can adjust these numbers if they won't fit your use case.  

Now, your users can create an auto-allocated network with:
``` shell
openstack network auto allocated topology create --or-show
```

#### Adding and Managing Users

This is probably something that is more necessary than recommended.  In this section, we will go over adding users, allocations, adjusting quotas for specific use cases, etc.  

To add a user, you can run:
``` shell
openstack user create --password-prompt <user name>
```
This will ask you for a password.  You can pass this information to each user to allow them to access the OpenStack cloud.  To delete a user, it's simply:
``` shell
openstack user delete <user name>
```

The newly created user will need a project to operate on as well.  If they don't have a project, you can create one with:
``` shell
openstack project create --description "<A good description here>" <project name>
```

In OpenStack, you don't simply add a user to a project.  There needs to be a role associaction created, associating the user and the project.  The role itself can be a privileged user or standard user.  If you don't have a standard user role created, you can create one with:
``` shell
openstack role create <role name>
```
Role associations are created with:
``` shell
openstack role add --project <project name> --user <user name> <role name>
```

To remove the role association, run:
``` shell
openstack role remove --user <user name> --project <project name> <role name>
```

#### Managing Quotas

Oftentimes, different projects or users will have different needs of the OpenStack cloud.  This can lead to the default quotas not allowing them to do something that fits their workflow.  For example, perhaps they need more than the default quota of volumes.  These can be adjusted with the CLI by running:
``` shell
openstack quota set --volumes <new volume quota> <project name>
```
Note that quotas are set per project and not per user.



### Maintenance

---

#### NTP

NTP runs very well and generally, issues stemming from NTP won't arise throughout the life of the cloud.  However, there can be the situation where there are clock drift errors in the logs and also odd behavior by compute nodes and their VMs.  Usually this is either two issues.  First, the time daemon running on your node may simply need a restart.  The second is that it could simply be a misconfiguration on said time daemon.  NTP configuration is outside the scope of this guide, but check the manuals for your specific time daemon for more information.

#### RabbitMQ

To check the status of your RabbitMQ cluster, run the following on one of the RabbitMQ nodes:
``` shell
rabbitmqctl cluster_status
```
Some things to look for here, are any errors or alarms under the [Alarms] section.

Other RabbitMQ errors can be tough to diagnose.  All the OpenStack services send messages through RabbitMQ, so there is a broad range of errors that an administrator can come across while trying to determine the cause.  If you are seeing something akin to this, be sure to check your RabbitMQ cluster with the check command.  If you find that this is indeed the cause, generally a dead RabbitMQ service on one of the nodes is the cause.  Try restarting the service or check the logs to find out what caused the service itself to become inactive.

#### MySQL (a.k.a. MariaDB)

To check the status of a MySQL cluster, SSH to the MySQL node and log into the database with:
`mysql`
Also, you can check the logs to make sure there are no discrepancies.
If you are running a multi-node MySQL cluster, you can also check each node at once by SSHing into a MySQL node and log into the database with:
`mysql`
and run
```
SHOW GLOBAL STATUS LIKE 'wsrep_%';
```
And verify the cluster count matches the number of nodes in your database cluster.

If MySQL is causing issues, you'll likely receive a 503 error when attempting to run commands from the CLI. This could be as simple as a dead `mysql` daemon on one of your nodes.  You can diagnose this by running through the checks above.  

Throughout the life of your cloud, you may come into issues where the database and OpenStack reflect different information.  This could be a VM that shows a host on one machine but it's actually on another, a volume that is stuck in a `reserved` state but won't change back to active/available, etc.  To fix this, we need to log into the MySQL cluster by running `mysql` and updating the discrepancy with `update ITEM set VALUE=DESIRED_VALUE  where id='UUID';`.  You can verify which specific item(s) you'll adjust by adjusting the formula to, `show ITEM where id='UUID';`.

#### Nova

To check the nova status, source the Admin openrc file or [app credential](https://docs.openstack.org/keystone/yoga/user/application_credentials.html) and run:
``` shell
openstack compute service list
```

This will list all nova services and show if any are down or not running.  A "XXX" status simply means that service has missed a heartbeat message in rabbitmq to the control node.  A "down" status means the service has missed enoughheartbeats to be considered off or problematic.

Nova errors will generally be the case where you are getting "No valid hosts available", when there should be hosts available.  This will simply be the case where we see a dead service on some/all nodes.  Running through the checks will show any/all down nodes.  OpenStack won't deploy new VMs to nova-compute services that are down.  Attempting a restart should bring the services back online or show some errors in the logs about what is causing the nova-compute service to go down.

Eventually, you will have to take some compute nodes offline for maintenance in some capacity.  The first thing you want to do, is set those nova-computes to disabled:
``` shell
openstack compute service set --disable --disable-reason maintenance <node hostname> nova-compute
```
Then, check to see if the node has VMs running on it.  You can do this with:
``` shell
openstack host show <node hostname>
```
If this has any CPU/Memory/Disk used, it probably has VMs running on it and they need to be migrated to other nodes.  We can do this with:
``` shell
nova host-evacuate-live <node hostname>
```
This will send the VMs on that node to other available nodes.  To verify this completed, check the node again with:
``` shell
openstack host show <node hostname>
```
And verify that it has no VMs running on it. Once they are migrated, the node will be out of production and available for the maintenance required.  After said maintenance is completed, you can bring the node back into production with:
``` shell
openstack compute service set --enable <node hostname> nova-compute
```

#### Neutron

To check the Neutron services status, source the Admin openrc file or app credential and run:

``` shell
openstack network agent list
```

This is similar to the previous command, except it shows all the neutron services.  If users are reporting that their VMs are having networking woes, this could be a good indicator that Neutron is struggling.  Running the check command can give you an idea of how the Neutron services are operating.  If you see some dead services or an isolated node that is exclusively having issues, you should restart the service and see if that alleviates the issue.  Also, you may want to check the logs to see what caused the outage.  Restarting the Neutron services on the controller nodes leads to extensive downtime, depending on how many networks exist on your cloud, as it takes an amount of time per network to rebuild each network.  Restarting these services are only recommended as a last resort.
