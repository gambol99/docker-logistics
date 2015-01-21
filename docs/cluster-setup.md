### **CoreOS Cluster Setup**
------

One of the main benefits of CoreOS - the cluster setup and indeed configuration in general is breathtakingly easy. Bring up the cluster as such. Note, if you want to make alterations to the size, version, cpu, memory etc of the cluster nodes, this can be done in config/coreos-config.rb

  $ vagrant box update # if need be
  $ vagrant up
  # Check all the machines are there
  $ export FLEETCTL_ENDPOINT=http://10.0.1.101:4001
  $ fleetctl list-machines

##### **Issues**

Occasionally and i'm not sure if it's me, the etcd discovery endpoint or CoreOS, but on occasion some of the boxes don't register with the discovery service (discovery.coreos.com); there's a simple script in the service/ directory to auto fix this. Effectively all it does it rerun the cloudinit setup.

  $ service/coreos-fixup
  # check the list again
  $ fleetctl list-machines
