# gitlab-ci-utils

Scripts that assist in building/running GitLab CI pipelines containing docker images, using `ansible` and `docker deploy stack`. There's also an accompanying ansible [role for deploying to Docker Swarm](https://github.com/raarts/stack-deploy). 

## General idea 

The general idea is that in your `project` you supply the following files:

	ansible.cfg
	project.yml
	inventories
	stacks/project.yml
	.gitlab-ci.yml

Example contents of these files:

	# ----------------- ansible.cfg ------------------------
	[defaults]
	display_args_to_stdout=True
	inventory=inventories/$ENV
	nocows=1
	poll_interval=10
	remote_user=root
	host_key_checking=False
	private_key_file=$HOME/.ssh/id_ecdsa.com
	
	[ssh_connection]
	pipelining=True
		
Most important part here is the name of the private key file, and telling ansible where to find the inventory file (which tells it where to find the docker managers). `ENV` is the GitLab environment name.



More about this one later.

	# ------------------- Dockerfile -----------------------
	- name: "Deploy the project"
	  hosts: all
	  become: true
	  roles:
	    - { role: stack-deploy, docker_registry: "gitlab.yourdomain.com" }
Mostly relevant for the name of your git-server (where the docker container registry is hosted)

	# ------------------- stacks/project.yml ---------------
	version: "3.5"
	
	services:
	  project:
	    image: ${CI_REGISTRY}/${CI_PROJECT_NAMESPACE}/${CI_PROJECT_NAME}/project:${CI_COMMIT_REF_NAME}
	    environment:
	      - ENV=${ENV}

Very minimal stack deploy file for your project. Adapt as needed.

And finally the most iportant file: `.gitlab-ci.yml`:

	image: raarts/gitlab-ci-deploy:latest
	
	variables:
	  DOCKER_DRIVER: overlay
	  # Why is this variable set? We are setting DOCKER_HOST below. See: https://github.com/moby/moby/issues/34544
	  GODEBUG: netdns=cgo
	  COMMON: common
	  REVIEW: review
	
	before_script:
	  - export DOCKER_HOST=tcp://docker:2375
	  - type docker >/dev/null 2>&1 && docker --version && echo "$CI_JOB_TOKEN" | docker login -u gitlab-ci-token --password-stdin $CI_REGISTRY
	
	cache:
	  key: "$CI_COMMIT_REF_NAME"
	  untracked: true
	
	stages:
	- build
	- deploy
	
	# ------------------------------ Stage build -------------------------------
	
	build-product:
	  stage: build
	  script:
	  - gitlab-ci-build product
	
	# ------------------------------ Stage deploy -------------------------------
	
	p-product:
	  stage: deploy
	  variables:
	    ENV: production
	  environment:
	    name: production
	  script:
	  - gitlab-prepare-ssh-access id_ecdsa.com
	  - ansible-playbook ${CI_PROJECT_NAME}.yml -e "stack=product env=${ENV} token=$READ_REGISTRY_TOKEN"

## Usage

Assuming you already run a Docker Swarm, create a DNS entry which points to all ip addresses of the managers (using multiple A records).

Next clone this repo and run ./install. This will install the neccessary scripts into your $HOME/bin directory. This directory must be in your PATH.

Next in the GitLab project create the following files (using the templates above)

 	ansible.cfg
	project.yml
	inventories
	stacks/project.yml
	.gitlab-ci.yml
	
## Adding images with software to your project

The directory structure for each image is as follows:

	Dockerfile
	gitlab-ci/config
	gitlab-ci/build

The Dockerfile is something you build from this template:

	# ------------------- Dockerfile -----------------------
	ARG FROM=alpine:3.6
	FROM ${FROM}
	
	RUN <instructions to install software>

You generate the gitlab-ci subdirectory using a script:

$ gitlab-ci-init

And then you fill the config file as follows:

	me=image
	deps=
	from=alpine
	fromtag=3.6

- `me`: is the name of the image you are creating
- `deps`: are dependant images in this same project. If one of these changes, then this image will also be rebuilt
- `from`: the parent image from where this image is built
- `fromtag`: the tag. Only to be used for building from public images

Now you start working on the software, and the Dockerfile to build it.

## Supplied scripts:

### gitlab-ci-build

Used in the .gitlab-ci.yml file. Checks if software was changed in a containers directory (or in one of its dependencies, and rebuilds it

### gitlab-ci-deploy

Calls any deploy scripts in the gitlab-ci directory of a subdirectory, for when deploying outside of the swarm is needed.

### gitlab-ci-init

Initializes the `gitlab-ci` directory in a container, creates template files

### gitlab-ci-test

Used in the .gitlab-ci.yml file. Checks if software was changed in a containers directory (or in one of its dependencies, and runs the test script in the gitlab-ci directory

### gitlab-image-exists

Python script to check if an image already exists in the container registry

### gitlab-prepare-ssh-access

Used in the .gitlab-ci.yml file. Copies the SSH_PRIVATE_KEY (which should be defined as a GitLab project CI/CD variable into the build container, to allow ansible access to the managers.
