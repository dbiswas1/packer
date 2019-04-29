- [Who should read this Blog](#who-should-read-this-blog)
- [Short Introduction](#short-introduction)
    + [Packer](#packer)
    + [Ansible](#ansible)
- [Problem we are trying to solve](#problem-we-are-trying-to-solve)
- [Why not use Dockerfile rather than Packer](#why-not-use-dockerfile-rather-than-packer)
- [Environment Used for this Exercise](#environment-used-for-this-exercise)
- [Actual Implementation](#actual-implementation)
    + [STEP 1: Install Packer and Ansible](#step-1-install-packer-and-ansible)
    + [STEP 2: Build a sample image using Ansible and Packer](#step-2-build-a-sample-image-using-ansible-and-packer)
    + [STEP 3: Verify the Exercise](#step-3-verify-the-exercise)

## Who should read this Blog
This blog is continuation to the series [(Part 1)](https://blog.avmconsulting.net/posts/2019-04-07-setup-kubernetes-cluster-with-terraform-and-kops-part-1) 
where by end of the series we would try to envision one end to end flow of  **Infrastructure As a Code** in true sense. 
Effort here is to build a **containerised ecosystem** which could host any microservices using **CICD**. In the previous 
[blog](https://blog.avmconsulting.net/posts/2019-04-07-setup-kubernetes-cluster-with-terraform-and-kops-part-1)  we built
`Kubernetes` Cluster using `Terraform`, in this blog we will see how an enterprise Docker images are built using `Packer`.

This Blog is not for the audience who wants to get the complete overview of packer rather the intent is to seed a thought
which will make reader think in right direction for building the Docker images which are of enterprise class.

## Short Introduction
If you are like me who builds his own Docker images in **CI Pipeline** because of various reasons like company security policy,
custom OS packages etc you would benefit and also for others you would know why Dockerfile is fragile, makes bloated images and 
looks very silly (this is strictly my personal opinion).

#### Packer
Pre-baked machine images have a lot of advantages, but most have been unable to benefit from them because images have 
been too tedious to create and manage. There were either no existing tools to automate the creation of machine images 
or they had too high of a learning curve. The result is that, prior to Packer, creating machine images, Docker images 
threatened the agility of operations teams, and therefore aren't used, despite the massive benefits.

Packer changes all of this. Packer is easy to use and automates the creation of any type of image. It embraces 
modern configuration management by encouraging you to use a framework such as **ansible** to install and configure 
the software within your Packer-made images.

In other words: Packer brings pre-baked images into the modern age, unlocking untapped potential and opening new opportunities.

Ref: https://www.packer.io/intro/why.html

#### Ansible
Ansible is a universal language, unraveling the mystery of how work gets done. Turn tough tasks into repeatable playbooks. 
Roll out enterprise-wide protocols with the push of a button. Give your team the tools to automate, solve, and share.

{{< box type="info" title="Note" >}}  Ref: https://www.ansible.com/overview/how-ansible-works {{< /box >}}

## Problem we are trying to solve
Use Packer and Ansible together without using Dockerfile to build images so that we really overcome the challenges of 
Using the Dockerfile. Make use of one tool (Packer) to build any images like Virtual Images (For Developers), AMI Images, 
Docker Images and many more images if needed all using the combination of Anible and Packer

## Why not use Dockerfile rather than Packer
* Two key technologies behind Docker image and container management are: 
  * [stackable image layers and copy-on-write (CoW)](https://docs.docker.com/storage/storagedriver/) 
* These layers (also called intermediate images) are generated when the commands in the Dockerfile are executed during the
Docker image build.
* Because of above your images size grows and at time you could also have image > 2GB for a python app, and 90% of your 
layers are not reused. So, actually, you don’t need all these layers.
* Also in any practical scenario an Organisation already has a configuration management tools like Ansible and there are 
existing Playbooks to build environment we have to just reuse them.
* Since Packer has both `Ansible provisioner` and `Docker builders` it gives a superb flexibility to use same playbook 
to build the Docker images which was earlier used for building images like AMI.
* If you are using Dockerfile you should use something like [docker-squash](http://jasonwilder.com/blog/2014/08/19/squashing-docker-images/)
for minimising the image size. which will be avoided if we use the approach we are following in this blog
* Lets see one example of Dockerfile [NGINX](https://github.com/nginxinc/docker-nginx/blob/e5123eea0d29c8d13df17d782f15679458ff899e/mainline/stretch/Dockerfile). 
I am giving only few lines in the following example. but one can see many use of `&&` and
`/` to reduce the size and Dockerfile looks giant chain of commands with newline escaping, inplace config patching 
with sed and cleanup as the last command.  

```
# NOTE: This is Partial DOCKERFILE

RUN set -x \
	&& apt-get update \
	&& apt-get install --no-install-recommends --no-install-suggests -y gnupg1 apt-transport-https ca-certificates \
	&& \
	NGINX_GPGKEY=573BFD6B3D8FBC641079A6ABABF5BD827BD9BF62; \
	found=''; \
	for server in \
		ha.pool.sks-keyservers.net \
		hkp://keyserver.ubuntu.com:80 \
		hkp://p80.pool.sks-keyservers.net:80 \
		pgp.mit.edu \
	; do \
		echo "Fetching GPG key $NGINX_GPGKEY from $server"; \
		apt-key adv --keyserver "$server" --keyserver-options timeout=10 --recv-keys "$NGINX_GPGKEY" && found=yes && break; \
	done; \
	test -z "$found" && echo >&2 "error: failed to fetch GPG key $NGINX_GPGKEY" && exit 1; \
	apt-get remove --purge --auto-remove -y gnupg1 && rm -rf /var/lib/apt/lists/* \
	&& dpkgArch="$(dpkg --print-architecture)" \
	&& nginxPackages=" \
		nginx=${NGINX_VERSION} \
		nginx-module-xslt=${NGINX_VERSION} \
		nginx-module-geoip=${NGINX_VERSION} \
		nginx-module-image-filter=${NGINX_VERSION} \
		nginx-module-njs=${NJS_VERSION} \
		
```

{{< box type="info" title="Note" >}} Ref: https://docs.docker.com/engine/docker-overview/ {{< /box >}}

## Environment Used for this Exercise
* Cloud: `AWS`
* Region: `us-west-2`
* Instance Type: `t2.medium`
* OS: Ubuntu `18.04`
* AMI : `ami-005bdb005fb00e791`
{{< box type="warning" title="Warning" >}} Please check the details before using scripts to launch it will incur some cost in the AWS {{< /box >}}

## Actual Implementation
We will see how the challenges of Dockerfile is mitigated using the following example. we will use a ansible playbook to setup 
required directories and also do clean up activities in same playbook. to begin with we will install both packer and ansible, then we will
create a packer file and ansible playbook
#### STEP 1: Install Packer and Ansible
* clone the repo `git clone https://github.com/dbiswas1/packer`
* cd packer
* chmod 755 setup_build_env.sh
* ./setup_build_env.sh

```
###############################################Partial Output###############################################################

ubuntu@ip-172-31-44-201:~/packer$ ./setup_build_env.sh

Hit:1 http://us-west-2.ec2.archive.ubuntu.com/ubuntu bionic InRelease
Hit:2 http://us-west-2.ec2.archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:3 http://us-west-2.ec2.archive.ubuntu.com/ubuntu bionic-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu bionic-security InRelease
Hit:5 http://ppa.launchpad.net/ansible/ansible/ubuntu bionic InRelease
Reading package lists... Done
Hit:1 http://us-west-2.ec2.archive.ubuntu.com/ubuntu bionic InRelease
Hit:2 http://us-west-2.ec2.archive.ubuntu.com/ubuntu bionic-updates InRelease
Hit:3 http://us-west-2.ec2.archive.ubuntu.com/ubuntu bionic-backports InRelease
Hit:4 http://security.ubuntu.com/ubuntu bionic-security InRelease
Hit:5 http://ppa.launchpad.net/ansible/ansible/ubuntu bionic InRelease .....

======================================START PACKER SETUP ================================================
[2019-04-28 08:12:44]: INFO: Start Packer Download -> Version : 1.4.0 and Flavor: linux_amd64
[2019-04-28 08:12:45]: INFO: Download Complete
[2019-04-28 08:12:45]: INFO: Unzip the Packer binary
Archive:  packer_1.4.0_linux_amd64.zip
  inflating: packer
[2019-04-28 08:12:46]: INFO: Unzip Complete
[2019-04-28 08:12:46]: INFO: Packer setup done -> Version : 1.4.0 and Flavor: linux_amd64
========================================END PACKER SETUP================================================
==================================Verify START====================================================
[2019-04-28 08:12:48]: INFO: VERIFY PACKER
1.4.0
[2019-04-28 08:12:49]: INFO: VERIFY ANSIBLE
ansible 2.7.10
  config file = /etc/ansible/ansible.cfg
  configured module search path = [u'/home/ubuntu/.ansible/plugins/modules', u'/usr/share/ansible/plugins/modules']
  ansible python module location = /usr/lib/python2.7/dist-packages/ansible
  executable location = /usr/bin/ansible
  python version = 2.7.15rc1 (default, Nov 12 2018, 14:31:15) [GCC 7.3.0]
[2019-04-28 08:12:49]: INFO: Validation Successful !!!
==================================Verify END======================================================

###############################################Partial Output###############################################################

```
#### STEP 2: Build a sample image using Ansible and Packer
{{< box type="info" title="Note" >}} Prerequisite: Lets Make sure Docker is installed {{< /box >}}
* Lets Create a packer json file `redis.json`
* Lets Create a Role in ansible which is generic to install `redis`
* Lets Create Ansible Playbook to use the role and customise for our Docker image `redis.yml`
* Please check the OFFICIAL [REDIS DOCKRFILE](https://github.com/dockerfile/redis/blob/master/Dockerfile) and compare 
the Example Playbook you will get better understanding.
* In the following output you would see how packer launches a customised container which can use existing ansible playbook
 in the organisation to build the Docker images which complies your organisation standards.
* Also by using ansible you get flexibility of having same playbook to build your Docker image on any Base Operating system 
without much changes.
* Run `cd packer/packer-files && packer build redis.json`
  
```
############################################### Output###############################################################
 ubuntu@ip-172-31-44-201:~/sample/packer/packer-files$ packer build redis.json
 docker output will be in this color.
 
 ==> docker: Creating a temporary directory for sharing data...
 ==> docker: Pulling Docker image: debian:jessie-slim
     docker: jessie-slim: Pulling from library/debian
     docker: Digest: sha256:d5248fdfe8cd99173cfb2c279e4f826ef29c839d37311de0065ac003486efb0d
     docker: Status: Image is up to date for debian:jessie-slim
 ==> docker: Starting docker container...
     docker: Run command: docker run -v /home/ubuntu/.packer.d/tmp:/packer-files -d -i -t --entrypoint=/bin/sh -- debian:jessie-slim
     docker: Container ID: 2a237cc066b161f2f0794fa2a35d6a343bfc94cdc2454d14c0af07975b50c6d7
 ==> docker: Using docker communicator to connect: 172.17.0.2
 ==> docker: Provisioning with Ansible...
 ==> docker: Executing Ansible: ansible-playbook --extra-vars packer_build_name=docker packer_builder_type=docker -o IdentitiesOnly=yes -i /tmp/packer-provisioner-ansible496461571 /home/ubuntu/packer/sample/redis2/packer/packer-files/redis.yml -e ansible_ssh_private_key_file=/tmp/ansible-key128705332
     docker:
     docker: PLAY [Setup Python] ************************************************************
     docker:
     docker: TASK [Boostrap python] *********************************************************
     docker: changed: [default]
     docker:
     docker: PLAY [Setup Redis] *************************************************************
     docker:
     docker: TASK [Gathering Facts] *********************************************************
     docker: ok: [default]
     docker:
     docker: TASK [../ansible-redis : Create group] *****************************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Create user] ******************************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Install dependency packages] **************************
     docker: changed: [default] => (item=ca-certificates)
     docker: changed: [default] => (item=wget)
     docker: changed: [default] => (item=gcc)
     docker: ok: [default] => (item=libc6-dev)
     docker: changed: [default] => (item=make)
     docker:
     docker: TASK [../ansible-redis : Create Required Redis Build Directories] **************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Create Data Home] *************************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Create Required Redis Home Directories] ***************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Download and verify Redis] ****************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Unzip Redis] ******************************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Disable Redis protected mode in server.h] *************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Uncomment Bind in the conf] ***************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Uncomment Daemonize in the conf] **********************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Copy the Redis conf file] *****************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Compile Redis sources] ********************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Copy the redis-senitel file] **************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Remove build deps] ************************************
     docker: changed: [default]
     docker:
     docker: TASK [../ansible-redis : Remove sources] ***************************************
     docker: changed: [default]
     docker:
     docker: TASK [Put runtime programs] ****************************************************
     docker: changed: [default] => (item=run.sh)
     docker:
     docker: PLAY [Squash the Container (Cleanup)] ******************************************
     docker:
     docker: TASK [Remove python] ***********************************************************
     docker: changed: [default]
     docker:
     docker: TASK [Remove apt lists] ********************************************************
     docker: changed: [default]
     docker:
     docker: PLAY RECAP *********************************************************************
     docker: default                    : ok=21   changed=20   unreachable=0    failed=0
     docker:
 ==> docker: Committing the container
     docker: Image ID: sha256:c98b7fa83acddafadcdb11b4ebf8c534c2f54a759549f3ad2259c64d259c3d5b
 ==> docker: Killing the container: 2a237cc066b161f2f0794fa2a35d6a343bfc94cdc2454d14c0af07975b50c6d7
 ==> docker: Running post-processor: docker-tag
     docker (docker-tag): Tagging image: sha256:c98b7fa83acddafadcdb11b4ebf8c534c2f54a759549f3ad2259c64d259c3d5b
     docker (docker-tag): Repository: avm-blog/redis:latest
 Build 'docker' finished.
 
 ==> Builds finished. The artifacts of successful builds are:
 --> docker: Imported Docker image: sha256:c98b7fa83acddafadcdb11b4ebf8c534c2f54a759549f3ad2259c64d259c3d5b
 --> docker: Imported Docker image: avm-blog/redis:latest
###############################################Output###############################################################

```
#### STEP 3: Verify the Exercise 
* Launch the container you used and test if it is launching using `docker run`
* You can also use push directive in Packer to push the image to your favourite registry
* check `docker history` also which gives idea that we have reduced the layer
* List the images with `docker images`
* finally cleanup `docker rm -f $(docker ps -qa) && docker rmi avm-blog/redis`
* See output below

```
###############################################Output###############################################################
ubuntu@ip-172-31-44-201:~/sample/packer/packer-files$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
avm-blog/redis      latest              c98b7fa83acd        21 minutes ago      125MB
debian              jessie-slim         c069cbf23371        4 weeks ago         81.4MB


ubuntu@ip-172-31-44-201:~/sample/packer/packer-files$ docker history c98b7fa83acd
IMAGE               CREATED             CREATED BY                                      SIZE                COMMENT
c98b7fa83acd        21 minutes ago                                                      43.4MB
c069cbf23371        4 weeks ago         /bin/sh -c #(nop)  CMD ["bash"]                 0B
<missing>           4 weeks ago         /bin/sh -c #(nop) ADD file:7e1c64289e566a098…   81.4MB


ubuntu@ip-172-31-44-201:~/sample/packer/packer-files$ docker run -it --rm avm-blog/redis
1:C 29 Apr 2019 17:01:20.509 # oO0OoO0OoO0Oo Redis is starting oO0OoO0OoO0Oo
1:C 29 Apr 2019 17:01:20.509 # Redis version=5.0.4, bits=64, commit=00000000, modified=0, pid=1, just started
1:C 29 Apr 2019 17:01:20.509 # Configuration loaded
                _._
           _.-``__ ''-._
      _.-``    `.  `_.  ''-._           Redis 5.0.4 (00000000/0) 64 bit
  .-`` .-```.  ```\/    _.,_ ''-._
 (    '      ,       .-`  | `,    )     Running in standalone mode
 |`-._`-...-` __...-.``-._|'` _.-'|     Port: 6379
 |    `-._   `._    /     _.-'    |     PID: 1
  `-._    `-._  `-./  _.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |           http://redis.io
  `-._    `-._`-.__.-'_.-'    _.-'
 |`-._`-._    `-.__.-'    _.-'_.-'|
 |    `-._`-._        _.-'_.-'    |
  `-._    `-._`-.__.-'_.-'    _.-'
      `-._    `-.__.-'    _.-'
          `-._        _.-'
              `-.__.-'

1:M 29 Apr 2019 17:01:20.510 # WARNING: The TCP backlog setting of 511 cannot be enforced because /proc/sys/net/core/somaxconn is set to the lower value of 128.
1:M 29 Apr 2019 17:01:20.510 # Server initialized
1:M 29 Apr 2019 17:01:20.510 # WARNING overcommit_memory is set to 0! Background save may fail under low memory condition. To fix this issue add 'vm.overcommit_memory = 1' to /etc/sysctl.conf and then reboot or run the command 'sysctl vm.overcommit_memory=1' for this to take effect.
1:M 29 Apr 2019 17:01:20.511 # WARNING you have Transparent Huge Pages (THP) support enabled in your kernel. This will create latency and memory usage issues with Redis. To fix this issue run the command 'echo never > /sys/kernel/mm/transparent_hugepage/enabled' as root, and add it to your /etc/rc.local in order to retain the setting after a reboot. Redis must be restarted after THP is disabled.
1:M 29 Apr 2019 17:01:20.511 * Ready to accept connections
###############################################Output###############################################################

```

{{< box type="info" title="References" >}} 
* https://docs.ansible.com/ansible-container/getting_started.html
* https://docs.docker.com/develop/develop-images/dockerfile_best-practices 
* https://www.packer.io/docs/builders/docker.html 
{{< /box >}}
