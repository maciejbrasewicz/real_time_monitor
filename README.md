# real_time_monitor

  #### Hosting a live dashboard that is fed by near real-time data.

This is an ETL pipeline to pull crypto exchange data from CoinCap API and load it into our data warehouse.

![Screenshot from 2022-12-21 19-09-21_edit_150658914136383](https://user-images.githubusercontent.com/49028274/208975712-78d2b790-3be3-4cd2-9368-653f2adab195.png)

> **Note:** We use python to pull, transform and load data. Our warehouse is postgres. We also spin up a Metabase instance for our presentation layer.

> **Note:** All of the components are running as docker containers.

  

## Setup
### Pre-requisites

 -  [git](https://git-scm.com/book/en/v2/Getting-Started-Installing-Git)
 -  [AWS account](https://aws.amazon.com/)
 -  [AWS CLI installed](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html)  
 -  [Terraform](https://learn.hashicorp.com/tutorials/terraform/install-cli)
 -  [Docker](https://docs.docker.com/engine/install/)  and  [Docker Compose plugin](https://docs.docker.com/compose/install/)  
 - > **Docker compose v2:** The original python project, called `docker-compose`, aka v1 of docker/compose repo, has now been deprecated and development has moved over to v2. To install the v2 `docker compose` as a CLI plugin on Linux, supported distribution can now install the `docker-compose-plugin` package. E.g. on debian, I run `apt-get install docker-compose-plugin`. 

- > **AWS Configuration and credential file settings:** Read more at [ credential file settings](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-files.html) & [configuration settings and precendence](https://docs.aws.amazon.com/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-precedence)

## Start
### Setup your project locally and on the cloud.

```shell
# Clone the code as shown below.
https://github.com/maciejbrasewicz/real_time_monitor.git
cd real_time_monitor
```
**Replace content in the following files:**

1.  **[CODEOWNERS](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/.github/CODEOWNERS)**: change the user id from  `@maciejbrasewicz`  to your GitHub user id.
2.  **[cd.yml](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/.github/workflows/cd.yml)**: In this file change the  `real_time_monitor`  part of the  `TARGET` to your repository name. If you haven't changed the name, leave it like that
3.  **[variable.tf](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/terraform/variable.tf)**: change the default values for  `alert_email_id`   variable with your email address.
If you are going to use a different [EC2 instance type](https://aws.amazon.com/free/?all-free-tier.sort-by=item.additionalFields.SortRank&all-free-tier.sort-order=asc&awsf.Free%20Tier%20Categories=categories#compute&trk=0dfd4a54-15a6-4541-8087-fe17b7d183fa&sc_channel=ps&s_kwcid=AL!4422!3!536392697404!e!!g!!ec2%20instance%20types&ef_id=Cj0KCQiA4uCcBhDdARIsAH5jyUl2ZD_X00lz5tCVec-xgazIq05UJVs29QOyPYQ2yNAaQgIBP56RfogaAkMwEALw_wcB:G:s&s_kwcid=AL!4422!3!536392697404!e!!g!!ec2%20instance%20types) than t2.micro, you can also change that in this file.
> **Note:**  when asked which purchasing option is right for me? I will answer in one of my blog posts. There is always a "better option"
4. -   **[main.tf](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/terraform/main.tf)**: make sure `Clone git repo to EC2` param is customized for your needs (line 112).

**[this file](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/terraform/main.tf)** defines all the services we need. In our main.tf, we create an EC2 instance, security group where we configure access and a cost alert. For instance, the security group for access to EC2: 

```shell
# Create security group for access to EC2 from your Anywhere
resource "aws_security_group" "sde_security_group" {
  name        = "sde_security_group"
  description = "Security group to allow inbound SCP & outbound 8080 (Airflow) connections"

  ingress {
    description = "Inbound SCP"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
...
```

The following commands will allow us to configure the project:

```shell
make up # docker containers & runs migrations under ./migrations
make ci # Runs auto formatting, lint checks, & all the test files under ./tests

# Create AWS services with Terraform
make tf-init
make infra-up # type in yes after verifying the changes TF will make or `terraform apply -auto-approve` to avoid interactive prompt in the future.
```
Wait until the EC2 instance is initialized, you can check this via your AWS UI
See "Status Check" on the EC2 console, it should be "2/2 checks passed" before proceeding. 
  
In the main.tf file we have created security group for access to EC2 . Now, just create an inbound rule for traffic on `port 3000`  to open Metabase at `http://your_ec2_public_dns:3000`. You can customize it to accept traffic from a particular IP, a particular IP range or Open to public. 

```shell
make cloud-metabase # this command will allow you to open the metabase at `http://your_ec2_public_dns:3000`
```
You can connect metabase to the warehouse with the configs in the [env](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/env) file. 

```shell
make db-migration # enter a description, e.g., create some schema
make warehouse-migration # to run the new migration on your warehouse
```


###  [Continuous delivery:](https://github.com/maciejbrasewicz/real_time_monitor/blob/main/.github/workflows/cd.yml)  

Set up the infrastructure with terraform, & defined the following repository secrets. You can set up the [repository secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets) by going to  `Settings > Secrets > Actions > New repository secret`.


1.  **`SERVER_SSH_KEY`**: We can get this by running  `terraform -chdir=./terraform output -raw private_key`  in the project directory and paste the entire content in a new Action secret called SERVER_SSH_KEY.
2.  **`REMOTE_HOST`**: Get this by running  `terraform -chdir=./terraform output -raw ec2_public_dns`  in the project directory.
3.  **`REMOTE_USER`**: The value for this is  **ubuntu**.


### Tear down infra

Make sure to destroy your cloud infrastructure.

```shell
make down # Stop docker containers on your computer
make infra-down # type in yes after verifying the changes TF will make
```

