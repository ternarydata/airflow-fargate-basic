# Airflow on AWS Fargate

Examples of running Airflow on AWS Fargate.

## Basics

We use the puckel/docker-airflow image available on Docker Hub, built from the
Dockerfile at https://github.com/puckel/docker-airflow/

This guide is based on the Fargate tutorial [here]
(https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_tutorial_fargate.html).

## Airflow with the sequential executor

The sequential executor is the most basic option for an Airflow deployment,
generally suitable for testing and development.
(See [here] (https://github.com/puckel/docker-airflow/#usage) and
[here]
(https://airflow.apache.org/_modules/airflow/executors/sequential_executor.html).)

### Initial configuration.

Work through the prerequisites and Step 1 of the [tutorial]
(https://docs.aws.amazon.com/AmazonECS/latest/developerguide/ECS_CLI_tutorial_fargate.html)
to install the ECS CLI and create a task execution role. (Note that
`task-execution-assume-role.json`
is included in this repo.)
Next, we configure an ECS Fargate cluster
```
ecs-cli configure --cluster airflow --region us-west-1 --default-launch-type FARGATE --config-name airflow
```
and store credentials.
```
ecs-cli configure profile --access-key AWS_ACCESS_KEY_ID --secret-key AWS_SECRET_ACCESS_KEY --profile-name airflow
```
Note that the CLI has no facilities for viewing
the configuration you've stored. You can view and directly edit
the config by looking at the
text files in `~/.ecs/`.

### Start the cluster

```
ecs-cli up --cluster airflow --ecs-profile airflow
```

This will create resources (a VPC, etc.) for our cluster, but the cluster will
initially be empty since we're using Fargate. Take note of the VPC and subnet
ids in the command output - we'll need them shortly.

Next, we create a security group that will allow inbound internet access once
we spin up the web server.
```
aws ec2 create-security-group --group-name "airflow-sg" --description "Airflow security group" --vpc-id "VPC_ID"
```
`"VPC_ID"` is from the VPC created when we ran `ecs-cli up`.

We've included Docker Compose and ECS parameter files in the repo.
`ecs-params.yml` is borrowed from the ECS tutorial. To start the Airflow
web server, you need to edit `ecs-params.yml` with the subnet and security group
ids that you got when you created the ECS cluster and security group. To start
the server, run
```
ecs-cli compose --project-name airflow service up --cluster-config airflow
```
While you wait for the server to start, you can edit the inbound rules for
the Airflow security group to allow access from your browser. In the AWS console
for region us-west-2,
go to EC2 &rarr; security groups and select the Airflow security group;
select the tab for inbound rules and edit; Add a custom TCP rule for port 8080;
select "My IP" as the source and save.

Return to the terminal window where you started the web server. Once this
command completes, you can check the server status by running
```
ecs-cli compose --project-name airflow service ps --cluster-config airflow
```
Keep running this command until the server health goes from "UNKNOWN" to
"HEALTHY". Copy the IP address from the output and paste it in your browser with
the port in the format `IP_ADDRESS:8080`. When you hit enter, the Airflow web
interface should load.

### Tear down

Make sure to tear down to minimize AWS charges. Start by bringing down the the
web server.
```
ecs-cli compose --project-name airflow service down --cluster-config airflow
```

Next, we bring down the cluster and delete its associated CloudFormation
stack; note that as written, the tear down instructions in the ECS tutorial
don't work. In fact, we encounter something of a catch-22:
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
CloudFormation to finish deleting resources. (You can also monitor cloud
formation through the console.)