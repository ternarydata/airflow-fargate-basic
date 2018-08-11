# Airflow on AWS Fargate

[Airflow](https://github.com/apache/incubator-airflow) is a platform to programmatically author, schedule, and monitor workflows and is a valuable tool in modern heterogeneous data environments involving numerous systems on-prem and in the cloud.

Airflow is extremely scalable and flexible, but at a cost of complexity. A full Airflow system with distributed execution utilizes two databases and several services. Containers are a great way to manage this complexity and maintain a resilient system; AWS Fargate allows us to run a "serverless" container cluster, i.e., without managing underlying EC2 instances. This tutorial will demonstrate an extremely simple Airflow setup running in a single container, and could serve as a starting point for building out a full Airflow system.

## Basics

We use the puckel/docker-airflow image available on Docker Hub, built from the
Dockerfile at https://github.com/puckel/docker-airflow/

This guide is based on the Fargate tutorial [here](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_tutorial_fargate.html).

## The sequential executor

The sequential executor is the most basic option for an Airflow deployment,
generally suitable for testing and development.
(See [here](https://github.com/puckel/docker-airflow/#usage) and
[here](https://airflow.apache.org/_modules/airflow/executors/sequential_executor.html).)

## Caveats

* ECS is not covered in the AWS free tier, so following these instructions can cost you money. (We spent 5 cents creating the guide!) Make sure to tear down when you're done to minimize charges.

* This setup is extremely limited - you can't do any interesting experimentation since we haven't set up a directory for DAG files. We'll discuss this in a future guide.

## Initial configuration.

Work through the prerequisites and Step 1 of the [tutorial](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_tutorial_fargate.html)
to install the ECS CLI and create a task execution role. (Note that
`task-execution-assume-role.json`
is included in this repo.)
Next, we configure an ECS Fargate cluster.
```
ecs-cli configure --cluster airflow --region us-west-1 --default-launch-type FARGATE --config-name airflow
```
For security, we handle things a little differently than the ECS tutorial. Instead of passing credentials at the command
line, we'll use placeholders for the key id and secret key.
```
ecs-cli configure profile --access-key None --secret-key None --profile-name airflow
```
Now, open `~/.ecs/credentials`.
```
version: v1
default: airflow
ecs_profiles:
  airflow:
    aws_access_key_id: None
    aws_secret_access_key: None
```
Fill in the key information and save.

## Start the cluster

```
ecs-cli up --cluster airflow --ecs-profile airflow
```

This will create resources (a VPC, etc.) for our cluster; however, we don't have to manage EC2 instances with Fargate, so no compute resources will be provisioned until we start a container. Take note of the VPC and subnet
IDs in the command output - we'll need them shortly.

Next, we create a security group allowing inbound internet access once
we spin up the web server.
```
aws ec2 create-security-group --group-name "airflow-sg" --description "Airflow security group" --vpc-id "VPC_ID"
```
`"VPC_ID"` is from the VPC created when we ran `ecs-cli up`.

We've included Docker Compose and ECS parameter files in the repo.
`ecs-params.yml` is borrowed from the ECS tutorial. To start the Airflow
web server, you need to edit `ecs-params.yml` with the subnet and security group
IDs that you got when creating the cluster and security group. To start
the server, run
```
ecs-cli compose --project-name airflow service up --cluster-config airflow
```
While you wait for the server to start, you can edit the Airflow security group to allow access from your browser. In the AWS console
for region us-west-2,
go to EC2 &rarr; security groups and select the Airflow security group;
select the tab for inbound rules and edit; Add a custom TCP rule for port 8080;
select "My IP" as the source and save.

![](/img/security-group.png)

Return to the terminal window where you started the web server. Once this
command completes, you can check the server status by running
```
ecs-cli compose --project-name airflow service ps --cluster-config airflow
```
Keep running this command until the server health goes from "UNKNOWN" to
"HEALTHY". Copy the IP address from the output and paste it in your browser with
the port in the format `IP_ADDRESS:8080`. When you hit enter, the Airflow web
interface should load.

![](/img/airflow-interface.png)

## Tear down

Start by bringing down the the web server.
```
ecs-cli compose --project-name airflow service down --cluster-config airflow
```

Next, we bring down the cluster and delete its associated CloudFormation
stack; note that as written, the tear down instructions in the ECS tutorial
don't work. We encounter something of a catch-22:
CloudFormation can't delete the VPC while our security group exists and
we can't delete the security group while the VPC network interface is still
present. To work around this, start the CloudFormation tear down
```
ecs-cli down --force --cluster-config airflow
```
and try to delete the Airflow
security group from the console while you wait for the down command to
complete;
your delete request will succeed once CloudFormation deletes the network
interface. Go back to the terminal and wait for
CloudFormation to finish deleting resources. (You can also monitor CloudFormation through the console.)

## Next steps

* Try setting up a DAGs folder and running one of the [example DAGs](https://github.com/apache/incubator-airflow/tree/master/airflow/example_dags).
* Try using one of the other Dockercompose files from the puckel/docker-airflow [project](https://github.com/puckel/docker-airflow/)
to run the local executor or celery executor. This will mean setting up a multi-container cluster and running one or more databases. You will need to change `ecs-params.yml` to accomodate the new services; you will also need to modify the compose file to make it compatible with Fargate. Note that these containerized databases are not ideal for production deployment - RDS/Postgres and Elasticache/Redis are much more resilient options.
