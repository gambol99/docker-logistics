Mesos + Marathon on CoreOS
--------------------------
A short list / group of instructions to being

    # Being up the CoreOS cluster of you haven't done so already
  
    $ vagrant box update
    $ vagrant up
    # Check all the machines are there
    $ export FLEETCTL_ENDPOINT=http://10.0.1.101:4001
    $ fleetctl list-machines
  
    # if not, run the fixup, as i've had issues with the discovery service
    $ service/coreos-fixup
    $ fleetctl list-machines
  
    # Lets bring up some utility services (service discovery)
    $ cd services/
    $ fleetctl start consul-master.service
    $ fleetctl start consul-slave[12].service
    $ fleetctl start registrator.service

    # Docker Service Proxy for auto-wiring of containers
    $ fleetctl start embassy.service

    # Bring up the zookeeper instance (note, this needs to be clustered at some point)
    $ fleetctl start zookeeper.service
  
    # Bring up the mesos-master and slaves
    # fleetctl start mesos-master.service
    # ...
    # fleetctl list-units
  
    # Check which machine the master is running on and then open a browser http://<IP>:5050
  
    # Start up the slaves
    $ fleetctl start mesos-slave.service
  
    # View in master ui to check when the slaves are registered and ready to use. Once there, we can bring up the marathon cluster
    
    # launch some mesos applications
    alias launch="curl -X POST -H 'Content-Type: application/json' http://10.0.1.102:8080/v2/apps -d"
    $ launch @frontend.json

    $ fleetctl start marathon.service
    # Once done ... open a browser to any of the CoreOS machines: http://IP:8080 and you should be the Marathon UI.
  
    # There is a number of services in the services/demo directory;
      - a frontend connected to a redis cache and x number of mysql slaves
      - a backend which would be connected to a master and slave mysql
      - a database master and x number of slaves replicating
  
    note: All the above use the Service Proxy to auto-wire and load balancer the connections between them.

