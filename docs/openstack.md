## OpenStack CoreOS

I had a number of issues getting CoreOS working on OpenStack (Havana), though all of them were related to the same issue. Namely that $_private_ip on the cloudinit file was incorrectly retrieved and instead the private ip address I kept receiving the public floating ip. After a round of googling, here's how the problem was resolved. Essentially it's a two stage process, which first creates a correct /etc/environment file via a scripted delivery and then bootstraps real cloudinit file.


Note: Though the variables have been stripped out below, our bootstrap process parsed this below in a ERB template. The only changes that need to be dynamic are the hostname, domain, perhaps the discovery token etc.

      #cloud-config
    
      write_files:
        - path: /run/cloud-config.yml
          permissions: '0644'
          content: |
            #cloud-config
            coreos:
                update:
                  reboot-strategy: etcd-lock
                etcd:
                  discovery: https://discovery.etcd.io/<CHANGE FOR YOUR TOKEN>
                  # multi-region deployments, multi-cloud deployments, and droplets without
                  # private networking need to use $_private_ipv4
                  addr: $_private_ipv4:4001
                  peer-addr: $_private_ipv4:7001
                fleet:
                  # used for fleetctl ssh command
                  public-ip: $_private_ipv4
                units:
                  - name: etcd.service
                    command: start
                  - name: fleet.service
                    command: start
        - path: /run/setup-environment.sh
          permissions: '0755'
          content: |
            #!/usr/bin/bash
    
            ENV="/etc/environment"
    
            # Test for RW access to $1
            touch $ENV
            if [ $? -ne 0 ]; then
                echo exiting, unable to modify: $ENV
                exit 1
            fi
    
            # Setup environment target
            sed -i -e '/^COREOS_PUBLIC_IPV4=/d' \
                -e '/^COREOS_PRIVATE_IPV4=/d' \
                "${ENV}"
    
            # We spin loop until the nova-agent sets up the IP addresses
            function get_ip () {
                IF=$1
                IP=
                while [ 1 ]; do
                    IP=$(ifconfig $IF | awk '/inet / {print $2}')
                    if [ "$IP" != "" ]; then
                        break
                    fi
                    sleep .1
                done
                echo $IP
            }
    
            # Echo results of IP queries to environment file as soon as network interfaces
            # get assigned IPs
            echo COREOS_PUBLIC_IPV4=$(get_ip eth0) >> $ENV # Also assigned to same IP
            echo COREOS_PRIVATE_IPV4=$(get_ip eth0) >> $ENV
      coreos:
        units:
          - name: setup-environment.service
            command: start
            runtime: true
            content: |
              [Unit]
              Description=Setup environment with private (and public) IP addresses
    
              [Service]
              Type=oneshot
              RemainAfterExit=yes
              ExecStart=/run/setup-environment.sh
          - name: second-stage-cloudinit.service
            runtime: true
            command: start
            content: |
              [Unit]
              Description=Run coreos-cloudinit with actual cloud-config after environment has been set up
              Requires=setup-environment.service
              After=setup-environment.service
              Requires=user-cloudinit-proc-cmdline.service
              After=user-cloudinit-proc-cmdline.service
    
              [Service]
              Type=oneshot
              RemainAfterExit=yes
              EnvironmentFile=/etc/environment
              ExecStartPre=/usr/bin/sed -i 's/\$_private/$private/g' /run/cloud-config.yml
              ExecStart=/usr/bin/coreos-cloudinit --from-file=/run/cloud-config.yml
    
      manage-resolv-conf: true
      hostname: CHANGE_ME_THE_HOSTNAME
      resolv_conf:
        nameservers: [ 'DNS_SERVICE_IP_ADDRESS_CHANGE_ME' ]
        searchdomains:
          - changeme.domain.com
        domain: changeme.domain.com
        options:
          rotate: true
          timeout: 1
    
      ssh_authorized_keys:
        - <MY PUBLIC KEY>
