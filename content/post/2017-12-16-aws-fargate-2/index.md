---
title: "Ansible AWX on Amazon Fargate - Part 2"
date: 2017-12-16T16:00:10-05:00
draft: false
tags: ["AWS", "Docker", "Ansible" ]
---
In [part one](/post/2017-12-14-awx-fargate-1/) we got our container registry filled out with our docker repositories. In this part we will get our PostgreSQL database set up using [Amazon RDS](https://aws.amazon.com/rds/postgresql/) and bring up our containers with [ECS](https://aws.amazon.com/ecs/). Once again, take note that any part of this guide that is insecure to run in production I'll have a bold **UNSAFE** tag next to it. I believe the only places you'll find this though is in the container environment variables which will have some secrets in plain text. There is a method to import these safely [HERE](https://aws.amazon.com/blogs/security/how-to-manage-secrets-for-amazon-ec2-container-service-based-applications-by-using-amazon-s3-and-docker/), but is outside the scope of this guide.

# Database Setup
Navigate to the Relational Database Service on your AWS account and follow these steps:

* Select **Launch DB Instance**
* Choose **PostgreSQL** and click **Next**

{{< img src="fargate2_select_engine.png" sizes="55vw" alt="Select PostgreSQL Engine" >}}

* On the next screen, choose **Dev/Test** unless you are creating your production ready database. In that case, choose **Production**

{{< img src="fargate2_select_usecase.png" sizes="55vw" alt="Select Dev/Test Use Case" >}}

* A lot of the information in the **Specify DB Details** page will depend on how much data you will be keeping.
    * At the time of this writing, PostgreSQL 9.6 is required for AWX. I just chose the latest patch version at creation time, **9.6.3**.
    * I chose a **db.t2.micro** for my testing to keep costs down. This can just be adjusted later if necessary.
    * The storage type and size can be left default for now.
    * **DB instance identifier** can be whatever you please. Set **Master Username** to **awx**. **VERY IMPORTANT**: make sure you keep a record of what you set for your **Master Password** as we'll need it later.
    * Screenshot of example settings

        {{< img src="fargate2_db_settings1.png" sizes="50vw" alt="RDS Settings" >}}
        {{< img src="fargate2_db_settings2.png" sizes="50vw" alt="RDS Settings" >}}

* On the final page, **Configure advanced settings**, the options will be dependent on your configured VPC's.
    * Select the VPC you want the DB in (should be the same VPC you plan on launching AWX in)
    * I would keep **Public accessibility** set to No.
    * Create a new security group with whatever name you'd like. We will go back and add access for your ECS Task security group later.
    * **Database name** should be **awx**
    * **Databse port** should remain default (**5432**)
    * Everything else can be left default.

* Launch your instance and that should be it for your PostgreSQL DB!
* Once created, navigate to the **Instances** menu within RDS and click on your new DB instance. Take note of the **Endpoint** that is in the **Connect** section of its settings, you'll need that later for setting up the containers.

# Creating AWX ECS Task
Now we're actually going to get AWX up and running! Navigate over to the Elastic Container Service in your AWS account and lets get started. I will include less screenshots in this section, but will provide the full JSON of my setup at the end that can be used to re-create this from the command line.

### Create the ECS Cluster
* On the clusters menu, click on **Create Cluster** and then select the **Networking only** box and click **Next Step**

    {{< img src="fargate2_ecs_cluster1.png" sizes="40vw" alt="creating the fargate cluster" >}}

* On the next page, just give it a name and ignore the part about creating a new VPC. Click **Create**

### Create the AWX Task Definition
Back on the main page of the **Elastic Container Service** select the **Task Definitions** menu on the left and then click the **Create new Task Definition**

* Select the Fargate launch type compatibility
* On the **Configure task and container definitions** page we will be making all of our configurations for our containers
    * **Task Definition Name** can be whatever you like. And for now, leave the **Task Role** alone, we'll come back to that one.
    * For **Task execution role** use the one that it auto-creates, this is the role that allows the containers to pull in the docker images from the repositories we made in [Part One](/post/2017-12-14-awx-fargate-1/)
    * For **Task memory** I selected 4GB and have been running this for a few days with no issues. **Task CPU (vCPU)** I have set at 2.
    * Now for the main attraction: setting up the containers. You will probably want a separate browser tab open with your container repositories visible so you can easily grab their URI's. Click on **Add container** to get started.

#### rabbitmq container configuration
* Set the **Container name** to whatever you want, but it probably makes sense to keep it the same as your image repository name.
* In the **Image** box you should put the URI to your rabbitmq repository **PLUS** the tag you want to use. ex:```ACCOUNT_NUMBER.dkr.ecr.us-east-1.amazonaws.com/tivo-rabbitmq:latest```
* For **Memory Limits (MiB)** I set a soft limit of 300 MiB for rabbitmq. **Note**: The soft limit allocates that amount of memory to the container when it is brought up but it can still use more than the amount set. I chose these soft limits based on running local containers and keeping track of their memory usage. Your mileage may vary with my settings, adjust as necessary. I do not set any hard limits, but do leave a little headroom for shared memory across all containers.
* I do not reserve any CPU units for any of the containers.
* The **Essential** checkbox should be selected.
* The only other thing to update for the rabbitmq container is the **Env Variables** of which there is only one:
    * ```RABBITMQ_DEFAULT_VHOST: awx```
* I do not set up CloudWatch logs for the rabbitmq container because I don't feel they would bring a lot of value. If you'd like those logs delivered, go ahead and select the checkbox for **Auto-configure CloudWatch Logs** in the **Storage and Logging** section.
* That's it for rabbitmq, save your settings and lets go on to memcached.

#### memcached container configuration
* Same as above for the **Container name** and **Image** settings, using your memcached information this time. **NOTE**: I'm not going to mention these settings going forward as they are all the same, just replace the name of the container/image.
* I set my memory soft limit here to 100 MiB as memcached is a fairly lightweight image (This may change later once you start adding some [fact caching](https://github.com/ansible/awx/blob/bfea00f6dc6af0fb01057ce38e9d0337e6c589aa/docs/fact_cache.md#tower-as-an-ansible-fact-cache)).
* After checking the box for **Essential** there shouldn't be any other settings required unless you would like logging enabled to CloudWatch (Once again I leave this disabled as personal preference)
* There are no environment variables required for this container.
* That's all for memcached, save your settings and we'll go on to the awx containers.

#### awx-task container configuration
* For **Memory Limits (MiB)** I set a soft limit of 1536. Again, this could change in the future depending on usage.
* Once again ensure that the **Essential** checkbox is selected.
* There are quite a few **Env Variables** that need to be set:
    * ```AWX_ADMIN_PASSWORD: PERSONAL_CHOICE``` **UNSAFE** Set this to what you want the default to be (The [awx installation procedure](https://github.com/ansible/awx/blob/devel/INSTALL.md)) uses ```password```
    * ```AWX_ADMIN_USER: admin```
    * ```DATABASE_HOST: DB_ENDPOINT``` replace **DB_ENDPOINT** with the endpoint we jotted down at the end of the database creation section above.
    * ```DATABASE_NAME: awx```
    * ```DATABASE_PASSWORD: PASSWORD``` **UNSAFE** replace **PASSWORD** with the one we jotted down during the creation of our db instance.
    * ```DATABASE_PORT: 5432```
    * ```DATABASE_USER: awx```
    * ```MEMCACHED_HOST: localhost``` **IMPORTANT** a cool feature of ECS Fargate is that any containers within the same task can talk to each other by using **localhost:PORT**
    * ```MEMCACHED_PORT: 11211```
    * ```RABBITMQ_HOST: localhost```
    * ```RABBITMQ_PASSWORD: guest``` **UNSAFE**
    * ```RABBITMQ_PORT: 5672```
    * ```RABBITMQ_USER: guest```
    * ```RABBITMQ_VHOST: awx```
    * ```SECRET_KEY: PERSONAL_CHOICE``` **UNSAFE**;**IMPORTANT** Whatever you set this to needs to remain the same between versions of awx or you won't be able to decrypt credentials.
* For this container, I check the box for **Auto-configure CloudWatch Logs** so that we have some visibility into any errors
* That's all for awx-task, time to move to our last one, awx-web

#### awx-web container configuration
* This one is easy. Every setting that I used above for awx-task is exactly the same for awx-web. Once you have that done, lets move on to the final steps

### Configure Task IAM Role
* We are almost ready to click **Create**, but first lets go back up to the top and click the **IAM Console** link to open it up in another tab:

    {{< img src="fargate2_task_create_iam.png" sizes="55vw" alt="IAM Link" >}}

* On the IAM console, click **Create role**
* Select the **AWS service** trusted entity type. Then select **EC2 Container Service**. Finally, select **EC2 Container Service Task**:

    {{< img src="fargate2_iam_trustedentity.png" sizes="55vw" alt="IAM Trusted Entity" >}}

* On the **Attach permissions policies** page select the following policies:
    * ```AmazonEC2FullAccess```
    * ```AmazonECSServiceRolePolicy```
* This should allow your AWX playbooks to create EC2 resources without credentials (in theory, I haven't tested this yet).
* On the **Review** page, name your role and put in a role description if you desire.
* Click **Create role** and then go back to your tab with the ECS Task settings page open.
* Select your newly created role in the **Task Role** dropdown, and finally click on **Create** at the bottom of the page.

# Create and start an ECS Service
* Back on the main page of Elastic Container Service, select the **Clusters** menu and click on the one we created earlier.
* You should now be on the **Services** tab of the cluster. Click on **Create**
* Select the following configuration:
    * Launch Type: ```FARGATE```
    * Task Definition: select the task definition we created above
    * Cluster: (Should already be set to the cluster we created earlier)
    * Service name: whatever you want, probably ```AWX```
    * Number of tasks: ```1```
    * Minimum healthy percent: ```100```
    * Maximum percent: ```200``` **NOTE** I don't think this is ideal because we don't want two tasks accessing the PostgreSQL database at the same time, but ECS won't allow you to set both the minimum and maximum to 100.
* Click **Next step** and proceed to the network settings:
    * Cluster VPC: Whatever VPC you have your PostgreSQL database in
    * Subnets: Any subnet within the VPC should be fine. I only select one, since we only have one task
    * Security groups: Click **Edit**
        * Create a new group that allows access to the 8052 port only from your public IP address. You can check your public IP at [whatismyipaddress.com](https://whatismyipaddress.com/). Remember this security group name so that we can allow access to the database.

            {{< img src="fargate2_service_sg.png" sizes="50vw" alt="security group" >}}

    * Auto-assign public IP: ```ENABLED```
    * Load balancer type: ```None```
    * Click **Next step**
* On the **Set Auto Scaling** page leave it at **Do not adjust the service’s desired count** and click **Next step**
* Finally, click **Create Service** which should launch your task and attempt to keep it up in the event of any container failure.

#### Allow access to the database from the containers
* Our last step is to configure access to the database. Navigate to the VPC service in your AWS console and select **Security Groups** from the left menu.
* First find the security group id that you assigned to the ECS Task (sg-XXXXXXXX) and copy it to your clipboard.
* Now find the security group that is assigned to your RDS database. On the **Inbound Rules** tab click on **Edit**
    * Set one rule that allows the 5432 port through the *TCP (6)* protocol with the source being the security-group pasted from your clipboard. See below:

        {{< img src="fargate2_db_securitygroup.png" sizes="40vw" alt="postgresql security group" >}}
* That's it, now your containers should be able to talk to the database.

# Final Thoughts and next steps

Congrats, you should now have an up and running AWX installation. Navigate to the **Tasks** tab of your ECS Cluster. There should be a link to your running task in the **Task** column. Here you will find the **ENI Id** (Elastic Network Interface) that is assigned to your task. If you click on that you will find the public IP that you can access AWX with. You can use the IP to access AWX: ```http://<PUBLIC_IP>:8052```

There is still some more we can do here to make our deployment more production ready, but is outside the scope of this guide:
1. We can run this task on a private VPC that is only reachable over a VPN from your office (home?) to your VPC.
2. Add in an application load balancer or an nginx container for HTTPS termination (I plan to write a post on how I achieve this with nginx)
3. As mentioned in the beginning, there is a safer way to pass environment variables to the tasks [documented here](https://aws.amazon.com/blogs/security/how-to-manage-secrets-for-amazon-ec2-container-service-based-applications-by-using-amazon-s3-and-docker/)
4. Automate the creation of container images and deployment using ansible

### Config output for use with the aws cli

As promised, included below is a json file dump of settings that can be used to re-create the task definition (not the cluster or service) with the aws cli.

Some things will need to be replaced in this output:

* ```ACCOUNT_NUMBER```: replace with your AWS account number
* ```taskRoleArn```: this variable will need to have the full arn to the task role we set up above. You will see mine listed, but with my account number removed.
* ```taskDefinitionArn```: this arn will depend on what you named your task definition. You will see mine listed, but with my account number removed.
* ```DB_ENDPOINT```: Change to your DB endpoint.
* ```DB_PASS```: Like DB_ENDPOINT, you'll need to change this to your configured password

```json
{
    "taskDefinition": {
        "status": "ACTIVE",
        "memory": "4096",
        "networkMode": "awsvpc",
        "family": "AWX",
        "placementConstraints": [],
        "requiresAttributes": [
            {
                "name": "ecs.capability.execution-role-ecr-pull"
            },
            {
                "name": "com.amazonaws.ecs.capability.docker-remote-api.1.18"
            },
            {
                "name": "ecs.capability.task-eni"
            },
            {
                "name": "com.amazonaws.ecs.capability.ecr-auth"
            },
            {
                "name": "com.amazonaws.ecs.capability.task-iam-role"
            },
            {
                "name": "ecs.capability.execution-role-awslogs"
            },
            {
                "name": "com.amazonaws.ecs.capability.logging-driver.awslogs"
            },
            {
                "name": "com.amazonaws.ecs.capability.docker-remote-api.1.21"
            },
            {
                "name": "com.amazonaws.ecs.capability.docker-remote-api.1.19"
            }
        ],
        "cpu": "2048",
        "executionRoleArn": "arn:aws:iam::ACCOUNT_NUMBER:role/ecsTaskExecutionRole",
        "compatibilities": [
            "EC2",
            "FARGATE"
        ],
        "volumes": [],
        "requiresCompatibilities": [
            "FARGATE"
        ],
        "taskRoleArn": "arn:aws:iam::ACCOUNT_NUMBER:role/ecsFullEC2Access",
        "taskDefinitionArn": "arn:aws:ecs:us-east-1:ACCOUNT_NUMBER:task-definition/AWX:10",
        "containerDefinitions": [
            {
                "memoryReservation": 300,
                "environment": [
                    {
                        "name": "RABBITMQ_DEFAULT_VHOST",
                        "value": "awx"
                    }
                ],
                "name": "rabbitmq",
                "mountPoints": [],
                "image": "ACCOUNT_NUMBER.dkr.ecr.us-east-1.amazonaws.com/my-rabbitmq:latest",
                "cpu": 0,
                "portMappings": [],
                "logConfiguration": {
                    "logDriver": "awslogs",
                    "options": {
                        "awslogs-region": "us-east-1",
                        "awslogs-stream-prefix": "ecs",
                        "awslogs-group": "/ecs/AWX"
                    }
                },
                "essential": true,
                "volumesFrom": []
            },
            {
                "memoryReservation": 100,
                "environment": [],
                "name": "memcached",
                "mountPoints": [],
                "image": "ACCOUNT_NUMBER.dkr.ecr.us-east-1.amazonaws.com/my-memcached:latest",
                "cpu": 0,
                "portMappings": [],
                "logConfiguration": {
                    "logDriver": "awslogs",
                    "options": {
                        "awslogs-region": "us-east-1",
                        "awslogs-stream-prefix": "ecs",
                        "awslogs-group": "/ecs/AWX"
                    }
                },
                "essential": true,
                "volumesFrom": []
            },
            {
                "memoryReservation": 1536,
                "environment": [
                    {
                        "name": "MEMCACHED_PORT",
                        "value": "11211"
                    },
                    {
                        "name": "DATABASE_NAME",
                        "value": "awx"
                    },
                    {
                        "name": "AWX_ADMIN_PASSWORD",
                        "value": "password"
                    },
                    {
                        "name": "DATABASE_HOST",
                        "value": "DB_ENDPOINT"
                    },
                    {
                        "name": "DATABASE_PORT",
                        "value": "5432"
                    },
                    {
                        "name": "RABBITMQ_PASSWORD",
                        "value": "guest"
                    },
                    {
                        "name": "SECRET_KEY",
                        "value": "awxsecret"
                    },
                    {
                        "name": "AWX_ADMIN_USER",
                        "value": "admin"
                    },
                    {
                        "name": "RABBITMQ_PORT",
                        "value": "5672"
                    },
                    {
                        "name": "RABBITMQ_USER",
                        "value": "guest"
                    },
                    {
                        "name": "MEMCACHED_HOST",
                        "value": "localhost"
                    },
                    {
                        "name": "RABBITMQ_VHOST",
                        "value": "awx"
                    },
                    {
                        "name": "RABBITMQ_HOST",
                        "value": "localhost"
                    },
                    {
                        "name": "DATABASE_PASSWORD",
                        "value": "DB_PASS"
                    },
                    {
                        "name": "DATABASE_USER",
                        "value": "awx"
                    }
                ],
                "name": "awx_task",
                "links": [],
                "mountPoints": [],
                "image": "ACCOUNT_NUMBER.dkr.ecr.us-east-1.amazonaws.com/my-awx-task:latest",
                "essential": true,
                "portMappings": [],
                "logConfiguration": {
                    "logDriver": "awslogs",
                    "options": {
                        "awslogs-region": "us-east-1",
                        "awslogs-stream-prefix": "ecs",
                        "awslogs-group": "/ecs/AWX"
                    }
                },
                "cpu": 0,
                "volumesFrom": []
            },
            {
                "memoryReservation": 1536,
                "environment": [
                    {
                        "name": "MEMCACHED_PORT",
                        "value": "11211"
                    },
                    {
                        "name": "DATABASE_NAME",
                        "value": "awx"
                    },
                    {
                        "name": "AWX_ADMIN_PASSWORD",
                        "value": "password"
                    },
                    {
                        "name": "DATABASE_HOST",
                        "value": "DB_ENDPOINT"
                    },
                    {
                        "name": "DATABASE_PORT",
                        "value": "5432"
                    },
                    {
                        "name": "RABBITMQ_PASSWORD",
                        "value": "guest"
                    },
                    {
                        "name": "SECRET_KEY",
                        "value": "awxsecret"
                    },
                    {
                        "name": "AWX_ADMIN_USER",
                        "value": "admin"
                    },
                    {
                        "name": "RABBITMQ_PORT",
                        "value": "5672"
                    },
                    {
                        "name": "RABBITMQ_USER",
                        "value": "guest"
                    },
                    {
                        "name": "MEMCACHED_HOST",
                        "value": "localhost"
                    },
                    {
                        "name": "RABBITMQ_VHOST",
                        "value": "awx"
                    },
                    {
                        "name": "RABBITMQ_HOST",
                        "value": "localhost"
                    },
                    {
                        "name": "DATABASE_PASSWORD",
                        "value": "DB_PASS"
                    },
                    {
                        "name": "DATABASE_USER",
                        "value": "awx"
                    }
                ],
                "name": "awx_web",
                "links": [],
                "mountPoints": [],
                "image": "ACCOUNT_NUMBER.dkr.ecr.us-east-1.amazonaws.com/my-awx-web:latest",
                "essential": true,
                "portMappings": [],
                "logConfiguration": {
                    "logDriver": "awslogs",
                    "options": {
                        "awslogs-region": "us-east-1",
                        "awslogs-stream-prefix": "ecs",
                        "awslogs-group": "/ecs/AWX"
                    }
                },
                "cpu": 0,
                "volumesFrom": []
            }
        ],
        "revision": 1
    }
}
```
