+++
title = 'Terraforming a Load Balancer in AWS'
date = 2024-02-06T20:16:08Z
draft = false
+++

## About {#About}

This guide assumes you have already [installed](https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html) and [configured](https://docs.aws.amazon.com/cli/latest/userguide/cli-chap-configure.htm) the AWS CLI and have [Terraform installed on your computer](https://developer.hashicorp.com/terraform/tutorials/aws-get-started/install-cli). Throughout the guide are code snippets from the files in this project. To keep you on your toes they require you to input your resource names etc. No brainless copy/paste allowed.

This proejct was adapted from [a blog post](https://sharmilas.medium.com/a-step-by-step-guide-to-creating-load-balancer-and-ec2-with-auto-scaling-group-using-terraform-752afd44df8e) by Sharmila S. I went through the project myself and wrote this guide to solidify my understanding of how infrastructure on AWS can be deployed via Terraform.

## Contents {#contents}
- [0 - Prereqs](#0-Prereqs)
- [1 - Setup](#1-Setup)
- [2 - VPCs & Subnets](#2-vpc-sub)
- [3 - NAT & Gateways](#3-NAT-Gate)
- [4 - Load Balancer](#4-ELB)
- [5 - Auto-Scaling](#5-ASG)
- [6 - Security Groups](#6-SGs)
- [7 - Deploy & Troubleshoot](#7-troubleshooting)

---
## 0 - Prerequisite Knowledge {#0-Prereqs}

- Networking
    - [Public/private networking](https://simple.wikipedia.org/wiki/IP_address)
    - [Subnets](https://www.dummies.com/article/technology/information-technology/networking/general-networking/network-administration-subnet-basics-184551/)
    - [Routing](https://www.dummies.com/article/technology/information-technology/networking/general-networking/network-basics-routers-185331/)
    - [NAT](https://www.computernetworkingnotes.com/ccna-study-guide/basic-concepts-of-nat-explained-in-easy-language.html)
    - [Network Ports](https://simple.wikipedia.org/wiki/Network_port)
- AWS
    - [EC2 basics](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/concepts.html)
    - [Elastic Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/what-is-load-balancing.html)
    - [Auto Scaling Groups](https://docs.aws.amazon.com/autoscaling/ec2/userguide/auto-scaling-groups.html)
    - AWS Networking ([Elastic IP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html), [NAT Gateways](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-nat-gateway.html), [Route tables](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Route_Tables.html))
- Terraform
    - [Working with AWS](https://developer.hashicorp.com/terraform/tutorials/aws-get-started)
- GitHub
    - [Basic operation](https://docs.github.com/en/get-started/quickstart/hello-world)
    - [.gitignore file](https://docs.github.com/en/get-started/getting-started-with-git/ignoring-files)
    - [GitHub Actions](https://docs.github.com/en/actions/learn-github-actions/understanding-github-actions)

---

## 1 - Project Setup {#1-Setup}

After creating the project's root directory create the `main.tf` file and add the provider for AWS:

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~>4.0"
    }
  }
}
```

The provider will need configured to use the region in which the project will be deployed. If you have set up the AWS CLI with your access keys then Terraform should pick these up. Otherwise these can be specified here:


```hcl
provider "aws" {
  region                   = "<your-region-here>"
  shared_config_files      = ["/Path/to/.aws/config"]
  shared_credentials_files = ["/Path/to/.aws/credentials"]
  profile                  = "PROFILE"
}
```

If you haven't already set up a `.gitignore` file, do so now and add the Terraform state files to it or copy GitHub's [Terraform.gitignore template](https://github.com/github/gitignore/blob/main/Terraform.gitignore). It is bad practice to [store the state files somewhere public](https://developer.hashicorp.com/terraform/language/state/sensitive-data). If you're a madlad who likes his or her S3 buckets then add the following code to `main.tf` to store the state in an existing, private S3 bucket:

```hcl
terraform {
    backend "s3" {
      bucket = "<your-bucket-here>"
      region = "<your-region-here>"
      key    = "path/to/directory"
    }
}
```

With your state file safely locked away from prying eyes, go ahead and run `terraform init` to initalise the project by downloading the specified providers.

[\^top](#contents)

---

## 2 - Configuring The VPC & Subnets {#2-vpc-sub}

The project requires a VPC with two public subnets and one private subnet.

As this project involves several moving parts it's best to keep things organised. This is best done by having separate `.tf` files for each aspect. Create the `vpc-with-subnets.tf` file and add the block for the first subnet.

```hcl
resource "aws_vpc" "<vpc-name-here>" {
  cidr_block = "10.0.0.0/23"
  tags = {
    Name = "<vpc-name-here>"
  }
}
```
I've given it a nice big /23 subnet, but for production environments you'll have to consider how many addresses you'll actually need.

Next, add in the subnet code block for the first public subnet:

```hcl
resource "aws_subnet" "<subnet1-name-here>" {
  vpc_id                  = aws_vpc.<vpc-name-here>.id
  cidr_block              = "10.0.0.0/27"
  map_public_ip_on_launch = true
  availability_zone       = "<your-region-here>"
}
```

Note that the subnet size is smaller (this is a subnet of the VPC subnet) and that `map_public_ip_on_launch` is set to `true` as this subnet is to be public. Add the second subnet code block changing the details where appropriate. See if you can figure it out on your own using your networking knowledge.

<details>
<summary>(only click here if you're really stuck)</summary>  
  
The second subnet starts after the end of the first subnet. The mask is a /27 which gives us 32 addresses. Starting at address 0, the next subnet begins at 10.0.0.32. The subnet name is different to the first subnet and `map_public_ip_on_launch` is set to `true`.
```hcl
resource "aws_subnet" "<subnet2-name-here>" {
  vpc_id                  = aws_vpc.<vpc-name-here>.id
  cidr_block              = "10.0.0.32/27"
  map_public_ip_on_launch = true
  availability_zone       = "<your-region-here>"
}
```  
  

Hopefull you're just here checking your work, if not then never mind champ you'll get it next time. I believe in you.
</details>

Finally, add the private subnet code block. Note `map_public_ip_on_launch` is set to `false` this time. Why might that be?

```hcl
resource "aws_subnet" "<subnet3-name-here>" {
  vpc_id                  = aws_vpc.<vpc-name-here>.id
  cidr_block              = "10.0.1.0/27"
  map_public_ip_on_launch = false
  availability_zone       = "<your-region-here>"
}
```

[\^top](#contents)

---

## 3 - NAT Gateway & Route Tables {#3-NAT-Gate}

I've split this part into two files, `gateways-public.tf` and `gateways-private.tf`. Let's start with setting up the public gateway. This will allow the load balancer to communicate on the public internet so users can access the services and resources behind it.

### 3.1 - Public Gateway

Create `gateways-public.tf`, define the Internet Gateway, and place it in the VPC we created in the previous step:

```hcl
resource "aws_internet_gateway" "<gw1-name-here>" {
  vpc_id = aws_vpc.<vpc-name-here>.id
}
```

Now add in the code block for creating the route table. This requires the VPC ID, the route(s) it will hold, and the ID of the gateway it will sit on.

```hcl
resource "aws_route_table" "<rt1-name-here>" {
  vpc_id = aws_vpc.<vpc-name-here>.id
  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.<gw1-name-here>.id
  }
}
```

The gateway and route table are looking very nice but won't do anything until the table is attached to a subnet. The following code block associates the route table to our first public subnet:

```hcl
resource "aws_route_table_association" "<rta1-name-here>" {
  subnet_id      = aws_subnet.<subnet1-name-here>.id
  route_table_id = aws_route_table.<rt1-name-here>.id
}
```

As before, see if you can figure out how to repeat this for our second public subnet.

<details>
<summary>(only click here if you're really stuck)</summary>

```hcl
resource "aws_route_table_association" "<rta2-name-here>" {
  subnet_id      = aws_subnet.<subnet2-name-here>.id
  route_table_id = aws_route_table.<rt1-name-here>.id
}
```
</details>

### 3.2 - Private Gateway

Similar to the public gateway but set up to only allow requests from the load balancer to reach the EC2 instances whilst allowing all traffic from them to pass out. This is done with a NAT gateway. The gateway sits between the public subnet with internet access (created in the previous step) and the private subnet.

To access the internet, the gateway will need an elastic IP. Create `gateways-private.tf` and add the EIP block:

```hcl
resource "aws_eip" "<eip-name-here>" {
  depends_on = [aws_internet_gateway.<gw1-name-here>]
  vpc        = true
  tags = {
    Name = "<eip-name-here>"
  }
}
```

Note the use of `depends_on` - this means the EIP won't be created until after the referenced gateway is created. Can't assign an IP address to nothing. We'll be seeing it again.

The NAT gateway also `depends_on` the internet gateway existing:

```hcl
resource "aws_nat_gateway" "<natgw-name-here>" {
  allocation_id = aws_eip.<eip-name-here>.id
  subnet_id     = aws_subnet.<subnet1-name-here>.id
  tags = {
    Name = "<natgw-name-here>"
  }
  depends_on = [aws_internet_gateway.<gw1-name-here>]
}
```

We'll need a route table for the private subnet to connect out to the internet. This is done by providing a route to 0.0.0.0/0, which will match all destination IP addresses and act as a default route. 

```hcl
resource "aws_route_table" "<rt2-name-here>" {
  vpc_id = aws_vpc.<vpc-name-here>.id
  route {
    cidr_block     = "0.0.0.0/0"
    nat_gateway_id = aws_nat_gateway.<natgw-name-here>.id
  }
}
```

And finally, the route table will need associated with the private subnet. You know what to do here.

<details>
<summary>(only click here if you're really stuck)</summary>

```hcl
resource "aws_route_table_association" "<rta3-name-here>" {
  subnet_id      = aws_subnet.<subnet3-name-here>.id
  route_table_id = aws_route_table.<rt2-name-here>.id
}
```
As before, we create the route table association resource then specify that our NAT gateway route table is going into our private subnet.

</details>

[\^top](#contents)

---

## 4 - Load Balancer {#4-ELB}

The load balancer (LB) will balance traffic between all targets in the target group. Because we're balancing traffic for specific port numbers we'll use an application laod balancer. Start by setting up `load-balancer.tf`:

```hcl
resource "aws_lb" "<lb-name-here>" {
  name               = "<lb-name-here>"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.<lbsg-name-here>.id]
  subnets            = [aws_subnet.<subnet1-name-here>.id, aws_subnet.<subnet2-name-here>.id]
  depends_on         = [aws_internet_gateway.<gw1-name-here>]
}
```

The security group that governs the LB has not been created yet (we'll handle security later). Note the subnets the LB is a member of and what it `depends_on` being created first.

The target group is set up to operate on port 80. I've left the protocol name out, let's see if you can figure out what four letter protocol runs on port 80.

```hcl
resource "aws_lb_target_group" "<tg-name-here>" {
  name     = "<tg-name-here>"
  port     = 80
  protocol = "<omitted-to-make-you-think>"
  vpc_id   = aws_vpc.<vpc-name-here>.id
}
```

And to listen for traffic coming in, the LB needs a listener. It'd be easy if I gave you the protocol here after asking for it earlier but we don't do things because they are easy. The listener needs some actions to do, in this case we want it to forward traffic from port 80 to the target group we defined earlier.

```hcl
resource "aws_lb_listener" "<lbl-name-here>" {
  load_balancer_arn = aws_lb.<lb-name-here>.arn
  port              = "80"
  protocol          = "<omitted-to-make-you-think>"
  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.<tg-name-here>.arn
  }
}
```
<details>
<summary>(only click here if you're really stuck)</summary>

Port 80 is used by the HTTP protocol. You should know this already if you're going to start learning cloud engineering.

</details>

[\^top](#contents)

---

## 5 - Auto Scaling {#5-ASG}

In this step we'll create an auto scaling group (ASG) to create the EC2 instances. To configure the EC2s we'll use a launch template.

### 5.1 - ASG & Launch Template

Starting with the ASG we give it a `min_size` to specify the number of EC2s for periods of low demand, `max_size` to give the maximum number of EC2s to provision, and the `desired_capacity` specifies the ideal number of EC2s to aim for. Be careful, as `max_size` can and will bankrupt you if you set it above your budget. This guide assumes you're using the free tier so we'll keep it low.

Create `asg-ec2.tf` and create the ASG:

```hcl
resource "aws_autoscaling_group" "<asg-name-here>" {
  max_size         = 1
  min_size         = 1
  desired_capacity = 1
  target_group_arns = [aws_lb_target_group.<tg-name-here>.arn]
  vpc_zone_identifier = [
    aws_subnet.<subnet3-name-here>.id
  ]
  launch_template {
    id      = aws_launch_template.<ltp-name-here>.id
    version = "$Latest"
  }
}
```

Note the reference to the target group we created earlier, the subnet we're putting the ASG in (private or public?), and the launch template. Let's create that launch template now:

```hcl
resource "aws_launch_template" "<ltp-name-here>" {
  name_prefix   = "<ltp-name-here>"
  image_id      = "<ami-id-here>"
  instance_type = "t2.micro"
  user_data     = filebase64("user_data.sh")

```
To keep it low cost/free we're using t2 micro EC2 instances. The `name_prefix` will be prepended to the instance name. For the `image_id` you need to use an Amazon Machine Image ID number from the region in which the EC2 instances will be deployed. For this project I'd recommend finding the AMI for Amazon Linux 2 in your region. We'll come back to the `user_data` later, but for now just know that this will be loaded when the EC2 instance is created.

The rest of the resource block contains the network interface information and a tag for organisation. The network interface has to be assigned to the subnet in which we're putting the EC2 instances (now is that a public or private subnet?) and the security group will be created later.

```hcl
  network_interfaces {
    associate_public_ip_address = false
    subnet_id                   = aws_subnet.<subnet3-name-here>.id
    security_groups             = [aws_security_group.<ec2sg-name-here>.id]
  }
  tag_specifications {
    resource_type = "instance"
    tags = {
      Name = "<instance-name>"
    }
  }
}
```

### 5.2 - User Data

The `user_data` referenced should be in the same directory as the `.tf` files. It's a bash script that will run on the EC2 instances when they're created. With this project we're just running a web server for demonstration purposes, so in production this user data will likely be more interesting.

Create `user_data.sh` and fill it with the following:

```bash
#!/bin/bash
sudo yum update -y
sudo yum install -y httpd
sudo systemctl start httpd
sudo systemctl enable httpd
echo "<h1>Ahoy there skipper, welcome to $(hostname -f)<h1>" > /var/www/html/index.html
```

If you're planning on becoming a cloud engineer then you should be able to recognise what this script does before starting this guide. But if you're really stuck...

<details>
<summary>(only click here if you're really stuck)</summary>

The first two lines update the system packages and install a web server.

- `yum` is the package manager used in Amazon Linux 2
- `yum update` upgrades the installed packages to the latest versions
- `yum install` installs a new software package
- `httpd` is how RHEL based distros refer to the Apache web server
- `-y` is a flag that automatically sends "yes" to yes/no prompts

The next two lines get the web server running.

- `systemctl` is used to control services
- `systemctl start` starts a service
- `systemctl enable` sets a service to start upon boot

And finally the last line sends some HTML code to the default site location for the web server software.

- `echo` prints a string to the standard output
- `$(hostname -f)` inserts the fqdn to the string
- `>` redirects where the string goes (from stdout to the index.html)

So all in all this bash script will update the EC2 instance's OS, install Apache, start Apache and set it to start on boot, then create a simple HTML document containing the fully qualified domain name of the EC2 instance.

</details>

[\^top](#contents)

---

## 6 - Security Groups {#6-SGs}

Finally, we need to allow traffic to pass through our load balancer. This is done by applying security groups to the various services.

### 6.1 - Security Group for ELB

Let's start by creating `security-groups.tf` and adding the SG for the load balancer:

```hcl
resource "aws_security_group" "<lbsg-name-here>" {
  name   = "<lbsg-name-here>"
  vpc_id = aws_vpc.<vpc-name-here>.id
```
Next we specify the ingress rules for permitting HTTP and HTTPS traffic. Note that the IP address range matches for CIDR block 0.0.0.0/0.

```hcl
  ingress {
    description      = "Allow HTTP requests from all sources"
    protocol         = "tcp"
    from_port        = 80
    to_port          = 80
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
```

As before, I've omitted the port number to get the noggin' joggin'. We specify the rule for a range of port numbers, but in this case we only want to target HTTPS traffic so the `from_port` and `to_port` will be the same value.

```
  ingress {
    description      = "Allow HTTPS requests from all sources"
    protocol         = "tcp"
    from_port        = <port-number-here>
    to_port          = <port-number-here>
    cidr_blocks      = ["0.0.0.0/0"]
    ipv6_cidr_blocks = ["::/0"]
  }
```

<details>
<summary>(only click here if you're really stuck)</summary>

Port 443 is used for HTTPS.

</details>

And to close the block off add the egress rule. As we wish to permit all traffic out, the rule's port range is 0 to 0. and the protocol is set to `-1` - this is semantically equivalent to _all_ protocols.


```hcl
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

}
```

### 6.2 - Security Group for EC2

In the same file (or in a separate file, I've grouped them together for convenience) add another security group resource block for the EC2 instances.

```hcl
resource "aws_security_group" "<ec2sg-name-here>" {
  name   = "<ec2sg-name-here>"
  vpc_id = aws_vpc.<vpc-name-here>.id
```

As you'll have noticed this block starts the same as the previous block for the ELB. The ingress and egress rules are going to be almost entirely the same, but with the ingress rule having a list of security groups provided as a source of traffic. This list consists of one entry: the ELB security group we made in the previous section.

```hcl
  ingress {
    description     = "Allow HTTP requests from the Load Balancer"
    protocol        = "tcp"
    from_port       = 80
    to_port         = 80
    security_groups = [aws_security_group.<lbsg-name-here>.id]
  }
```

And finally, to allow traffic out we add an egress block as before. Specify all protocols and match all IP addresses. You know what to do. And if not...


<details>
<summary>(only click here if you're really stuck)</summary>

```
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
```

As `protocol` is set to `-1`, the `from_port` and `to_port` must be set to 0.

</details>

[\^top](#contents)

---

## 7 - Deploying & Troubleshooting {#7-troubleshooting}

### 7.1 - Terraform

And now our project is ready to roll. Run `terraform init` to install the required providers if you haven't already, then run `terraform plan` to check the changes to be made. If everything has been configured correctly there should be no error messages. From here we have a few options.

- `terraform fmt` will format the files to the Terraform standards (whilst the code _will_ run if only syntactically correct, we do not work in isolation and others must be able to read our code - for this reason we should adhere to the Terraform formatting)
- `terraform validate` will check the syntax of the configuration files in the project
- `terraform apply` means it's Go Time: Terraform will convert the configuration files to API requests and build our infrastructures on the AWS cloud


### 7.2 - Testing

First, let's recap by taking an overview of what this project does.
From server to client:
-  our EC2 instance(s) is(/are) deploying with Apache and a simple web page that has the fully qualified domain name of the instance upon which it is running
- to reach the web page, user's requests go through the load balancer
- the load balancer selects an EC2 instance from the target group to which the traffic shall be passed
- the user traffic reaches the load balancer from its DNS name, which can be found via the AWS console or CLI (`aws elb describe-load-balancers --load-balancer-name <elb-name-here>`)
- depending on workload, the ASG will create more or fewer EC2 instances to populate the target group the through which the load balancer will share traffic

In effect, this means we can see which EC2 instance we've reached when we reload the page. 

### 7.3 - Troubleshooting

There's rarely a one-size-fits-all way to write troubleshooting for projects, so here are some general tips that helped me.

- Each piece of this project has a place it fits and other pieces it interacts with. Make sure your pieces are in the right place and are connected to the right parts.
- Understanding what each part does is a massive help. Even if it's just a vague overview it makes it so that you're not blindly trying to figure out what's going wrong with something totally alien.
- Check your syntax, check your formatting, check spelling mistakes and typos, check IP addresses and other numerical values are correct.
- Don't think of error messages as locks for which you need to find a key. Error messages are puzzles with hints on how to solve them. Try to understand what error messages mean, combine this with understanding what each piece connects to and how it works to make troubleshooting a breeze.
- Take things slowly and think through the code as though one step at a time. What is each part doing and why?
- When in doubt, Google search your query in simple terms. If you don't understand that, ask others for help via forums or online chat groups. Learn how to ask a Really Good Question. 

[\^top](#contents)
