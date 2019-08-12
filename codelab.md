author:            David Mattia
summary:           NodeJs App deployed to AWS using Terraform and DOcker
id:                node-terraform
categories:        Web
environments:      Web
status:            draft
feedback link:     github.com/dmattia
analytics account: 0

# Deploy NodeJs App on AWS Using Terraform and Docker

## Overview of the tutorial
Duration: 3:00

This tutorial shows you how to deploy a web app on AWS in a reliable way (similar to the way we do it at Transcend.io). We use the following stack, which will be doven into deeper in future sections:

* __NodeJs__: For making the web app

```bash
brew install node
```

* __Terraform__: Infrastructure as Code tool for deploying to AWS

```bash
brew install terraform
```

* __Docker__: For bundling the dependencies of the web app into a Container

```bash
brew install docker
```

* __Amazon Web Services__: For hosting the web app

```bash
brew install awscli
```

Positive
: It may be helpful to familiarize yourself with these services using our internal Notion docs

This tutorial assumes basic familiarity with web applications and hosting

## Create a basic NodeJS app
Duration: 05:00

### In this section

We will create a basic web app that can run on localhost.

### Tutorial

- In a new directory, initialize a `package.json` file to track dependencies

```bash
npm init
```

- For this app, we will use `Express` as our web server

```bash
npm install --save express
```

- Create a file named `app.js` for the web server with the following contents:

```javascript
const app = require('express')();

app.get('/', (req, res) => {
  res.send('Hello, World!\n');
});

app.listen(3000, '0.0.0.0')
```

This file, as you might expect, starts a web server on localhost (If unfamiliar, that is what `0.0.0.0` refers to) on port `3000`.

- Verify that your server works. On the command line, run:

```bash
node app.js
```

[Click here to view the app in your browser](http://localhost:3000/)

## Make a Docker Image
Duration: 10:00

In the last step, we used a `package.json` file to explicitly remember our dependencies. This is a common approach for ensuring that other team members or cloud environments can easily use the same libraries as we do.

However, it is not always enough. The previous step lacked:

- __Complete Dependencies__: While a `package.json` lists javascript dependencies, it does not list all dependencies, such as `Node` itself or even that it must run on some operating system.
- __Container Isolation__: By creating a container, you can explicitly decide how that container can communicate with other processes, helping with security.
- __Full Instructions for running the app__: A `package.json` may have helpful scripts such as `npm run` for running the app, there is no set of commands that will properly build all apps.

With Docker, you can:

- Fully list all dependencies (excluding a linux kernal, which is assumed)
- Isolate your web app by default
- Give simple instructions for how to run the app

### Blueprints of our image

Create a file named `Dockerfile`. In it, paste the following code:

```docker
FROM node:10
WORKDIR /usr/src/app
COPY package*.json ./
RUN npm i
COPY . .
EXPOSE 3000
CMD [ "node", "app.js" ]
```

Let's examine this code line by line:

```docker
FROM node:10
```

This says that your app will depend on `Node` version 10.

```docker
WORKDIR /usr/src/app
```

This says that for any other commands in your docker file, you will be in the `/usr/src/app` directory of the virtual file system.

```docker
COPY package*.json ./
```

This tells the image to copy over your `package.json` and `package-lock.json` files. The `COPY` command takes its first argument from your current directory (outside of docker) and its second argument relative to your `WORKDIR` inside docker.

```docker
RUN npm i
```

This installs the dependencies you listed in the `package.json` file.

```docker
COPY . .
```

This copies over all the other files from your current workspace, minus those in your `.dockerignore` file. We do not want to copy over our `node_modules` because we already ran `npm i` last step. So we can create a new file named `.dockerignore` with the contents: 

```bash
node_modules
npm-debug.log
```

This also saves a bit of time as `node_modules` can be quite large and slow to copy.

```docker
EXPOSE 3000
```

By default, docker will not give any external process access to inside the container. We want to allow one port to be exposed so that outsiders can access the webapp. This exposes port `3000`, which happens to be the port we hosted our `Node` app on locally earlier.

```docker
CMD [ "node", "app.js" ]
```

The last step is to tell the container to run the `app.js` file we copied over using the `node` binary, which hosts the app.

### Running our image locally

Let's build an image!

```bash
docker build -t some-image-name .
```

This will build an image to your local machine named `some-image-name`.

In the output, you may notice that it also tags your image as `some-image-name:latest`. Tags are a way to version and, well, tag, your images in case you ever want multiple images related to the same app/job.

Now, lets run it on localhost:

```bash
docker run -p 12345:3000 -d some-image-name 
```

This command says to run the `some-image-name` image locally. The `-p 12345:3000` flag is an example of port forwarding. Your local machine and the docker container have their own, distinct sets of ports. Port forwarding enables you to say "Whenever someone asks for my 12345 port, send them to some docker container's 3000 port instead." This is kind of like a proxy server, if that helps you.

[Click here to view the app in your browser](http://localhost:12345/)

### Kill the Docker process

Let's cleanup, as we no longer want to run this web app locally.

Running

```bash
docker ps
```

will give you an overview of the currently running docker images. Copy the name of the image you just started, and run

```bash
docker kill <process_name>
```

to stop it. You may want to run `docker ps` one last time to verify it has stopped.

## Terraform. Wow. What is that?
Duration: 3:00

Terraform lets you declare infrastructure as code. This is a pretty popular paradigm predicated on the idea that code is easier to version, share, and change than infrastructure made through web consoles. Here are a few more specific benefits of Terraform:

* You can put your infrastructure under version control.
* Ease of automation. If your infra is defined in code, it is easy to automate deploys.
* Speed of automation. Terraform deploys changes very quickly, compared to a human manually navigating a cloud service provider's website.
* Running `terraform plan` shows you what infra will change before you make any changes so you can easily see exactly how it will change.
* Running `terraform graph` gives you a visual representation of how your infra exists.
* Immutable Infrastructure: By deploying new servers with most changes, terraform ensures your system works properly. There's far less likelihood that someone ssh'd in to your EC2 instance at some point and ran commands to fix the system that were never documented. With terraform, changes are forced to be reproducible.
* Declarative code: Say your terraform code says to make an EC2 server. If you run `terraform apply` 10 times, you'll still only have one server, not 10. It automatically cleans up old, no longer necessary resources to save some money and confusion.
* Masterless: Terraform does not require you to run any sort of server that your infra must depend on. You do need to keep track of state, but we'll get to that later.
* It's open sourced! Unlike AWS CloudFormation, you [can easily find the source code for the terraform CLI online](https://github.com/hashicorp/terraform).

## Host your image on AWS ECR
Duration: 15:00

ECR is the Amazon Elastic Container Registry. If you have ever used Docker Hub, it is basically the same thing. At Transcend, we use ECR because it gives us cheap/free private repos, unlike Docker Hub (TODO: verify this. Right now this is just my best guess).

It's entire job is to host Docker images in repos. You use it similarly to using S3, where you create a repo (instead of an S3 bucket) and then can upload images (instead of files) to that repo. Just like S3, it keeps track of versions and tags for you.

Positive
: You should have already set up an AWS profile locally that you have permissions to deploy AWS resources with. You can also use a personal account (which I did), or just follow along without actually deploying anything if you won't be working on Terraform changes very often.

Create a new folder named `deployment` to store your terraform code and `cd` into it.

To start, create a file named `provider.tf`. In this, we will specify that we want to deploy to AWS specifically. Terraform supports many cloud providers. This looks like:

```terraform
provider "aws" {
  region  = "eu-west-1"
  profile = "test"
}
```

This says that all deploys will be in the `eu-west-1` region. It also says that I would like to use my `test` profile in the `awscli`, which I set up to be my personal account.

Now, create a file named `ecr.tf` with the contents:

```terraform
resource "aws_ecr_repository" "ecr_repo" {
    name = "ecr_example_repo"
}
```

This follows the syntax:

```terraform
resource "some aws resource" "some terraform name that lets you reference this resource from other resources in terraform" {
    name = "the name that will appear in the AWS console for this resource"
    ...other args...
}
```

To find a list of usable aws resource names and the arguments they take, [check out the docs](https://www.terraform.io/docs/providers/aws/index.html).

### Deploy to AWS

Run the command

```bash
terraform init
```

to initialize your directory as containing terraform code. This will download all plugins available from the aws provider you listed in `provider.tf`

Next, run

```bash
terraform plan
```

This step is optional, but highly recommended anytime you change infrastructure.

You should see output that looks something like:

```plaintext
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  # aws_ecr_repository.ecr_repo will be created
  + resource "aws_ecr_repository" "ecr_repo" {
      + arn                  = (known after apply)
      + id                   = (known after apply)
      + image_tag_mutability = "MUTABLE"
      + name                 = "ecr_example_repo"
      + registry_id          = (known after apply)
      + repository_url       = (known after apply)
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

That looks good. It shows us that an `aws_ecr_repository` will be created. As this matches our expectation, we can run:

```bash
terraform apply
```

After confirming the plan, you can go to your ECR page on your AWS account and will see that an empty repository was made!

Negative
: If you don't see the repo, ensure that you are looking in the correct region.

### Pushing our Docker Image to ECR

We have a repo, now we need to push our local docker image to it. This is done in two steps, tagging our local image and pushing our changes.

Find the repository url from your docker image, and copy it. Then, run:

```bash
docker tag some-example-image:latest <repo_url>:latest
```

This is kind of similar to a `git remote add origin <repo_url>` in git.

Then, run:

```bash
docker push <repo_url>:latest
```

to upload the image to the remote repo. This is similar to a `git push` in git.

Head back to your AWS console, and verify you can see the image you uploaded.

## The Rest of the Owl
Duration: 30:00

The remaining step is to deploy the ECR image to AWS, which requires quite a few aws services, each with some terraform code to specify it.

Negative
: Terraform can be very verbose for simple examples, which you'll see in this section. With great control comes annoying levels of specification. This is as basic an example I could think of, with no logging, load balancer, etc.

Let's start with the fun stuff, permissions and roles!

### IAM roles

Create a file named `iam.tf` with the contents:

```terraform
resource "aws_iam_role" "ecs_role" {
  name = "ecs_role_example_app"

  assume_role_policy = <<POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "",
      "Effect": "Allow",
      "Principal": {
        "Service": "ecs-tasks.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
POLICY
}

resource "aws_iam_role_policy_attachment" "ecs_policy_attachment" {
  role = "${aws_iam_role.ecs_role.name}"

  // This policy adds logging + ecr permissions
  policy_arn = "arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy"
}
```

This creates a new IAM role named `ecs_role_example_app` with an attached `AmazonECSTaskExecutionRolePolicy`. This policy ensures that the role will be able to pull from ECR.

### A Custom Virtual Private Cloud

Next, create a file `network.tf` that contains:

```terraform
resource "aws_vpc" "vpc_example_app" {
    cidr_block = "10.0.0.0/16"
    enable_dns_hostnames = true
    enable_dns_support = true
}

resource "aws_subnet" "public_a" {
    vpc_id = "${aws_vpc.vpc_example_app.id}"
    cidr_block = "10.0.1.0/24"
    availability_zone = "${var.aws_region}a"
}

resource "aws_subnet" "public_b" {
    vpc_id = "${aws_vpc.vpc_example_app.id}"
    cidr_block = "10.0.2.0/24"
    availability_zone = "${var.aws_region}b"
}

resource "aws_internet_gateway" "internet_gateway" {
    vpc_id = "${aws_vpc.vpc_example_app.id}"
}

resource "aws_route" "internet_access" {
    route_table_id = "${aws_vpc.vpc_example_app.main_route_table_id}"
    destination_cidr_block = "0.0.0.0/0"
    gateway_id = "${aws_internet_gateway.internet_gateway.id}"
}

resource "aws_security_group" "security_group_example_app" {
    name = "security_group_example_app"
    description = "Allow TLS inbound traffic on port 80 (http)"
    vpc_id = "${aws_vpc.vpc_example_app.id}"

    ingress {
        from_port = 80
        to_port = 3000
        protocol = "tcp"
        cidr_blocks = ["0.0.0.0/0"]
    }

    egress {
        from_port = 0
        to_port = 0
        protocol = "-1"
        cidr_blocks = ["0.0.0.0/0"]
    }
}
```

This creates a VPC that other resources can go into. It has a public subnet (in two availability zones) that can connect to the internet via an internet gateway.

For security reasons, we specify that only port 3000 should be exposed to the public, but outgoing traffic from our resources is unrestricted.

If this is confusing (it was for me at first), then I would recommend [this youtube playlist](https://www.youtube.com/watch?v=pAOrBxZ7584).

### Fargate deployment

Fargate is the final, and most exciting step. It is a service that deploys Docker containers for us, which means we're finally at the step of having our simple NodeJs app running on AWS infrastructure!

Create a file `fargate.tf` with the contents:

```terraform
resource "aws_ecs_task_definition" "backend_task" {
    family = "${var.ecs_family}"

    // Fargate is a type of ECS that requires awsvpc network_mode
    requires_compatibilities = ["FARGATE"]
    network_mode = "awsvpc"

    // Valid sizes are shown here: https://aws.amazon.com/fargate/pricing/
    memory = "512"
    cpu = "256"

    // Fargate requires task definitions to have an execution role ARN to support ECR images
    execution_role_arn = "${aws_iam_role.ecs_role.arn}"

    container_definitions = <<EOT
[
    {
        "name": "example_app_container",
        "image": "<your_ecr_repo_url>:latest",
        "memory": 512,
        "essential": true,
        "portMappings": [
            {
                "containerPort": 3000,
                "hostPort": 3000
            }
        ]
    }
]
EOT
}

resource "aws_ecs_cluster" "backend_cluster" {
    name = "backend_cluster_example_app"
}

resource "aws_ecs_service" "backend_service" {
    name = "backend_service"

    cluster = "${aws_ecs_cluster.backend_cluster.id}"
    task_definition = "${aws_ecs_task_definition.backend_task.arn}"

    launch_type = "FARGATE"
    desired_count = 1

    network_configuration {
        subnets = ["${aws_subnet.public_a.id}", "${aws_subnet.public_b.id}"]
        security_groups = ["${aws_security_group.security_group_example_app.id}"]
        assign_public_ip = true
    }
}
```

Please fill in where I specified `<your_ecr_repo_url>`

Fargate is a type of the Elastic Container Service, which has three concepts:

* Clusters: a logical grouping of tasks or services
* Services: where you group tasks and specify how many of each task you want running at a time
* Tasks: defines a specified docker image to use, along with what IAM roles are needed to use it

It should be pretty easy to map those concepts to the three terraform resource blocks above.

There are quite a few arguments I won't go over in detail here, but they mostly relate to:

* Connecting the service to the VPC we created in `network.tf`
* Ensuring this is the cheapest possible Fargate instance you can run so you don't spend more than a few cents on your demo app.
* Connecting the task, service, and cluster to each other.
* Giving the web app a public IP Address so you can visit the website you just made.

Find the public IP Address on the task page in your AWS console, and go to `http://<your_public_ip>:3000` to view your super scalable hello world application!

## Sections coming sometime later

* Terraform variables
* Terraform outputs
* Logging
* Dev/Prod env modules
* AWS Tags
* Load balancing
* Moving Terraform state to the Cloud
* Using containers you didn't write, such as Datadog Agent
* SSL certs and Domain Names