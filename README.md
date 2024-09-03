# Minimal Airflow Docker Compose deployment
## Overview

The application is based off of Docker Compose to fully deploy a functional Airflow environment.

The compose file is heavily based on the example given in the [Airflow documentation](https://airflow.apache.org/docs/apache-airflow/stable/howto/docker-compose/index.html)

## Docker Compose

The steps defined in the `compose.yaml` file are:

- Defines common environment variables for all containers(`airflow-comon-env`);
- defines three volumes based on project root directories (`dags`, `logs` and `plugins`);
- defines common dependencies for all containers on postgres and redis (`airflow-common-depends-on`);
- container definitions for each airflow components

## Troubleshooting process

First issues came up from a top level reading of the file, in which I changed the `POSTGRES_USER` variable from `admin` to `airflow`on line 31.

After that I initially had some issues with the user ID for airflow, which is by default `50000`. It would likely work if the UID was set to `1000`, but I would rather keep it at an expeted standard.

After getting the Airflow UI up and running I noticed that there weren't any DAGs, so I listed the contents of `/opt/airflow/dags`, only to find it empty. I then looked slightly to the left and saw that the `dags` directory was incorrectly named `dag`, and changed that as well.

After that I went throught the DAG to see if it looked right and my handy linter whispered to me that there was a missing `:` at function declaration.

With all that fixed, I ran my DAG on the interface. [Logs are attached](./logs/scheduler/2024-08-31/smooth.py.log).

## Airflow Architecture

There are several examples of the architecture in Airflow's own [documentation](https://airflow.apache.org/docs/apache-airflow/stable/core-concepts/overview.html). I added a few more things to better fit what I belive is the intended working state of this deployment:

![airflow_architecture](./assets/airflow_architecture_v2.drawio.png)

## Cloud Architecture Propositions

Sticking to the Docker Compose approach, we could set auto scaling groups tied to ECS in order to scale parts of the infrastructure. Only the workers need to be scaled up, so I belive we could simply implement them as an ASG:

![aws_airflow_asg](./assets/aws_airflow_asg.drawio.png)

I've placed a Load balancer up front, considering that there could be more than just airflow running in that infrastructure. I did not consider a Redis alternative in this example, but one would be needed in order to scale out to a larger size, but that would entail additional cost if integrated with AWS native tools such as ElastiCache. The relational database would be better integrated and more reliable as an RDS.

A more elegant solution would involve an EKS managed cluster that would essentially deploy all of the required containers as pods in it:

![aws_airflow_eks](./assets/aws_airflow_eks.drawio.png)

In this example, I believe the DAGs should be located in a Git repository (third party, in this case, such as GitHub or GitLab) to facilitate collaboration from DAG authors. Instead of using ElastiCache, I think running it in a container would prove cheaper, though maybe not as scalable. As with the ASG example, I believe only the workers need scaling at first, but other components can be scaled up and down easily with Kubernetes.

The first idea would be easier to implement at first, but would probably be harder to maintain in the long run.