# CoreOS

## References & Thanks
These are partial list of learning material. Thank you to all the authors and speakerts.

- [Quick Start from coreos](https://coreos.com/docs/quickstart/)
- [SaltStack Salt Lake City meetup with CoreOS - Alex Polvi](https://www.youtube.com/watch?v=eDQSpKdEZf4)
- [Intro to CoreOS - Rackspace Tech Talks](https://www.youtube.com/watch?v=l4oaIW37tU4) and repo at [github](https://github.com/bradgignac/intro-to-coreos)
- [Getting Started with Docker](https://serversforhackers.com/articles/2014/03/20/getting-started-with-docker/?utm_source=CenturyLink+Labs+List&utm_campaign=d05ba99a16-Weekly_Lab_Blast_October_1_2014&utm_medium=email&utm_term=0_d971fc65c9-d05ba99a16-134076965) good walk thru 
- [Install on bare metal](https://www.youtube.com/watch?v=vy6hWsOuCh8)

[thanks to many other authors, developers, ...]

## Setup

### depedency
- download Virtualbox >= 4.3.10 and Vagrant >= 1.6

### clone the repo

```
git clone https://github.com/coreos/coreos-vagrant/
cd coreos-vagrant
```

### Vagrantfile
- make $number_instance=3
- uncomment the line to share folder
- if you use Virtualbox, remove vmware providing

```
coreos/coreos-vagrant [master●] » Cat Vagrantfile
# -*- mode: ruby -*-
# # vi: set ft=ruby :

require 'fileutils'

Vagrant.require_version ">= 1.6.0"

CLOUD_CONFIG_PATH = File.join(File.dirname(__FILE__), "user-data")
CONFIG = File.join(File.dirname(__FILE__), "config.rb")

# Defaults for config options defined in CONFIG
$num_instances = 3
$update_channel = "alpha"
$enable_serial_logging = false
$vb_gui = false
$vb_memory = 1024
$vb_cpus = 1

# Attempt to apply the deprecated environment variable NUM_INSTANCES to
# $num_instances while allowing config.rb to override it
if ENV["NUM_INSTANCES"].to_i > 0 && ENV["NUM_INSTANCES"]
  $num_instances = ENV["NUM_INSTANCES"].to_i
end

if File.exist?(CONFIG)
  require CONFIG
end

Vagrant.configure("2") do |config|
  config.vm.box = "coreos-%s" % $update_channel
  config.vm.box_version = ">= 308.0.1"
  config.vm.box_url = "http://%s.release.core-os.net/amd64-usr/current/coreos_production_vagrant.json" % $update_channel

  config.vm.provider :virtualbox do |v|
    # On VirtualBox, we don't have guest additions or a functional vboxsf
    # in CoreOS, so tell Vagrant that so it can be smarter.
    v.check_guest_additions = false
    v.functional_vboxsf     = false
  end

  # removed vmware provider here

  # plugin conflict
  if Vagrant.has_plugin?("vagrant-vbguest") then
    config.vbguest.auto_update = false
  end

  (1..$num_instances).each do |i|
    config.vm.define vm_name = "core-%02d" % i do |config|
      config.vm.hostname = vm_name

      if $enable_serial_logging
        logdir = File.join(File.dirname(__FILE__), "log")
        FileUtils.mkdir_p(logdir)

        serialFile = File.join(logdir, "%s-serial.txt" % vm_name)
        FileUtils.touch(serialFile)

        config.vm.provider :virtualbox do |vb, override|
          vb.customize ["modifyvm", :id, "--uart1", "0x3F8", "4"]
          vb.customize ["modifyvm", :id, "--uartmode1", serialFile]
        end
      end

      if $expose_docker_tcp
        config.vm.network "forwarded_port", guest: 2375, host: ($expose_docker_tcp + i - 1), auto_correct: true
      end

      config.vm.provider :virtualbox do |vb|
        vb.gui = $vb_gui
        vb.memory = $vb_memory
        vb.cpus = $vb_cpus
      end

      ip = "172.17.8.#{i+100}"
      config.vm.network :private_network, ip: ip
      # Uncomment below to enable NFS for sharing the host machine into the coreos-vagrant VM.
      #config.vm.synced_folder ".", "/home/core/share", id: "core", :nfs => true, :mount_options => ['nolock,vers=3,udp']

      if File.exist?(CLOUD_CONFIG_PATH)
        config.vm.provision :file, :source => "#{CLOUD_CONFIG_PATH}", :destination => "/tmp/vagrantfile-user-data"
        config.vm.provision :shell, :inline => "mv /tmp/vagrantfile-user-data /var/lib/coreos-vagrant/", :privileged => true
      end

      config.ssh.forward_agent = true
    end
  end
end
```


### user-data file:
- make copy of user-data.sample to user-data, and modify it as below.
- get token `https://discovery.etcd.io/new` and copy the token to user-data. In new version of github repo, this portion has been automated with a scripts in config.rb. Just uncomment those lines, or copy the token into file.

```
#cloud-config
coreos:
  etcd:
    # generate a new token for each unique cluster from https://discovery.etcd.io/new
    # WARNING: replace each time you 'vagrant destroy'
    discovery: https://discovery.etcd.io/<token>       <= replace token with your return
    addr: $public_ipv4:4001
    peer-addr: $public_ipv4:7001
  fleet:
    public-ip: $public_ipv4
  units:
    - name: etcd.service
      command: start
    - name: fleet.service
      command: start
```

### config.rb
- make a copy of config.rb.sample to config.rb, and modify it. This many eliminates the need of manually get a token.

```
# To automatically replace the discovery token on 'vagrant up', uncomment
# the lines below:
#
if File.exists?('user-data') && ARGV[0].eql?('up')
  require 'open-uri'
  require 'yaml'

  token = open('https://discovery.etcd.io/new').read

  data = YAML.load(IO.readlines('user-data')[1..-1].join)
  data['coreos']['etcd']['discovery'] = token

  lines = YAML.dump(data).split("\n")
  lines[0] = '#cloud-config'

  open('user-data', 'r+') do |f|
    f.puts(lines.join("\n"))
  end
end
#
# coreos-vagrant is configured through a series of configuration
# options (global ruby variables) which are detailed below. To modify
# these options, first copy this file to "config.rb". Then simply
# uncomment the necessary lines, leaving the $, and replace everything
# after the equals sign..

# Size of the CoreOS cluster created by Vagrant
$num_instances=3

# Official CoreOS channel from which updates should be downloaded
$update_channel='alpha'

# Log the serial consoles of CoreOS VMs to log/
# Enable by setting value to true, disable with false
# WARNING: Serial logging is known to result in extremely high CPU usage with
# VirtualBox, so should only be used in debugging situations
#$enable_serial_logging=false

# Enable port forwarding of Docker TCP socket
# Set to the TCP port you want exposed on the *host* machine, default is 2375
# If 2375 is used, Vagrant will auto-increment (e.g. in the case of $num_instances > 1)
# You can then use the docker tool locally by setting the following env var:
#   export DOCKER_HOST='tcp://127.0.0.1:2375'
$expose_docker_tcp=2375
# Setting for VirtualBox VMs
$vb_gui = false
$vb_memory = 1024
$vb_cpus = 1
```

Since each time we destroy vm, we need to get new token; the new script in config.rb automated this part. 

### Setup ssh for vagrant
Execute the command below to make ssh work,
$ ssh-add ~/.vagrant.d/insecure_private_key

## Start Cluster

```
coreos/coreos-vagrant [master●] » vagrant up
Bringing machine 'core-01' up with 'virtualbox' provider...
Bringing machine 'core-02' up with 'virtualbox' provider...
Bringing machine 'core-03' up with 'virtualbox' provider...
==> core-01: Importing base box 'coreos-alpha'...
==> core-01: Matching MAC address for NAT networking...
==> core-01: Checking if box 'coreos-alpha' is up to date...
==> core-01: Setting the name of the VM: coreos-vagrant_core-01_1412391751988_56901
==> core-01: Clearing any previously set network interfaces...
==> core-01: Preparing network interfaces based on configuration...
    core-01: Adapter 1: nat
    core-01: Adapter 2: hostonly
==> core-01: Forwarding ports...
    core-01: 2375 => 2375 (adapter 1)
    core-01: 22 => 2222 (adapter 1)
==> core-01: Running 'pre-boot' VM customizations...
==> core-01: Booting VM...
==> core-01: Waiting for machine to boot. This may take a few minutes...
    core-01: SSH address: 127.0.0.1:2222
    core-01: SSH username: core
    core-01: SSH auth method: private key
    core-01: Warning: Connection timeout. Retrying...
==> core-01: Machine booted and ready!
==> core-01: Setting hostname...
==> core-01: Configuring and enabling network interfaces...
==> core-01: Running provisioner: file...
==> core-01: Running provisioner: shell...
    core-01: Running: inline script
==> core-02: Importing base box 'coreos-alpha'...
```
To share a host folder, you need to enter host password.

## Log in
To login in one container after the cluster is up,

```
coreos/coreos-vagrant [master●] » vagrant ssh core-01
CoreOS (alpha)
core@core-01 ~ $ ls -a
.  ..  .bash_logout  .bash_profile  .bashrc  .ssh  share
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
0cf0f17a...	172.17.8.102	-
9c05f59f...	172.17.8.101	-
fe71c27e...	172.17.8.103	-
core@core-01 ~ $
```

Just to get a status; docker is working, but everything is blank.

```
core@core-01 ~ $ docker ps
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
core@core-01 ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS              PORTS               NAMES
core@core-01 ~ $ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
core@core-01 ~ $ docker info
Containers: 0
Images: 0
Storage Driver: btrfs
Execution Driver: native-0.2
Kernel Version: 3.16.2+
Operating System: CoreOS 459.0.0
core@core-01 ~ $
```

### test data sharing
From [Getting Started with etcd](https://coreos.com/docs/distributed-configuration/getting-started-with-etcd/)

Put data in and read back at same core host (vm+ extra features)

```
core@core-01 ~ $ etcdctl ls --recursive
/coreos.com
/coreos.com/updateengine
/coreos.com/updateengine/rebootlock
/coreos.com/updateengine/rebootlock/semaphore
core@core-01 ~ $ etcdctl set /message Hello
Hello
core@core-01 ~ $ etcdctl get /message
Hello
core@core-01 ~ $ etcdctl ls --recursive
/coreos.com
/coreos.com/updateengine
/coreos.com/updateengine/rebootlock
/coreos.com/updateengine/rebootlock/semaphore
/message
```

Read it from different core host, works.

```
core@core-01 ~ $ exit
Connection to 127.0.0.1 closed.
coreos/coreos-vagrant [master●] » vagrant ssh core-02
CoreOS (alpha)
core@core-02 ~ $ etcdctl get /message
Hello
core@core-02 ~ $
```
Remove data from etcd,

```
core@core-02 ~ $ etcdctl rm /message

core@core-02 ~ $ exit
logout
Connection to 127.0.0.1 closed.
coreos/coreos-vagrant [master●] » vagrant ssh core-01
Last login: Sun Oct  5 23:08:06 2014 from 10.0.2.2
CoreOS (alpha)
core@core-01 ~ $ etcdctl get /message
Error:  100: Key not found (/message) [6665]
core@core-01 ~ $
```

### get ip address of a core host

```
core@core-02 ~ $ ip address show
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host
       valid_lft forever preferred_lft forever
2: enp0s3: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:c9:17:8c brd ff:ff:ff:ff:ff:ff
    inet 10.0.2.15/24 brd 10.0.2.255 scope global dynamic enp0s3
       valid_lft 77661sec preferred_lft 77661sec
    inet6 fe80::a00:27ff:fec9:178c/64 scope link
       valid_lft forever preferred_lft forever
3: enp0s8: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP qlen 1000
    link/ether 08:00:27:71:95:1b brd ff:ff:ff:ff:ff:ff
    inet 172.17.8.102/24 brd 172.17.8.255 scope global enp0s8
       valid_lft forever preferred_lft forever
    inet6 fe80::a00:27ff:fe71:951b/64 scope link
       valid_lft forever preferred_lft forever
core@core-02 ~ $
```
We have `inet 172.17.8.102`, because we assigned ip prefix in Vagrantfile `ip = "172.17.8.#{i+100}`

### install or run a container

We listed our containers before, nothing is there. We will do a test run `busybox`,

```
core@core-01 ~ $ docker run busybox /bin/echo hi
Unable to find image 'busybox' locally
Pulling repository busybox
e72ac664f4f0: Download complete
511136ea3c5a: Download complete
df7546f9f060: Download complete
e433a6c5b276: Download complete
hi
core@core-01 ~ $ docker ps -a
CONTAINER ID        IMAGE               COMMAND             CREATED             STATUS                      PORTS               NAMES
5e9cf530c804        busybox:latest      "/bin/echo hi"      39 seconds ago      Exited (0) 38 seconds ago                       elegant_leakey
core@core-01 ~ $ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
busybox             latest              e72ac664f4f0        4 days ago          2.433 MB
core@core-01 ~ $
```

### install a service

To find out service status; nothing,

```
core@core-01 ~ $ fleetctl list-unit-files
UNIT	HASH	DSTATE	STATE	TARGET
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
0cf0f17a...	172.17.8.102	-
9c05f59f...	172.17.8.101	-
fe71c27e...	172.17.8.103	-
core@core-01 ~ $ fleetctl list-units
UNIT	MACHINE	ACTIVE	SUB
core@core-01 ~ $
```

From system [doc](https://coreos.com/docs/launching-containers/launching/getting-started-with-systemd/), unit files are located within the R/W filesystem at `/etc/systemd/system`.  Got error,

#### hello.service
Followed [quick start](https://coreos.com/docs/quickstart/),

Define a service. Multiple instances can be run at different ports. 

```
core@core-01 /etc/systemd/system $ cat hello.service
[Unit]
Description=My Service
After=docker.service

[Service]
TimeoutStartSec=0
ExecStartPre=-/usr/bin/docker kill hello
ExecStartPre=-/usr/bin/docker rm hello
ExecStartPre=/usr/bin/docker pull busybox
ExecStart=/usr/bin/docker run --name hello busybox /bin/sh -c "while true; do echo Hello World; sleep 1; done"
ExecStop=/usr/bin/docker stop hello
core@core-01 /etc/systemd/system $
```

Enable the service, I may not need to do this,

```
core@core-01 /etc/systemd/system $ sudo systemctl enable /etc/systemd/system/hello.service
The unit files have no [Install] section. They are not meant to be enabled
using systemctl.
Possible reasons for having this kind of units are:
1) A unit may be statically enabled by being symlinked from another unit's
   .wants/ or .requires/ directory.
2) A unit's purpose may be to act as a helper for some other unit which has
   a requirement dependency on it.
3) A unit may be started when needed via activation (socket, path, timer,
   D-Bus, udev, scripted systemctl call, ...).
core@core-01 /etc/systemd/system $ $ sudo systemctl start hello.service
-bash: $: command not found
core@core-01 /etc/systemd/system $
```

Load hello.service, (submit and load may got reversed here.)

```
core@core-01 /etc/systemd/system $ fleetctl load hello.service
Unit hello.service loaded on 0cf0f17a.../172.17.8.102
core@core-01 /etc/systemd/system $ fleetctl start hello.service
Unit hello.service launched on 0cf0f17a.../172.17.8.102
core@core-01 /etc/systemd/system $
```
Submit a service. I may not put the service at right place.

```
core@core-01 /etc/systemd/system $ sudo vi hello.service
core@core-01 /etc/systemd/system $ sudo systemctl enable /etc/systemd/system/hello.service
The unit files have no [Install] section. They are not meant to be enabled
using systemctl.
Possible reasons for having this kind of units are:
1) A unit may be statically enabled by being symlinked from another unit's
   .wants/ or .requires/ directory.
2) A unit's purpose may be to act as a helper for some other unit which has
   a requirement dependency on it.
3) A unit may be started when needed via activation (socket, path, timer,
   D-Bus, udev, scripted systemctl call, ...).
core@core-01 /etc/systemd/system $ $ sudo systemctl start hello.service
-bash: $: command not found
core@core-01 /etc/systemd/system $ fleetctl load hello.service
Unit hello.service loaded on 0cf0f17a.../172.17.8.102
core@core-01 /etc/systemd/system $ fleetctl start hello.service
Unit hello.service launched on 0cf0f17a.../172.17.8.102
core@core-01 /etc/systemd/system $ fleetctl status hello.service
The authenticity of host '172.17.8.102' can't be established.
RSA key fingerprint is 54:2f:44:ae:8b:a0:8a:23:c0:a1:fd:ff:81:d7:81:db.
Are you sure you want to continue connecting (yes/no)? Error running remote command: timed out while initiating SSH connection
```
Get status of a service,

```
core@core-01 /etc/systemd/system $ fleetctl status hello.service
The authenticity of host '172.17.8.102' can't be established.
RSA key fingerprint is 54:2f:44:ae:8b:a0:8a:23:c0:a1:fd:ff:81:d7:81:db.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.8.102' (RSA) to the list of known hosts.
● hello.service - My Service
   Loaded: loaded (/run/fleet/units/hello.service; linked-runtime)
   Active: active (running) since Mon 2014-10-06 03:54:48 UTC; 2min 39s ago
  Process: 3512 ExecStartPre=/usr/bin/docker pull busybox (code=exited, status=0/SUCCESS)
  Process: 3504 ExecStartPre=/usr/bin/docker rm hello (code=exited, status=1/FAILURE)
  Process: 3441 ExecStartPre=/usr/bin/docker kill hello (code=exited, status=1/FAILURE)
 Main PID: 3524 (docker)
   CGroup: /system.slice/hello.service
           └─3524 /usr/bin/docker run --name hello busybox /bin/sh -c while true; do echo Hello World; sleep 1; done

Oct 06 03:57:18 core-02 docker[3524]: Hello World
Oct 06 03:57:19 core-02 docker[3524]: Hello World
Oct 06 03:57:20 core-02 docker[3524]: Hello World
Oct 06 03:57:21 core-02 docker[3524]: Hello World
Oct 06 03:57:22 core-02 docker[3524]: Hello World
Oct 06 03:57:23 core-02 docker[3524]: Hello World
Oct 06 03:57:24 core-02 docker[3524]: Hello World
Oct 06 03:57:25 core-02 docker[3524]: Hello World
Oct 06 03:57:26 core-02 docker[3524]: Hello World
Oct 06 03:57:27 core-02 docker[3524]: Hello World
core@core-01 /etc/systemd/system $
```
Why core-02, not core-01? 

To stop a service, the file is still there.

```
core@core-01 /etc/systemd/system $ fleetctl stop hello.service
Unit hello.service loaded on 0cf0f17a.../172.17.8.102
core@core-01 /etc/systemd/system $ ls
docker-tcp.socket  hello.service  sockets.target.wants
```

If we add a `@` at end of hello, hello@.service, it indicates it is a template.

#### dillinger.service

Get file or data from shared drive,

```
core@core-01 ~ $ ls share -la
total 52
drwxr-xr-x 14  501   20  476 Oct  6 18:55 .
drwxr-xr-x  1 core core  126 Oct  6 03:57 ..
drwxr-xr-x 14  501   20  476 Oct 12 16:18 .git
-rw-r--r--  1  501   20  117 Oct  3 15:59 .gitattributes
-rw-r--r--  1  501   20   15 Oct  6 03:17 .gitignore
drwxr-xr-x  3  501   20  102 Oct  3 16:01 .vagrant
-rw-r--r--  1  501   20 4087 Oct  3 15:59 README.md
-rw-r--r--  1  501   20 2830 Oct  5 23:00 Vagrantfile
-rw-r--r--  1  501   20 1357 Oct  5 21:22 config.rb
-rw-r--r--  1  501   20 1679 Oct  3 15:59 config.rb.sample
drwxr-xr-x  3  501   20  102 Oct  6 23:52 dillinger
-rw-rw-r--  1  501   20  349 Oct  6 03:27 hello.service
-rw-r--r--  1  501   20  616 Oct 12 12:39 user-data
-rw-r--r--  1  501   20  722 Oct  3 15:59 user-data.sample
core@core-01 ~ $
```
Submit -> Start a service. The service will be loaded to one host. 

```
core@core-02 ~/share/dillinger $ fleetctl submit dillinger.service
core@core-02 ~/share/dillinger $ fleetctl list-units
UNIT		MACHINE				ACTIVE		SUB
hello.service	fe71c27e.../172.17.8.103	inactive	dead
core@core-02 ~/share/dillinger $ fleetctl start dillinger.service
Unit dillinger.service launched on 0cf0f17a.../172.17.8.102
core@core-02 ~/share/dillinger $
```

find out actually what happen in log,

```
core@core-02 ~/share/dillinger $ fleetctl journal dillinger.service
-- Logs begin at Sun 2014-10-05 23:05:48 UTC, end at Tue 2014-10-07 00:28:27 UTC. --
Oct 07 00:25:08 core-02 systemd[1]: Starting dillinger.io...
Oct 07 00:25:08 core-02 systemd[1]: Started dillinger.io.
Oct 07 00:25:08 core-02 docker[13738]: Unable to find image 'dscape/dillinger' locally
Oct 07 00:25:08 core-02 docker[13738]: Pulling repository dscape/dillinger
Oct 07 00:25:08 core-02 systemd[1]: dillinger.service: main process exited, code=exited, status=1/FAILURE
Oct 07 00:25:08 core-02 systemd[1]: Unit dillinger.service entered failed state.
Oct 07 00:25:08 core-02 docker[13738]: 2014/1
```
*I was not online!* do it again.

```
core@core-02 ~/share/dillinger $ fleetctl list-units
UNIT			MACHINE				ACTIVE		SUB
dillinger.service	0cf0f17a.../172.17.8.102	failed		failed
hello.service		fe71c27e.../172.17.8.103	inactive	dead
core@core-02 ~/share/dillinger $
```

start service,

```
core@core-02 ~/share/dillinger $ fleetctl destroy dillinger.service
Destroyed dillinger.service
core@core-02 ~/share/dillinger $ fleetctl submit dillinger.service
core@core-02 ~/share/dillinger $ fleetctl list-units
UNIT		MACHINE				ACTIVE		SUB
hello.service	0cf0f17a.../172.17.8.102	inactive	dead
core@core-02 ~/share/dillinger $ fleetctl load dillinger.service
Unit dillinger.service loaded on 9c05f59f.../172.17.8.101
core@core-02 ~/share/dillinger $ fleetctl list-units
UNIT			MACHINE				ACTIVE		SUB
dillinger.service	9c05f59f.../172.17.8.101	inactive	dead
hello.service		0cf0f17a.../172.17.8.102	inactive	dead
core@core-02 ~/share/dillinger $ fleetctl start dillinger.service
Unit dillinger.service launched on 9c05f59f.../172.17.8.101
core@core-02 ~/share/dillinger $ fleetctl journal dillinger.service
The authenticity of host '172.17.8.101' can't be established.
RSA key fingerprint is 34:7e:cc:b8:62:f2:64:9e:3d:b9:62:2e:86:86:e8:a7.
Are you sure you want to continue connecting (yes/no)? yes
Warning: Permanently added '172.17.8.101' (RSA) to the list of known hosts.
-- Logs begin at Sun 2014-10-05 23:04:57 UTC, end at Tue 2014-10-07 14:14:30 UTC. --
Oct 07 14:14:06 core-01 systemd[1]: Starting dillinger.io...
Oct 07 14:14:06 core-01 systemd[1]: Started dillinger.io.
Oct 07 14:14:06 core-01 docker[18735]: Unable to find image 'dscape/dillinger' locally
Oct 07 14:14:06 core-01 docker[18735]: Pulling repository dscape/dillinger
core@core-02 ~/share/dillinger $ fleetctl status dillinger.service
● dillinger.service - dillinger.io
   Loaded: loaded (/run/fleet/units/dillinger.service; linked-runtime)
   Active: active (running) since Tue 2014-10-07 14:14:06 UTC; 3min 6s ago
 Main PID: 18735 (docker)
   CGroup: /system.slice/dillinger.service
           └─18735 /usr/bin/docker run -p 3000:8080 dscape/dillinger

Oct 07 14:14:06 core-01 systemd[1]: Starting dillinger.io...
Oct 07 14:14:06 core-01 systemd[1]: Started dillinger.io.
Oct 07 14:14:06 core-01 docker[18735]: Unable to find image 'dscape/dillinger' locally
Oct 07 14:14:06 core-01 docker[18735]: Pulling repository dscape/dillinger
core@core-02 ~/share/dillinger $ fleetctl list-units
UNIT			MACHINE				ACTIVE		SUB
dillinger.service	9c05f59f.../172.17.8.101	active		running
hello.service		0cf0f17a.../172.17.8.102	inactive	dead
core@core-02 ~/share/dillinger $
```

### get token from existing cluster
`grep DISCOVERY /run/systemd/system/etcd.service.d/20-cloudinit.conf`

```
core@core-02 ~ $ grep DISCOVERY /run/systemd/system/etcd.service.d/20-cloudinit.conf
Environment="ETCD_DISCOVERY=https://discovery.etcd.io/0af370d6b7051b3b483021eb25d75b66"
core@core-02 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
0cf0f17a...	172.17.8.102	-
9c05f59f...	172.17.8.101	-
fe71c27e...	172.17.8.103	-
core@core-02 ~ $
```
### remove a host and rejoin the cluster

Two service is run on 03,

```
core@core-01 ~ $ fleetctl list-units
UNIT			MACHINE				ACTIVE	SUB
dillinger.service	fe71c27e.../172.17.8.103	active	running
hello.service		fe71c27e.../172.17.8.103	active	running
core@core-01 ~ $
```


In coreos-host, `exit` 
then in Mac `vagrant halt core-03` to stop one core-host (vm); 

```
Connection to 127.0.0.1 closed.
coreos/coreos-vagrant [master●] » vagrant halt core-03
==> core-03: Attempting graceful shutdown of VM...
coreos/coreos-vagrant [master●] »
```

the service on core-03 will be automatically re-launched on different host.

```
coreos/coreos-vagrant [master●] » vagrant ssh core-01
Last login: Sun Oct 12 16:02:15 2014 from 10.0.2.2
CoreOS (alpha)
core@core-01 ~ $ fleetctl list-units
UNIT			MACHINE				ACTIVE	SUB
dillinger.service	0cf0f17a.../172.17.8.102	failed	failed
hello.service		9c05f59f.../172.17.8.101	active	running
core@core-01 ~ $
```

I do not know why only one service started, an the other did not; check Dockerfile later???.

I have trouble to manually start it.

```
core@core-01 ~ $ fleetctl start dillinger.service
core@core-01 ~ $ fleetctl list-units
UNIT			MACHINE				ACTIVE	SUB
dillinger.service	0cf0f17a.../172.17.8.102	failed	failed
hello.service		9c05f59f.../172.17.8.101	active	running
core@core-01 ~ $ fleetctl status dillinger.service
Error running remote command: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none publickey], no supported methods remain
core@core-01 ~ $
```

To re-launch the host, `vagrant up core-03`, 

```
coreos/coreos-vagrant [master●] » vagrant up core-03
Bringing machine 'core-03' up with 'virtualbox' provider...
==> core-03: Checking if box 'coreos-alpha' is up to date...
==> core-03: Clearing any previously set forwarded ports...
==> core-03: Fixed port collision for 22 => 2222. Now on port 2201.
==> core-03: Clearing any previously set network interfaces...
==> core-03: Preparing network interfaces based on configuration...
    core-03: Adapter 1: nat
    core-03: Adapter 2: hostonly
==> core-03: Forwarding ports...
    core-03: 2375 => 2377 (adapter 1)
    core-03: 22 => 2201 (adapter 1)
==> core-03: Running 'pre-boot' VM customizations...
==> core-03: Booting VM...
==> core-03: Waiting for machine to boot. This may take a few minutes...
    core-03: SSH address: 127.0.0.1:2201
    core-03: SSH username: core
    core-03: SSH auth method: private key
    core-03: Warning: Connection timeout. Retrying...
==> core-03: Machine booted and ready!
==> core-03: Setting hostname...
==> core-03: Configuring and enabling network interfaces...
==> core-03: Exporting NFS shared folders...
==> core-03: Preparing to edit /etc/exports. Administrator privileges will be required...
Password:
==> core-03: Mounting NFS shared folders...
==> core-03: Machine already provisioned. Run `vagrant provision` or use the `--provision`
==> core-03: to force provisioning. Provisioners marked to run always will still run.
coreos/coreos-vagrant [master●] »
```
The previous failure is possible due to shared drive requires passward during the startup, confirm it later.

The host will automativally join the cluster;

```
coreos/coreos-vagrant [master●] » vagrant ssh core-01
Last login: Sun Oct 12 19:42:21 2014 from 10.0.2.2
CoreOS (alpha)
core@core-01 ~ $ fleetctl list-machines
MACHINE		IP		METADATA
0cf0f17a...	172.17.8.102	-
9c05f59f...	172.17.8.101	-
fe71c27e...	172.17.8.103	-
```

## Conclusion

### Lesson learned

- CoreOS automated/unbundled many system admin task from DevOps, which makes DevOps much more productive. 
- CoreOS allows DevOps to customize the OS for the app, which makes overall architecture simplier and better performance.
- ssh knownhosts file is on host (Mac), which may require manual clean after host destryed.

### ToDo

- learn to auto deploy multiple apps with a script or a template,
- learn to create a cluster in Openstack,
- learn to create a cluster in DigitalOcean,
- learn to load balance the containers, monitor and metrix the services,
- learn go language,
- read more doc 
	- [CoreOS Cluster Architectures](https://coreos.com/docs/cluster-management/setup/cluster-architectures/)
	- [fleetctl](https://coreos.com/docs/launching-containers/launching/fleet-using-the-client/), 
	- [systemctl](???)
	- [Fleet](https://www.youtube.com/watch?v=jWeAJOQIjDM)
	- [Deploy on openstack](http://blog.scottlowe.org/2014/08/13/deploying-coreos-on-openstack-using-heat/)
	- [Google cloud](http://googlecloudplatform.blogspot.com/2014/05/official-coreos-images-are-now-available-on-google-compute-engine.html)

### Questions
- For a cluster, can we put core host on different clouds? with the same token? trouble shooting Digital Ocean?
- Can we form a cluster with different hw/datacenter and put Openstack on CoreOS? undercloud?
- How to add service before boot? [boot a service](https://coreos.com/docs/launching-containers/launching/launching-containers-fleet/)
- Why Dillinger did not started properly?
