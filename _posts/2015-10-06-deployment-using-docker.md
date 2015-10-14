---
title: Deployment using Docker
author: douglaseggleton
---
This post covers how we use Docker to build and ship applications from development to a production environment and how it can be used in the various stages of the pipeline. We used the strategy outlined in this post to deploy a Node.js application serving a RESTful API, but is equally applicable to any web-based app.

## Getting started with Docker
[Docker](https://www.docker.com/) provides a way to run applications as a process within a containerised environment, meaning everything required for the application to run is inside the container. Therefore, we only require docker on the host machine to run the application.

Containers are started from a [Docker Image](https://docs.docker.com/userguide/dockerimages/). The instructions to create the image are kept in a `Dockerfile` and the image can be built using the `docker build` command.

```
FROM node:wheezy
COPY . /node-app
RUN cd /node-app; npm install
WORKDIR /node-app
CMD ["node", "app.js"]
EXPOSE 8888
```

This image is based on a `node:wheezy` base image, which will be imported from [Docker Hub](http://dockerhub.com/). The contents of this project are then copied to the image and we specify the `node app.js` command to be run when starting the container. Port 8888 is then exposed from the container to the host machine.

## Setting up the environments
A pipeline to production typically involves pushing code through multiple environments such as: work in progress, staging and pre-production. This can involve a number of machines that all require Docker to be installed. Provisioning the machines to have Docker installed can be done using a configuration management tool, e.g. Chef. We also provision the machines with a 'deploy' user with access to Docker and an authorised SSH key to allow our Jenkins machine to orchestrate the deployment to the environments.

## Deployment pipeline
Changes in our codebase are monitored with Jenkins and this triggers our pipeline, which consists of the following steps:

1. **Build/Test/Package** - The project is built (using a [Makefile](https://www.gnu.org/software/make/)) and the `docker build` command is executed to build the image. A container based on the image is then created and the unit/acceptance tests are run within it. After the tests have passed, the image is passed to the next stage of the pipeline.
2. **Push to Docker Hub** - The app image is pushed to Docker Hub and tagged with the Jenkins build number. The build number is then passed as a parameter to the next job.
3. **Deploy to Environment (WIP/Staging/Production)** - The deployment tool is then used to perform the following actions on the destination machines:
 1. Pull the required image version from Docker Hub.
 2. Remove any existing containers.
 3. Create new containers based on the image.

![Docker](/images/2015-10-06-deployment-using-docker/deployment-using-docker.png)

## Deployment Tool
The application needs to be deployed to several machines across multiple environments. This requires Docker commands to be run on each of the machines in an environment. We utilise [Fabric](http://www.fabfile.org/), an extremely powerful deployment framework, to perform the tasks needed to securely deploy our application  over SSH. Our tool consists of a library of Docker 'tasks' and a higher level 'deploy' task in the `fabfile.py`. The specified version can be deployed to all machines in an environment with a single command: `fab environment:staging deploy:1.2.3`

{% highlight python %}
# deploy_tool/tasks/docker.py
from fabric.api import run, settings, task, env

def docker(command):
    '''Execute a docker command'''
    return run('%s %s' % (env.params.docker, command))

@task
def list_containers():
    docker('ps -a')

@task
def login(username, password, email):
    docker('login -u %s -p %s -e %s' % (username, password, email))

@task
def run_container(project, tag, name, options):
    '''Run a detached docker container'''
    docker('run --detach %s --name=%s %s:%s' % (options, name, project, tag))
{% endhighlight %}

{% highlight python %}
# deploy_tool/fabfile.py
from tasks import docker

@task
def deploy(tag):
  # Authenticate with DockerHub
  execute(docker.login, env.docker_user, env.docker_pass, env.docker_email)

  # Remove old images
  execute(docker.remove_old_images, env.params.project)

  # Pull the tagged version from Docker Hub
  execute(docker.pull, env.params.project, tag)

  # Remove existing container
  execute(docker.remove_container, env.params.container_name)

  # Create a new container from the image
  execute(docker.run_container, env.params.project, tag, env.params.container_name,
    '--publish 80:8888 --env NODE_ENV="%s" --log-driver=syslog --restart=always')
{% endhighlight %}

The deploy tool also has its own environment dependencies, so this can be built into a Docker image and run as a containerised process. The image for the deploy tool can be based on a Python image, with the addition of the deploy tool's environment dependencies specified in a [`requirements.txt`](https://pip.readthedocs.org/en/1.1/requirements.html). The image below specifies an entry point to a shell script that provides the interface to our deploy tool.

```
FROM python:2
WORKDIR /deploy_tool
COPY ./requirements.txt /deploy_tool/requirements.txt
RUN pip install --no-cache-dir -r requirements.txt
COPY . /deploy_tool
ENTRYPOINT ["./deploy.sh"]
```
We copy the `requirements.txt` file and run `pip install` before copying the deploy tool's code. This is to allow Docker to cache and reuse previously built image layers, thereby reducing the build time.

## Summary
Docker has been a great solution for the team here at Wonga and has simplified the way in which we deploy applications.

Using Docker in a deployment strategy gives the benefit of being able to configure an application's environment and package it together with the app when pushing it through a production pipeline. The same approach can also be used for packaging tools used throughout the pipeline. Some of the issues we have encountered using this approach:

* **Docker Hub Availibility** - The pipeline is dependent on Docker Hub, and whilst availability has not been a huge issue, we have experienced outages. This could be improved by either hosting our own internal Docker hub service or pushing a local artifact along the pipeline.
* **Zero Downtime Deployment** - The current solution removes existing containers before starting new ones. This could be improved by: starting the new containers first, switching the service over to the new containers and then wait for demand on the old containers to cease before removing them.

For more information on how Docker works, visit [Understanding Docker](https://docs.docker.com/introduction/understanding-docker/).
