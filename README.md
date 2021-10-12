# Docker Desktop Replacement using Vagrant

 :exclamation:  Please support software authors.

I strongly believe in paying for software so it can be sustainably 
developed. I hope my organization and many others pay the commercial fee for 
Docker Desktop. I want it to thrive. This write-up is intended for those 
seeking alternatives for hopefully great reasons.

## User Experience Motivation

After Docker [announced changes to the Docker Desktop licensing policy](https://www.docker.com/blog/updating-product-subscriptions/), 
I saw multiple inquiries seeking altnernatives for a local development 
experience using Docker that was very similar to Docker Desktop. When I first 
started using Docker, I used virtual machines and it worked pretty well. So I 
wanted to see how close I could get to a good experience. 

I want a local docker executable on my Mac terminal to `Just work.` :tm: 

I am happy with the results. Using this approach, running docker locally is as 
simple as `vagrant up` followed by using `docker` commands directly in my 
host shell.

Using a remote docker daemon running in a Vagrant VM and connecting to it 
with simple port mapping, it's almost as simple as the Docker Desktop 
experience. Vagrant provides file system sharing, so even if 
you `vagrant ssh` into the Linux guest, you can pass files around simply 
between the host (say a Mac) and the Linux guest.

### Step 1 - Launch Docker Deamon in a VM using Vagrant

Vagrant supports a [Docker Provisioner](https://www.vagrantup.com/docs/provisioning/docker) 
that installs Docker into a Virtual Machine. I tried Ubuntu 20.04 as the guest 
image (Thanks Chef Bento project!). Notably, the `bento` Vagrant box had all of 
the base requirements to do file sharing between the host and the guest. Some 
other Ubuntu Vagrant boxes I tried did not work with file sharing.

Set the current working directory where you cloned this repository and use a 
`Vagrantfile` like the following:

```sh
Vagrant.configure("2") do |config|
  config.vm.box = "bento/ubuntu-20.04"
  config.vm.provision "docker"
  config.vm.network "forwarded_port", guest: 2375, host: 2375   # docker port
  config.vm.synced_folder ".", "/vagrant"
end
```

Then start the guest VM, where Vagrant will install docker.

`vagrant up` 

### Step 2 - Configure a Docker Network Listener for the Guest VM

Adjust the default settings for the Docker Provisioner so docker daemon listens 
on port 2375 and port-forwarding can work on the vagrant host.

From a host terminal, get a prompt in the Ubuntu guest VM running Docker:

`vagrant ssh` 

Open an editing session for the dockerd service.

`sudo systemctl --full edit docker.service`

Adjust the `dockerd` line so it listens on port 2375.

`ExecStart=/usr/bin/dockerd -H tcp://0.0.0.0:2375 -H fd:// --containerd=/run/containerd/containerd.sock`

Restart docker:

`sudo systemctl restart docker`

### Step 3 - Build a Docker executable for Mac

The docker binary for Mac is distributed as part of the Docker Desktop package 
and is readily found prebuilt as a standalone binary. You can also compile it 
on your own with [these steps](https://www.reddit.com/r/docker/comments/mooiqh/install_cli_only_on_mac/). Someone in you organization could do this step and distribute the binary to others since 
compling it yourself is a bit much to expect for every developer to do on their 
own.

You may perform these steps in the Linux guest if you do not have access to a 
docker executable for Mac. Using the `vagrant ssh` command from your host will 
give you a prompt.

Clone the repo somewhere, I did these steps in the `/tmp` directory: 

`git clone https://github.com/docker/cli.git`

Change directories into the cli repository:

`cd cli`

Checkout a [recent tag](https://github.com/docker/cli/tags), mine was: v20.10.9:

`git checkout v20.10.9`

Build the binary for your target platform:

`docker buildx bake --set binary.platform=darwin/amd64`

Get the built binary in the ./build directory and move it somewhere in your 
path. Since I built the docker for Mac in the Linux guest VM, I moved it to 
guest directory `/vagrant` so that the `docker` executable is now visible on my 
Mac host.

`vagrant@vagrant:/tmp/cli$ mv ./build/docker-darwin-amd64 /vagrant/docker`

### Step 4 - Using the Docker executable for Mac with the Remote Docker

Test the binary. In the directory with the `Vagrantfile`, look for the `docker` 
binary. Set the `DOCKER_HOST`environment variable so that docker makes a 
network connection to local port 2375.

`export DOCKER_HOST=tcp://127.0.0.1:2375`

The version command should show that the binary was just built and show both 
the client and server versions.

`./docker version`

If the version command works, move the `docker` executable somewhere on your 
path. I used `/usr/local/bin` since that is part of my host `$PATH`.

`mv ./docker /usr/local/bin`

Optionally, decide to make `DOCKER_HOST` set in future shells by configuring a 
`~/.bash_profile` or equivalent.

`export DOCKER_HOST=tcp://127.0.0.1:2375`

## Daily Usage

If you try and use docker without the Guest VM launched you'll get an error 
very similar to when Docker Desktop isn't running.

`Cannot connect to the Docker daemon at tcp://127.0.0.1:2375. Is the docker daemon running?`

Typically a `vagrant up` issued from the location of the `Vagrantfile` will 
start Docker. Occasionally, Vagrant will inform you that an updated box is 
available. If you download a new Box, you will need to perform step 2 again.
