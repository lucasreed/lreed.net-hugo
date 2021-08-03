---
title: "Ansible AWX on Amazon Fargate - Part 1"
date: 2017-12-14T18:11:54-05:00
draft: false
tags: ["AWS", "Docker", "Ansible"]
---

AWS recently announced a lot of things at RE:Invent and one of the more exciting new toys is [Fargate](https://aws.amazon.com/blogs/aws/aws-fargate/)! Another thing I've been acquainting myself with lately is Ansible's [AWX](https://github.com/ansible/awx), the open source version of their Ansible frontend: [Tower](https://www.ansible.com/tower).

I'm writing this series to show how I got AWX up and running in Fargate. Since both of these things are fairly new, it took quite a bit of [documentation reading](http://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_GetStarted.html), but all in all I'm happy with the end result.

**DISCLAIMER**: There are some aspects of this guide that are not very secure. I will be addressing these in my own installation, but for now I will put a big bold **UNSAFE** tag next to anything that should later be fixed (I'm looking at you docker environment variables!).

We will be utilizing the following aspects of AWS:

* RDS - Backend PostgreSQL Database
* IAM - Allowing access to various things
* VPC - Where the containers get launched
* EC2 - For our security group(s)
* Route 53 - Setting a few DNS records
* CloudWatch - Container logging
* Elastic Container Service - All things containers, including the new Fargate launch type

There are also some assumptions I'll be making since I don't want the scope of the series to get too crazy:

* Already have a VPC up and running with at least one subnet configured
* Comfortable running commands in the terminal
* Workstation that has [docker installed](https://www.docker.com/community-edition)
* [awscli](http://docs.aws.amazon.com/cli/latest/userguide/installing.html) installed and configured for your AWS account
* General understanding of Docker and the AWS services mentioned above

# Setup
I decided that I wanted to keep my set of containers all together in a container registry I control. Since the Fargate launch type only supports images in Amazon ECR or public repositories in Docker Hub, we'll be creating repositories for each container within AWS.
### Creating a container repository
In your AWS account, navigate to the Elastic Container Service. If you have never used ECS before, you will have a welcome screen with a couple links. Click the **Get Started** button and then cancel the introductory app creation. Once you're at the main ECS screen click on Repositories link on the left menu and then select **Get Started**.

You should see something like this:
{{< img src="config_repo.png" alt="repo configuration" >}}

We will be creating one repository for each of the following containers, I name mine slightly different than the ones I pull from docker hub:

```bash
my-awx-task
my-awx-web
my-memcached
my-rabbitmq
```

Once you've created one repository for each of the above containers take note of each Repository URI. Now lets go to the terminal!

### Pull containers, tag, then push
First we'll pull in all the necessary containers. I specify a version (rather than *latest*) for the awx containers so that I can plan upgrades later:
```bash
docker pull ansible/awx_task:1.0.1.316
docker pull ansible/awx_web:1.0.1.316
docker pull memcached:alpine
docker pull rabbitmq:3
```
You should see something similar in your docker image list:
```bash
# docker image ls
REPOSITORY          TAG                 IMAGE ID            CREATED             SIZE
rabbitmq            3                   5dbe0bc7d118        25 hours ago        127MB
ansible/awx_task    1.0.1.316           30fe21b65ea6        2 days ago          1.06GB
ansible/awx_web     1.0.1.316           15a91a74672f        2 days ago          1.03GB
memcached           alpine              4ad3382204db        13 days ago         7.02MB
```
Then we will generate the docker login command with:
```bash
aws ecr get-login --no-include-email --region us-east-1
```
Run the long 'docker login' command it outputs.

Now we want to start tagging the containers we pulled down and then pushing them up to our new repositories. I'm going to make two tags for each container, replacing AWS_ACCOUNT with your aws account number:

*AWX Task*
```bash
docker tag ansible/awx_task:1.0.1.316 AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-task:1.0.1.316
docker tag ansible/awx_task:1.0.1.316 AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-task:latest
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-task:1.0.1.316
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-task:latest
```

*AWX Web*
```bash
docker tag ansible/awx-web:1.0.1.31 AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-web:1.0.1.316
docker tag ansible/awx-web:1.0.1.31 AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-web:latest
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-web:1.0.1.316
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-web:latest
```

*RabbitMQ*
```bash
docker tag rabbitmq:3 AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq:3
docker tag rabbitmq:3 AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq:latest
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq:3
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq:latest
```

*Memcached*
```bash
docker tag memcached:alpine AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-memcached:alpine
docker tag memcached:alpine AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-memcached:latest
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-memcached:alpine
docker push AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-memcached:latest
```

You should now have your own repositories set up for each container. Example of memcached repo:

{{< img src="repo_with_tags.png" alt="memcached repository with two tags" >}}

Below is what your docker images should look like on your local machine (substituting your *AWS_ACCOUNT* of course)

```bash
# docker image ls |grep my-
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq    3                   5dbe0bc7d118        25 hours ago        127MB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq    latest              5dbe0bc7d118        25 hours ago        127MB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-task    1.0.1.316           30fe21b65ea6        2 days ago          1.06GB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-task    latest              30fe21b65ea6        2 days ago          1.06GB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-web     1.0.1.316           15a91a74672f        2 days ago          1.03GB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-awx-web     latest              15a91a74672f        2 days ago          1.03GB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-memcached   alpine              4ad3382204db        13 days ago         7.02MB
AWS_ACCOUNT.dkr.ecr.us-east-1.amazonaws.com/my-memcached   latest              4ad3382204db        13 days ago         7.02MB
```

# Next Time
In Part 2 I'll go over setting up your PostgreSQL Database with Amazon RDS, as well as creating our ECS Task. We'll also make sure IAM will allow you pass along the logs to CloudWatch!
