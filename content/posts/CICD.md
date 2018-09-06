---
title: "Continous Integration/Continuous Deployment with Node, Docker, AWS, and Gitlab"
date: 2018-09-03T20:18:53-05:00
showDate: true
draft: false
---

## Introduction

This article will teach you the required knowledge in order to set up <a href="https://en.wikipedia.org/wiki/Continuous_integration" target="_blank">continuous integration</a>/<a href="https://en.wikipedia.org/wiki/Continuous_delivery#Relationship_to_continuous_deployment" target="_blank">continuous delivery</a> using <a href="https://about.gitlab.com/features/gitlab-ci-cd/" target="_blank">Gitlab's CI/CD</a>, AWS (<a href="https://aws.amazon.com/elasticbeanstalk/" target="_blank">elastic beanstalk</a>, <a href="https://aws.amazon.com/blogs/aws/cloud-container-management/" target="_blank">amazon container service</a>), <a href="https://opensource.com/resources/what-docker" target="_blank">Docker</a> and <a href=https://nodejs.org/en/about/" target="_blank">Node</a>. These technologies in combination with <a href="https://nvie.com/posts/a-successful-git-branching-model/" target="_blank">"a successful git branching model"</a> will give your organization a well defined process for code review, testing, and deploying your applications.  

If you do not have a basic understanding of the topics presented above, take the time to read the above links.

## High Level Overview

The goal that we are aiming for is that a push to the master branch of your project repository will trigger Gitlab's CI/CD to build a image, run tests, upload the image to amazon's container service, and then trigger elastic beanstalk to update to the image. 

In order to get all of this set up will we need to complete the following steps below:

1. Create a Node application.
2. Containerize your Node application (specify a Dockerfile).
3. Create an Amazon container service repository.
4. Define your AWS dockerrun file (Tells Amazon's Elastic Beanstalk where to look for container images).
5. Create an Elastic Beanstalk project.
6. Define your gitlab-ci.yml file (define scripts to run upon a push to a specific branch such as master).
7. Create a Gitlab project and push to the master branch.

## Step 1 - Create a Node Application

Our application that we want to containerize will be an <a href="https://expressjs.com" target="_blank">expressjs</a> Node server. Ensure that you have <a href="https://nodejs.org/en/" target="_blank">Node</a> and <a href="https://www.npmjs.com/get-npm" target="_blank"> npm </a> installed (Depending on how you install Node, npm is usually included). 

Create a new npm project by:

  1. Creating a new project directory `mkdir myproject`
  2. Initalizing a Node project `npm init` 
  3. In your project directory create a filed named `server.js`
  4. Install express `npm install express --save`
  
The content of `server.js`:

```
const express = require('express');
const app = express();

app.get('/', function (request, response) {
  response.send('Hello!');
});

app.listen(3000, function () {
  console.log('Listening on port 3000');
});
```
You can confirm that the application works as expected by starting the application with `node server.js` and navigating to http://localhost:3000/.

## Step 2 - Containerize your Node Application

Containerizing your Node application will allow you to easily deploy your application to the cloud providers (AWS, Azure, GCP). You can read about some of the pros/cons of docker containers <a href="https://blog.philipphauer.de/discussing-docker-pros-and-cons/" target="_blank">here</a>. 

To containerize your application ensure than you have docker installed (download <a href="https://www.docker.com/get-started" target="_blank">here</a>), and then create a new file named `Dockerfile`. The contents of the file will look like: 

```
FROM node:carbon

#Create application directory
WORKDIR /usr/src/app

#Install application dependencies
#A wildcard is used to ensure both package.json + package-lock.json are copied
COPY package*.json ./

#For dev: RUN npm install
#For prod: RUN npm install --only=production
RUN npm install

#bundle application source
COPY . .

EXPOSE 3000

CMD ["node", "start"]
```
Now that we have our Dockerfile we want to build and run an image. The following steps will explain how to build, view, and run a docker image. 

1. Build the image with the command `docker build -t <tag_name> .` (This will build and tag your image with the name `tag_name`).  
2. View the images on your machine with the command `docker images` (Take note of the image_name, we will need it in the following step).
3. Run your image with the command `docker run -d <image_name>:<tag_name> -p 3000:3000`. This command will run the image in detached mode with port 3000 of your local machine mapped to port 3000 of the container.

To confirm that everything is working as expected navigate to http://localhost:3000/. 

Documentation on all docker CLI commands can be found <a href="https://docs.docker.com/engine/reference/commandline/cli/" target="_blank">here</a>. 

## Step 3 - Create an Amazon Container Service Repository

You will need to create an Amazon container service repository. You can do this through the AWS UI, or with this command: `aws ecr create-repository --repository-name example-project-name` (You will need to install and login to the AWS cli to use this command). Once you create the repository take note of the uri. It will look something like `9710533123123.dkr.ecr.us-west-2.amazonaws.com/example-project-name`. The uri will be used in the following steps. 

## Step 4 - Define your AWS Dockerrun File

Now we will create an AWS dockerrun file. Name the file `<myproject>-dockerrun.aws.json`. The docker run file is a configuration file for Elastic Beanstalk. It will tell Elastic Beanstalk where to look for container images.

Below is a simple dockerrun file:

```
{
    "AWSEBDockerrunVersion": "1",
    "Image": {
      "Name": "9710533123123.dkr.ecr.us-west-2.amazonaws.com/example-project-name",
      "Update": "true"
    },
    "Ports": [
      {
        "ContainerPort": "3000"
      }
    ]
}
```

You will need to upload this file when creating your Elastic Beanstalk environment. 

Note that the `ContainerPort` field matches the port specified in our Dockerfile and that the `Name` field matches the uri from the previous step.

## Step 5 - Create an Elastic Beanstalk Project

You will need to create an Elastic Beanstalk project and environment (You can have many environments under a single project). You will need to upload your AWS dockerrun file when creating the environment. This can be accomplished through the AWS UI. When creating an Elastic Beanstalk environment select Docker under the `Preconfigured Platform` option and upload your AWS dockerrun file under the `Application code` option. Be sure to take note of the environment name as it will be needed during the `deploy` stage of our CI/CD pipeline.

## Step 6 - Define your gitlab-ci.yml File

Next you will need to create a file named `.gitlab-ci.yml`. The file needs to be named exactly as specifed or else Gitlab will not recognize it. This is the file where we will specify the scripts that will build the container, push to Amazon's container service, and trigger elastic beanstalk to update.

Below is a simple `.gitlab-ci.yml` file that we will use for our Node application: 

```
#https://docs.gitlab.com/ee/ci/docker/using_docker_build.html for more information on how gitlab ci/cd works. 

image: docker:stable
# When using dind, it's wise to use the overlayfs driver for
# improved performance.
variables:
  DOCKER_DRIVER: overlay2
  STAGING_CONTAINER_IMAGE_URL: 9710533123123.dkr.ecr.us-west-2.amazonaws.com/example-project-name

services:
- docker:dind

before_script:
- docker info
- apk add --no-cache curl jq python py-pip # install aws cli
- pip install awscli 
- $(aws ecr get-login --no-include-email --region us-west-2) # log in to aws

build:
  stage: build
  script:
    - docker pull $STAGING_CONTAINER_IMAGE_URL:latest || true # pull down the latest image if it exists, this will speed up build times.
    - docker build -f Dockerfile --cache-from $STAGING_CONTAINER_IMAGE_URL:latest -t $STAGING_CONTAINER_IMAGE_URL:$CI_COMMIT_SHA -t $STAGING_CONTAINER_IMAGE_URL:latest .
    - docker push $STAGING_CONTAINER_IMAGE_URL:$CI_COMMIT_SHA
    - docker push $STAGING_CONTAINER_IMAGE_URL:latest
  only:
    - master

test:
  stage: test
  script:
    - docker pull $STAGING_CONTAINER_IMAGE_URL:$CI_COMMIT_SHA
    - docker run $STAGING_CONTAINER_IMAGE_URL:$CI_COMMIT_SHA npm test

deploy:
  stage: deploy
  script: 
    - aws elasticbeanstalk create-application-version --application-name venture1-backend --version-label latest --source-bundle S3Bucket=$S3_BUCKET,S3Key=$AWS_DOCKER_RUN_JSON --region us-west-2
    - aws elasticbeanstalk update-environment --environment-name ExampleProject-env-1 --version-label latest --region us-west-2
  only:
    - master
```
The above yaml contains the stages that will be executed during a push to the master branch. This is the logic that connects Gitlab's CI/CD to Amazon's Elastic Beanstalk. With the above yaml each time you push to your project's master branch Gitlab will schedule a job onto it's runners and execute the defined stages.

Starting from the top of the yaml file you will notice `image: docker:stable`. What this does is tells Gitlab's CI/CD runners to execute the stages of our yaml in a container that has docker installed. This is necassary because we will be pushing containers to AWS in the build stage. You can read more about Docker in Docker (dind) <a href="https://docs.gitlab.com/ee/ci/docker/using_docker_build.html" target="_blank">here</a>. 

The second line of signifigance that you see is `variables`. You can define environment variables that you will need to access here. I've defined `DOCKER_DRIVER` and `STAGING_CONTAINER_IMAGE_URL`. Our gitlab-ci.yml file is checked into source control, if you have sensitive variables that you do not want to be tracked in version control Gitlab allows you to add environment varibles to specific jobs via the Gitlab UI. These variables will automatically be set when the job is executing. In addition to being able to add custom variables Gitlab will automatically set `CI_COMMIT_SHA` (which we will use to tag a version of our application; this is useful because it makes it easy for us to map a specific version of code to a container image). You can find all the variables that Gitlab automatically sets <a href="https://docs.gitlab.com/ee/ci/variables/#variables" target="_blank">here</a>.


Moving on to the four stages in our yaml; `before_script`, `build`, `test` and `deploy`. 

  1. The `before_script` stage is responsible for installing the AWS cli onto the runner (we need to do this each time because Gitlab will 'randomly' schedule your job onto one of their many machines and the enviornment that they schedule the task onto is ephemeral). The first line of this stage `docker info` prints information about the current installation of docker (I've found this useful for debugging purposes, it is not required). 
  2. The `build` stage is responsible for: building a container using the most recent code, tagging the new container with `CI_COMMIT_SHA` and `latest` (we will update to `latest`, but also keep a tag of `CI_COMMIT_SHA` so that we can revert to a given commit), and pushing the container to Amazon's container service.
  3. The `test` stage is responsible for executing any tests that you have (ie. unit tests). The entire job will fail if the tests fail.
  4. The `deploy` stage is responsible for telling Amazon's Elastic Beanstalk to create a new application version from our new container and updating to that version. 

## Step 7 - Create a Gitlab Project and push to the Master branch

The final step is to create a Gitlab project. You can do this through <a href="https://gitlab.com/users/sign_up" target="_blank"> Gitlab's UI</a>. Once you've created the project and configured git you will be able to push to the master branch and watch the pipeline execute. 

## Conclusion

You now have the knowledge that is needed in order to set up CI/CD for a simple Node project. The principles covered in this tutorial extend to other languages.

I would like to highlight that Elastic Beanstalk has an autoscaling service, which means that our setup will work nicely as the usage of the application increases (ie. You can configure Elastic Beanstalk to autoscale on CPU usage). Additionally, Elastic Beanstalk gives you the ability to access the underlying AWS resources should you need more control.

The above is an important point because I've worked with more complicated infrastructure (ie. <a href="https://kubernetes.io/" target="_blank">Kubernetes</a>) and what I've noticed is that <i> for most </i> software projects Elastic Beanstalk (or other similar products) provide sufficent features (ie. Management of: autoscaling, software deployments, operating system updates, networks, firewalls, and load balancing). 

For many, it is not worth the cost of implementing more complex systems. You want to avoid premature optimization. 

Ultimately, it is up to the organization to clearly/accurately identify what is needed (Doing this incorrectly is expensive); the infrastructure that an organization chooses should be specific to their circumstances. There is no silver bullet.  

By containerizing your applications early it will give you flexiblity to transition your infrastructure and/or switch cloud providers should you need.

If you have any questions/suggestions to improve this article contact me at jamesleahy@uvic.ca

