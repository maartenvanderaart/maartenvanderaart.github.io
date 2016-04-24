---
layout: post
title:  "Docker Swarm on CoreOS"
date:   2016-04-18 20:15:00 +0200
categories: docker swarm coreos
---
Since Docker's [Swarm][docker-swarm] became stable with the release of Docker 1.9 last November, quite a few articles have been written discussing ways to setup a cluster for development purposes. None that I've found did so on a Linux desktop (they all use [Docker Machine][docker-machine]), yet this was my predicament and so for future reference and those that will come after me, here is a quick write-up of how I got a multi-node CoreOS cluster running, with Swarm on top.

I wanted to experiment with the new Docker Compose syntax and thought it a good idea to do so on a Swarm cluster. To not have to redeploy my cluster manually, I forked the excellent [coreos-vagrant][coreos-vagrant] and tweaked the configuration a bit so that a Swarm cluster starts up automagically when running ```vagrant up```. You can find my files [here][coreos-vagrant-fork], for reference.

There are two files you need to edit to make this work: ```config.rb``` and ```user-data```, the first is to tell Vagrant what to do, the second contains the cloud-config used to bootstrap the [etcd cluster][etcd] and configure systemd to start services. 

Copy the ```config.rb.sample``` to ```config.rb``` and change the forllowing lines (does not have to be exacly the same, but this was running nicely on my somewhat ageing T420):
{% highlight ruby %}
# Size of the CoreOS cluster created by Vagrant
# The minimum of number of instances for a decent cluster is 3
$num_instances=5
...
# Official CoreOS channel from which updates should be downloaded
# Living on the edge here (you want to experiment, right?)
$update_channel='alpha'
...
# You can then use the docker tool locally by setting the following env var:
#   export DOCKER_HOST='tcp://${public_ipv4}:2375'
$expose_docker_tcp=2375
...
# Customize VMs
#$vm_gui = false
$vm_memory = 1024
#$vm_cpus = 1
{% endhighlight %}

Next, create a file ```user-data``` with this content:
{% highlight yaml %}

#cloud-config

---
coreos:
  etcd2:
    # Discovery URL will be replaced on 'vagrant up'
    discovery: https://discovery.etcd.io/willbereplacedonvagrantup
    advertise-client-urls: http://$public_ipv4:2379
    initial-advertise-peer-urls: http://$public_ipv4:2380
    listen-client-urls: http://0.0.0.0:2379,http://0.0.0.0:4001
    listen-peer-urls: http://$public_ipv4:2380,http://$public_ipv4:7001
  units:
  - name: etcd2.service
    command: start
  - name: docker.service
    command: start
    drop-ins:
    - name: custom.conf
      content: |
        [Service]
        # Start the docker engine on a different port, so that it does not
        # conflict with the swarm-manager that will be running on 2375. This
        # way you can interact with the Swarm as if it were a regular instance
        # with tools like the Docker client or Compose. Also, tell the engine
        # to advertise itself via etcd.
        Environment="DOCKER_OPTS=-H=0.0.0.0:2376 -H unix:///var/run/docker.sock --cluster-advertise eth1:2376 --cluster-store etcd://127.0.0.1:2379"
  # Start a swarm-agent at boot.
  - name: swarm-agent.service
    command: start
    enable: true
    content: |
      [Unit]
      Description=Docker Swarm agent
      After=docker.service
      Requires=docker.service

      [Service]
      TimeoutStartSec=0
      ExecStartPre=-/usr/bin/docker kill swarm-agent
      ExecStartPre=-/usr/bin/docker rm swarm-agent
      ExecStartPre=/usr/bin/docker pull swarm:latest
      ExecStart=/usr/bin/docker run -d --name swarm-agent --net=host swarm:latest join --addr=$public_ipv4:2376 etcd://127.0.0.1:2379

      [Install]
      WantedBy=multi-user.target
  # Start a swarm-manager at boot.
  - name: swarm-manager.service
    command: start
    enable: true
    content: |
      [Unit]
      Description=Docker Swarm manager
      After=swarm-agent.service
      Requires=swarm-agent.service

      [Service]
      TimeoutStartSec=0
      ExecStartPre=-/usr/bin/docker kill swarm-manager
      ExecStartPre=-/usr/bin/docker rm swarm-manager
      ExecStartPre=/usr/bin/docker pull swarm:latest
      ExecStart=/usr/bin/docker run -d --name swarm-manager --net=host swarm:latest manage etcd://127.0.0.1:2379

      [Install]
      WantedBy=multi-user.target
{% endhighlight %}

As you can see, every node will start both an agent and a manager. The agent must run on every node, but the manager nodes will elect a leader among them. The other nodes will proxy command to and from this leader when spoken to. When the leader is no longer available, the remaining managers elect a new leader. This way the Swarm manager will be fault tolerant. 

Now that the configuration has been done, we can start our cluster:

{% highlight shell %}
maarten@clunky:~/swarm-vagrant$ vagrant up
Bringing machine 'core-01' up with 'virtualbox' provider...
Bringing machine 'core-02' up with 'virtualbox' provider...
Bringing machine 'core-03' up with 'virtualbox' provider...
Bringing machine 'core-04' up with 'virtualbox' provider...
Bringing machine 'core-05' up with 'virtualbox' provider...
==> core-01: Importing base box 'coreos-alpha'...
==> core-01: Matching MAC address for NAT networking...
==> core-01: Checking if box 'coreos-alpha' is up to date...
==> core-01: Setting the name of the VM: coreos-vagrant_core-01_1461007486758_85609
==> core-01: Clearing any previously set network interfaces...
==> core-01: Preparing network interfaces based on configuration...
    core-01: Adapter 1: nat
    core-01: Adapter 2: hostonly
==> core-01: Forwarding ports...
    core-01: 2375 (guest) => 2375 (host) (adapter 1)
    core-01: 22 (guest) => 2222 (host) (adapter 1)

# on and on it goes...

==> core-05: Running 'pre-boot' VM customizations...
==> core-05: Booting VM...
==> core-05: Waiting for machine to boot. This may take a few minutes...
    core-05: SSH address: 127.0.0.1:2203
    core-05: SSH username: core
    core-05: SSH auth method: private key
==> core-05: Machine booted and ready!
==> core-05: Setting hostname...
==> core-05: Configuring and enabling network interfaces...
==> core-05: Running provisioner: file...
==> core-05: Running provisioner: shell...
    core-05: Running: inline script
{% endhighlight %}

You can now export any of the nodes as DOCKER_HOST, like this (assuming a public ip of 172.17.8.101): ```export DOCKER_HOST=tcp://172.17.8.101:2375```. Note that the port is the default port, on which the Docker engine instances are not themselves listening on. You can ommit the port number, I just added it for illustration.

Now, using the client, we can ask for info:

{% highlight shell %}
maarten@clunky:~/swarm-vagrant$ docker info
Containers: 10
 Running: 10
 Paused: 0
 Stopped: 0
Images: 5
Server Version: swarm/1.2.0
Role: primary
Strategy: spread
Filters: health, port, dependency, affinity, constraint
Nodes: 5
 core-01: 172.17.8.101:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.5.0-coreos-r1, operatingsystem=CoreOS 1010.1.0 (MoreOS), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-04-18T19:44:13Z
  └ ServerVersion: 1.10.3
 core-02: 172.17.8.102:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.5.0-coreos-r1, operatingsystem=CoreOS 1010.1.0 (MoreOS), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-04-18T19:44:30Z
  └ ServerVersion: 1.10.3
 core-03: 172.17.8.103:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.5.0-coreos-r1, operatingsystem=CoreOS 1010.1.0 (MoreOS), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-04-18T19:43:51Z
  └ ServerVersion: 1.10.3
 core-04: 172.17.8.104:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.5.0-coreos-r1, operatingsystem=CoreOS 1010.1.0 (MoreOS), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-04-18T19:44:22Z
  └ ServerVersion: 1.10.3
 core-05: 172.17.8.105:2376
  └ Status: Healthy
  └ Containers: 2
  └ Reserved CPUs: 0 / 1
  └ Reserved Memory: 0 B / 1.022 GiB
  └ Labels: executiondriver=native-0.2, kernelversion=4.5.0-coreos-r1, operatingsystem=CoreOS 1010.1.0 (MoreOS), storagedriver=overlay
  └ Error: (none)
  └ UpdatedAt: 2016-04-18T19:44:32Z
  └ ServerVersion: 1.10.3
Plugins: 
 Volume: 
 Network: 
Kernel Version: 4.5.0-coreos-r1
Operating System: linux
Architecture: amd64
CPUs: 5
Total Memory: 5.112 GiB
Name: core-01

{% endhighlight %}

A healthy cluster! Now you can treat it as a regular Docker instance, except for the new cluster goodies that became available, such as the [overlay network driver][overlay-network]. Building and deploying applications is out of scope for this post, but would make for a fun next one :-)

[docker-swarm]:		https://www.docker.com/products/docker-swarm
[docker-machine]:	https://www.docker.com/products/docker-machine
[coreos-vagrant]:	https://github.com/coreos/coreos-vagrant
[etcd]:			https://coreos.com/etcd
[overlay-network]:	https://docs.docker.com/engine/userguide/networking/get-started-overlay
[coreos-vagrant-fork]:	https://github.com/maartenvanderaart/coreos-vagrant
