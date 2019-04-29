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
* Finally run `cd packer-files && packer build redis.json`
* Please check the OFFICIAL [DOCKRFILE]() and compare the Example you will get better understanding
```
###############################################Partial Output###############################################################
 
###############################################Partial Output###############################################################

```
#### STEP 3: Verify the Exercise 

{{< box type="info" title="References" >}} 
* https://docs.ansible.com/ansible-container/getting_started.html
* https://docs.docker.com/develop/develop-images/dockerfile_best-practices 
* https://www.packer.io/docs/builders/docker.html 
{{< /box >}}
