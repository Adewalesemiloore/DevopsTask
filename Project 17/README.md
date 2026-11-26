<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/Adewalesemiloore/DevopsTask">
    <img src="Images/Project logo.png" alt="Project Logo" width="80" height="80">
  </a>

<h3 align="center">AUTOMATE INFRASTRUCTURE WITH IAC USING TERRAFORM — PART 2</h3>
  <p><b style="color: #fb6900">Maintainer:</b> <kbd>Adewale Oluwasemiloore</kbd> &nbsp; <b style="color: #fb6900">Date:</b> <kbd>November 26, 2026</kbd></p>
</div>

<p align="center">
  <img src="https://img.shields.io/badge/Amazon_AWS-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/Terraform-7B42BC?style=for-the-badge&logo=terraform&logoColor=white" />
  <img src="https://img.shields.io/badge/Amazon_RDS-527FFF?style=for-the-badge&logo=amazonrds&logoColor=white" />
  <img src="https://img.shields.io/badge/Route_53-8C4FFF?style=for-the-badge&logo=amazonroute53&logoColor=white" />
  <img src="https://img.shields.io/badge/IAM-DD344C?style=for-the-badge&logo=amazonaws&logoColor=white" />
  <img src="https://img.shields.io/badge/HCL-7B42BC?style=for-the-badge&logo=hashicorp&logoColor=white" />
  <img src="https://img.shields.io/badge/IaC-informational?style=for-the-badge" />
</p>

---

## Project Overview

In Project 16 I used Terraform to create a VPC and public subnets. In this part I carry on and automate the rest of the architecture I built manually in Project 15: private subnets, internet and NAT gateways, route tables, IAM roles, security groups, an ACM certificate, external and internal load balancers, launch templates and Auto Scaling Groups, EFS with a KMS key, and a Multi-AZ-ready MySQL RDS instance.

Along the way I follow a few best practices: use variables and `length()` instead of hard-coded counts, use `cidrsubnet()` for CIDR allocation, keep variables and resources in separate files, and **tag every resource** using `merge()` and `format()`.

A good grasp of networking (IP addresses, subnets, CIDR notation, routing, internet and NAT gateways) makes this project much easier, so it is worth revising those first.

---

## Table of Contents

- [Part 1 — Tagging and Variables](#part-1--tagging-and-variables)
- [Part 2 — Networking: Internet Gateway, NAT Gateway and Routes](#part-2--networking-internet-gateway-nat-gateway-and-routes)
- [Part 3 — IAM Roles and Policies](#part-3--iam-roles-and-policies)
- [Part 4 — Security Groups](#part-4--security-groups)
- [Part 5 — ACM Certificate and DNS Records](#part-5--acm-certificate-and-dns-records)
- [Part 6 — Load Balancers and Target Groups](#part-6--load-balancers-and-target-groups)
- [Part 7 — Launch Templates and Auto Scaling Groups](#part-7--launch-templates-and-auto-scaling-groups)
- [Part 8 — Storage and Database: EFS, KMS and RDS](#part-8--storage-and-database-efs-kms-and-rds)
- [Part 9 — Variables and tfvars](#part-9--variables-and-tfvars)
- [Part 10 — Plan, Apply and Explore](#part-10--plan-apply-and-explore)
- [Part 11 — Destroy](#part-11--destroy)
- [Errors & Fixes](#errors--fixes)

---

## Part 1 — Tagging and Variables

Tags organise resources into virtual groups, make them searchable and filterable, allow billing reports by department or environment, help spot unused resources, and show ownership when several teams share an account.

### 1.1 — Default tags in `terraform.tfvars`

Define the default tag set once:

```hcl
tags = {
  Enviroment      = "production" 
  Owner-Email     = "owner@example.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

![Default tags in terraform.tfvars](./Images/Task%2017.1.png)

### 1.2 — Apply tags with `merge()`

Every resource then merges the default tags with its own `Name`:

```hcl
tags = merge(
    var.tags,
    {
      Name = "Name of the resource"
    },
  )
```

![Using merge() to tag a resource](./Images/Task%2017.2.png)

### 1.3 — Declare the `tags` variable

```hcl
variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}
```

![The tags variable in variables.tf](./Images/Task%2017.3.png)

Any later change to the tags is made in one place, `terraform.tfvars`. The key-value pairs are still literal values, so the next improvement is to give each its own variable and use `var.<name>`. I applied the same idea to every resource that follows.

---

## Part 2 — Networking: Internet Gateway, NAT Gateway and Routes

### 2.1 — Internet Gateway and `format()`

In `internet_gateway.tf`:

```hcl
resource "aws_internet_gateway" "ig" {
  vpc_id = aws_vpc.main.id

  tags = merge(
    var.tags,
    {
      Name = format("%s-%s!", aws_vpc.main.id,"IG")
    } 
  )
}
```

`format()` builds a unique name: the first `%s` takes the VPC id, the second appends the literal `IG`.

When resources are created in a loop with `count`, each needs a unique `Name` tag. With `format()` and `count.index` each private subnet gets a distinct name, such as:

```
Name = PrivateSubnet-0
Name = PrivateSubnet-1
Name = PrivateSubnet-2
```

```hcl
tags = merge(
  var.tags,
  {
    Name = format("PrivateSubnet-%s", count.index)
  } 
)
```

### 2.2 — NAT Gateway and Elastic IP

In `natgateway.tf`. The `depends_on` makes explicit that the Internet Gateway must exist before the EIP and NAT Gateway are created.

```hcl
resource "aws_eip" "nat_eip" {
  vpc        = true
  depends_on = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-EIP", var.name)
    },
  )
}

resource "aws_nat_gateway" "nat" {
  allocation_id = aws_eip.nat_eip.id
  subnet_id     = element(aws_subnet.public.*.id, 0)
  depends_on    = [aws_internet_gateway.ig]

  tags = merge(
    var.tags,
    {
      Name = format("%s-Nat", var.name)
    },
  )
}
```

![NAT Gateway and Elastic IP code](./Images/Task%2017.4.png)

### 2.3 — Route tables

In `route_tables.tf` I created the private and public route tables, the public route to the Internet Gateway, and the subnet associations:

```hcl
# create private route table
resource "aws_route_table" "private-rtb" {
  vpc_id = aws_vpc.main.id


  tags = merge(
    var.tags,
    {
      Name = format("%s-Private-Route-Table", var.name)
    },
  )
}

# associate all private subnets to the private route table
resource "aws_route_table_association" "private-subnets-assoc" {
  count          = length(aws_subnet.private[*].id)
  subnet_id      = element(aws_subnet.private[*].id, count.index)
  route_table_id = aws_route_table.private-rtb.id
}


# create route table for the public subnets
resource "aws_route_table" "public-rtb" {
  vpc_id = aws_vpc.main.id


  tags = merge(
    var.tags,
    {
      Name = format("%s-Public-Route-Table", var.name)
    },
  )
}


# create route for the public route table and attach the internet gateway
resource "aws_route" "public-rtb-route" {
  route_table_id         = aws_route_table.public-rtb.id
  destination_cidr_block = "0.0.0.0/0"
  gateway_id             = aws_internet_gateway.ig.id
}


# associate all public subnets to the public route table
resource "aws_route_table_association" "public-subnets-assoc" {
  count          = length(aws_subnet.public[*].id)
  subnet_id      = element(aws_subnet.public[*].id, count.index)
  route_table_id = aws_route_table.public-rtb.id
}
```

This adds the VPC, 2 public subnets, 4 private subnets, 1 Internet Gateway, 1 NAT Gateway, 1 EIP and 2 route tables to AWS in a multi-AZ setup.

### 2.4 — Validate, find the gaps, and apply

`terraform fmt` and `terraform validate` reported undeclared resources (details in [Errors & Fixes](#errors--fixes)).

![terraform validate errors](./Images/Task%2017.5.png)
![subnets.tf with public and private subnets](./Images/Task%2017.6.png)

After declaring the missing resources and variables, the configuration validated, and a plan and apply created the networking.

![Subnet and availability zone variables](./Images/Task%2017.7.png)
![terraform.tfvars with subnet CIDRs and tags](./Images/Task%2017.8.png)
![terraform validate success](./Images/Task%2017.9.png)
![terraform plan](./Images/Task%2017.10.png)
![Plan summary: 14 to add, 1 to change, 2 to destroy](./Images/Task%2017.11.png)
![terraform apply](./Images/Task%2017.12.png)
![Apply complete](./Images/Task%2017.13.png)

---

## Part 3 — IAM Roles and Policies

EC2 instances need a role to access specific AWS resources. I built this in `roles.tf` in four steps.

### 3.1 — AssumeRole

AssumeRole uses the Security Token Service (STS) to return temporary credentials (access key ID, secret key and session token). Here it lets EC2 assume the role.

```hcl
resource "aws_iam_role" "ec2_instance_role" {
name = "ec2_instance_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Sid    = ""
        Principal = {
          Service = "ec2.amazonaws.com"
        }
      },
    ]
  })

  tags = merge(
    var.tags,
    {
      Name = "aws assume role"
    },
  )
}
```

### 3.2 — IAM policy

The policy defines the permissions. For example, allowing `ec2:Describe*`:

```hcl
resource "aws_iam_policy" "policy" {
  name        = "ec2_instance_policy"
  description = "A test policy"
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = [
          "ec2:Describe*",
        ]
        Effect   = "Allow"
        Resource = "*"
      },
    ]

  })

  tags = merge(
    var.tags,
    {
      Name =  "aws assume policy"
    },
  )

}
```

### 3.3 — Attach the policy to the role

```hcl
resource "aws_iam_role_policy_attachment" "test-attach" {
    role       = aws_iam_role.ec2_instance_role.name
    policy_arn = aws_iam_policy.policy.arn
}
```

### 3.4 — Instance profile

```hcl
resource "aws_iam_instance_profile" "ip" {
    name = "aws_instance_profile_test"
    role =  aws_iam_role.ec2_instance_role.name
}
```

---

## Part 4 — Security Groups

All security groups live in a single `security.tf`, and each resource references the group it needs. Rules that refer to another security group use `aws_security_group_rule`.

The groups cover: external ALB, bastion, Nginx, internal ALB, webservers and the data layer.

```hcl
# security group for alb, to allow acess from any where for HTTP and HTTPS traffic
resource "aws_security_group" "ext-alb-sg" {
  name        = "ext-alb-sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow TLS inbound traffic"


  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  ingress {
    description = "HTTPS"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = merge(
    var.tags,
    {
      Name = "ext-alb-sg"
    },
  )


}


# security group for bastion, to allow access into the bastion host from you IP
resource "aws_security_group" "bastion_sg" {
  name        = "vpc_web_sg"
  vpc_id      = aws_vpc.main.id
  description = "Allow incoming HTTP connections."


  ingress {
    description = "SSH"
    from_port   = 22
    to_port     = 22
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = merge(
    var.tags,
    {
      Name = "Bastion-SG"
    },
  )
}


#security group for nginx reverse proxy, to allow access only from the extaernal load balancer and bastion instance
resource "aws_security_group" "nginx-sg" {
  name   = "nginx-sg"
  vpc_id = aws_vpc.main.id


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = merge(
    var.tags,
    {
      Name = "nginx-SG"
    },
  )
}


resource "aws_security_group_rule" "inbound-nginx-http" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.ext-alb-sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}


resource "aws_security_group_rule" "inbound-bastion-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.nginx-sg.id
}


# security group for ialb, to have acces only from nginx reverser proxy server
resource "aws_security_group" "int-alb-sg" {
  name   = "my-alb-sg"
  vpc_id = aws_vpc.main.id


  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }


  tags = merge(
    var.tags,
    {
      Name = "int-alb-sg"
    },
  )


}


resource "aws_security_group_rule" "inbound-ialb-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.nginx-sg.id
  security_group_id        = aws_security_group.int-alb-sg.id
}


# security group for webservers, to have access only from the internal load balancer and bastion instance
resource "aws_security_group" "webserver_sg" {
  name   = "webserver-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "webserver-sg"
    },
  )
}


resource "aws_security_group_rule" "inbound-web-https" {
  type                     = "ingress"
  from_port                = 443
  to_port                  = 443
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.int-alb-sg.id
  security_group_id        = aws_security_group.webserver_sg.id
}


resource "aws_security_group_rule" "inbound-web-ssh" {
  type                     = "ingress"
  from_port                = 22
  to_port                  = 22
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.webserver_sg.id
}


# security group for datalayer to allow traffic from webserver on nfs and mysql port and bastion host on mysql port
resource "aws_security_group" "datalayer_sg" {
  name   = "datalayer-sg"
  vpc_id = aws_vpc.main.id

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = merge(
    var.tags,
    {
      Name = "datalayer-sg"
    },
  )
}


resource "aws_security_group_rule" "inbound-nfs-port" {
  type                     = "ingress"
  from_port                = 2049
  to_port                  = 2049
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver_sg.id
  security_group_id        = aws_security_group.datalayer_sg.id
}


resource "aws_security_group_rule" "inbound-mysql-bastion" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion_sg.id
  security_group_id        = aws_security_group.datalayer_sg.id
}


resource "aws_security_group_rule" "inbound-mysql-webserver" {
  type                     = "ingress"
  from_port                = 3306
  to_port                  = 3306
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver_sg.id
  security_group_id        = aws_security_group.datalayer_sg.id
}
```

---

## Part 5 — ACM Certificate and DNS Records

In `cert.tf` I request a wildcard certificate, validate it through Route 53 DNS records, and create alias records for the Tooling and WordPress hostnames. Replace the domain with your own.

![Hosted zone](./Images/Task%2017.14.png)

```hcl
resource "aws_acm_certificate" "company" {
  domain_name       = "*.example.com"
  validation_method = "DNS"
}


# calling the hosted zone
data "aws_route53_zone" "company" {
  name         = "example.com"
  private_zone = false
}


# selecting validation method
resource "aws_route53_record" "company" {
  for_each = {
    for dvo in aws_acm_certificate.company.domain_validation_options : dvo.domain_name => {
      name   = dvo.resource_record_name
      record = dvo.resource_record_value
      type   = dvo.resource_record_type
    }
  }


  allow_overwrite = true
  name            = each.value.name
  records         = [each.value.record]
  ttl             = 60
  type            = each.value.type
  zone_id         = data.aws_route53_zone.company.zone_id
}


# validate the certificate through DNS method
resource "aws_acm_certificate_validation" "company" {
  certificate_arn         = aws_acm_certificate.company.arn
  validation_record_fqdns = [for record in aws_route53_record.company : record.fqdn]
}


# create records for tooling
resource "aws_route53_record" "tooling" {
  zone_id = data.aws_route53_zone.company.zone_id
  name    = "tooling.example.com"
  type    = "A"


  alias {
    name                   = aws_lb.ext-alb.dns_name
    zone_id                = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}


# create records for wordpress
resource "aws_route53_record" "wordpress" {
  zone_id = data.aws_route53_zone.company.zone_id
  name    = "wordpress.example.com"
  type    = "A"


  alias {
    name                   = aws_lb.ext-alb.dns_name
    zone_id                = aws_lb.ext-alb.zone_id
    evaluate_target_health = true
  }
}
```

---

## Part 6 — Load Balancers and Target Groups

All in `alb.tf`: first the ALB, then target groups, then listeners.

### 6.1 — External (internet-facing) ALB

```hcl
resource "aws_lb" "ext_alb" {
  name     = "ext_alb"
  internal = false
  security_groups = [
    aws_security_group.ext_alb_sg.id,
  ]


  subnets = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]


  tags = merge(
    var.tags,
    {
      Name = "ACS_ext_alb"
    },
  )


  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}

resource "aws_lb_target_group" "nginx_tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
  name        = "nginx-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}

resource "aws_lb_listener" "nginx_listner" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.company.certificate_arn


  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx_tgt.arn
  }
}

# ----------------------------
#Internal Load Balancers for webservers
#---------------------------------


resource "aws_lb" "ialb" {
  name     = "ialb"
  internal = true
  security_groups = [
    aws_security_group.int_alb_sg.id,
  ]


  subnets = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]


  tags = merge(
    var.tags,
    {
      Name = "ACS-int-alb"
    },
  )


  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}

# --- target group  for wordpress -------


resource "aws_lb_target_group" "wordpress_tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }


  name        = "wordpress-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}


# --- target group for tooling -------


resource "aws_lb_target_group" "tooling_tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }


  name        = "tooling-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}

# For this aspect a single listener was created for the wordpress which is default,
# A rule was created to route traffic to tooling when the host header changes


resource "aws_lb_listener" "web_listener" {
  load_balancer_arn = aws_lb.ialb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.company.certificate_arn


  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.wordpress_tgt.arn
  }
}


# listener rule for tooling target


resource "aws_lb_listener_rule" "tooling_listener" {
  listener_arn = aws_lb_listener.web_listener.arn
  priority     = 99


  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tooling_tgt.arn
  }


  condition {
    host_header {
      values = ["tooling.example.com"]
    }
  }
}
```

Target group for Nginx:

```hcl
resource "aws_lb_target_group" "nginx-tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }
  name        = "nginx-tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```

Listener:

```hcl
resource "aws_lb_listener" "nginx-listner" {
  load_balancer_arn = aws_lb.ext-alb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.oyindamola.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.nginx-tgt.arn
  }
}
```

Outputs in `output.tf`:

```hcl
output "alb_dns_name" {
  value = aws_lb.ext-alb.dns_name
}

output "alb_target_group_arn" {
  value = aws_lb_target_group.nginx-tgt.arn
}
```

### 6.2 — Internal ALB

Same concept as the external ALB, added to `alb.tf`:

```hcl
# ----------------------------
#Internal Load Balancers for webservers
#---------------------------------

resource "aws_lb" "ialb" {
  name     = "ialb"
  internal = true
  security_groups = [
    aws_security_group.int_alb_sg.id,
  ]

  subnets = [
    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]

  tags = merge(
    var.tags,
    {
      Name = "ACS-int-alb"
    },
  )

  ip_address_type    = "ipv4"
  load_balancer_type = "application"
}
```

Target groups for WordPress and Tooling:

```hcl
# --- target group  for wordpress -------

resource "aws_lb_target_group" "wordpress_tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }

  name        = "wordpress_tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}

# --- target group for tooling -------

resource "aws_lb_target_group" "tooling_tgt" {
  health_check {
    interval            = 10
    path                = "/healthstatus"
    protocol            = "HTTPS"
    timeout             = 5
    healthy_threshold   = 5
    unhealthy_threshold = 2
  }

  name        = "tooling_tgt"
  port        = 443
  protocol    = "HTTPS"
  target_type = "instance"
  vpc_id      = aws_vpc.main.id
}
```

Listener with a host-header rule to route Tooling traffic:

```hcl
# For this aspect a single listener was created for the wordpress which is default,
# A rule was created to route traffic to tooling when the host header changes

resource "aws_lb_listener" "web_listener" {
  load_balancer_arn = aws_lb.ialb.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate_validation.oyindamola.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.wordpress_tgt.arn
  }
}

# listener rule for tooling target

resource "aws_lb_listener_rule" "tooling_listener" {
  listener_arn = aws_lb_listener.web-listener.arn
  priority     = 99

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.tooling_tgt.arn
  }

  condition {
    host_header {
      values = ["tooling.oyindamola.gq"]
    }
  }
}
```

---

## Part 7 — Launch Templates and Auto Scaling Groups

The ASGs scale the EC2 instances out and in with traffic. For now the launch templates use a generic AMI. A custom AMI is built later with Packer in Project 19.

Two files are used: `asg-bastion-nginx.tf` (Bastion and Nginx) and `asg-wordpress-tooling.tf` (WordPress and Tooling).

### 7.1 — SNS topic and notifications

```hcl
#### creating sns topic for all the auto scaling groups
resource "aws_sns_topic" "main_sns" {
name = "Default_CloudWatch_Alarms_Topic"
}
```

### 7.2 — Bastion and Nginx

```hcl
#### creating sns topic for all the auto scaling groups
resource "aws_sns_topic" "main_sns" {
  name = "Default_CloudWatch_Alarms_Topic"
}

# creating notification for all the auto scaling groups
resource "aws_autoscaling_notification" "main_notifications" {
  group_names = [
    aws_autoscaling_group.bastion_asg.name,
    aws_autoscaling_group.nginx_asg.name,
    aws_autoscaling_group.wordpress_asg.name,
    aws_autoscaling_group.tooling_asg.name,
  ]

  notifications = [
    "autoscaling:EC2_INSTANCE_LAUNCH",
    "autoscaling:EC2_INSTANCE_TERMINATE",
    "autoscaling:EC2_INSTANCE_LAUNCH_ERROR",
    "autoscaling:EC2_INSTANCE_TERMINATE_ERROR",
  ]

  topic_arn = aws_sns_topic.main_sns.arn
}
# launch template for bastion
resource "random_shuffle" "az_list" {
  input = data.aws_availability_zones.available.names
}


resource "aws_launch_template" "bastion_launch_template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.bastion_sg.id]


  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }


  key_name = var.keypair


  placement {
    availability_zone = "random_shuffle.az_list.result"
  }


  lifecycle {
    create_before_destroy = true
  }


  tag_specifications {
    resource_type = "instance"


    tags = merge(
      var.tags,
      {
        Name = "bastion-launch-template"
      },
    )
  }


  user_data = filebase64("${path.module}/bastion.sh")
}


# ---- Autoscaling for bastion  hosts


resource "aws_autoscaling_group" "bastion_asg" {
  name                      = "bastion-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1


  vpc_zone_identifier = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]


  launch_template {
    id      = aws_launch_template.bastion_launch_template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "bastion-launch-template"
    propagate_at_launch = true
  }


}


# launch template for nginx


resource "aws_launch_template" "nginx_launch_template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.nginx-sg.id]


  iam_instance_profile {
    name = aws_iam_instance_profile.ip.id
  }


  key_name = var.keypair


  placement {
    availability_zone = "random_shuffle.az_list.result"
  }


  lifecycle {
    create_before_destroy = true
  }


  tag_specifications {
    resource_type = "instance"


    tags = merge(
      var.tags,
      {
        Name = "nginx-launch-template"
      },
    )
  }


  user_data = filebase64("${path.module}/nginx.sh")
}


# ------ Autoscslaling group for reverse proxy nginx ---------


resource "aws_autoscaling_group" "nginx_asg" {
  name                      = "nginx-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1


  vpc_zone_identifier = [
    aws_subnet.public[0].id,
    aws_subnet.public[1].id
  ]


  launch_template {
    id      = aws_launch_template.nginx_launch_template.id
    version = "$Latest"
  }


  tag {
    key                 = "Name"
    value               = "nginx-launch-template"
    propagate_at_launch = true
  }


}


# attaching autoscaling group of nginx to external load balancer
resource "aws_autoscaling_attachment" "asg_attachment_nginx" {
  autoscaling_group_name = aws_autoscaling_group.nginx_asg.id
  lb_target_group_arn    = aws_lb_target_group.nginx_tgt.arn
}
```

### 7.3 — WordPress and Tooling

```hcl
# launch template for wordpress


resource "aws_launch_template" "wordpress_launch_template" {
  image_id      = var.ami
  instance_type = "t2.micro"

  vpc_security_group_ids = [aws_security_group.webserver_sg.id]
  key_name               = var.keypair

  iam_instance_profile {
    name = aws_iam_instance_profile.ip.name
  }

  placement {
    availability_zone = random_shuffle.az_list.result[0]
  }

  lifecycle {
    create_before_destroy = true
  }

  user_data = filebase64("${path.module}/wordpress.sh")

  tag_specifications {
    resource_type = "instance"

    tags = merge(
      var.tags,
      {
        Name = "wordpress-launch-template"
      },
    )
  }
}


# ---- Autoscaling for wordpress application


resource "aws_autoscaling_group" "wordpress_asg" {
  name                      = "wordpress-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1
  vpc_zone_identifier = [


    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]


  launch_template {
    id      = aws_launch_template.wordpress_launch_template.id
    version = "$Latest"
  }
  tag {
    key                 = "Name"
    value               = "wordpress-asg"
    propagate_at_launch = true
  }
}


# attaching autoscaling group of  wordpress application to internal loadbalancer
resource "aws_autoscaling_attachment" "asg_attachment_tooling" {
  autoscaling_group_name = aws_autoscaling_group.tooling_asg.id
  lb_target_group_arn    = aws_lb_target_group.tooling_tgt.arn
}


# launch template for toooling
resource "aws_launch_template" "tooling_launch_template" {
  image_id               = var.ami
  instance_type          = "t2.micro"
  vpc_security_group_ids = [aws_security_group.webserver_sg.id]


  iam_instance_profile {
    name = aws_iam_instance_profile.ip.name
  }


  key_name = var.keypair


  placement {
    availability_zone = "random_shuffle.az_list.result"
  }


  lifecycle {
    create_before_destroy = true
  }


  tag_specifications {
    resource_type = "instance"


    tags = merge(
      var.tags,
      {
        Name = "tooling-launch-template"
      },
    )


  }


  user_data = filebase64("${path.module}/tooling.sh")
}


# ---- Autoscaling for tooling -----


resource "aws_autoscaling_group" "tooling_asg" {
  name                      = "tooling-asg"
  max_size                  = 2
  min_size                  = 1
  health_check_grace_period = 300
  health_check_type         = "ELB"
  desired_capacity          = 1


  vpc_zone_identifier = [


    aws_subnet.private[0].id,
    aws_subnet.private[1].id
  ]


  launch_template {
    id      = aws_launch_template.tooling_launch_template.id
    version = "$Latest"
  }


  tag {
    key                 = "Name"
    value               = "tooling-launch-template"
    propagate_at_launch = true
  }
}
# attaching autoscaling group of  tooling application to internal loadbalancer
resource "aws_autoscaling_attachment" "asg_attachment_wordpress" {
  autoscaling_group_name = aws_autoscaling_group.wordpress_asg.id
  lb_target_group_arn    = aws_lb_target_group.wordpress_tgt.arn
}
```

---

## Part 8 — Storage and Database: EFS, KMS and RDS

### 8.1 — KMS key and EFS

An EFS file system needs a KMS key. In `efs.tf`:

```hcl
resource "aws_kms_key" "company_kms" {
  description             = "KMS key"
  deletion_window_in_days = 10
  enable_key_rotation     = true
}

# create key alias
resource "aws_kms_alias" "alias" {
  name          = "alias/kms"
  target_key_id = aws_kms_key.company_kms.key_id
}

# create Elastic file system
resource "aws_efs_file_system" "company_efs" {
  encrypted  = true
  kms_key_id = aws_kms_key.company_kms.arn

  tags = merge(
    var.tags,
    {
      Name = "company-efs"
    },
  )
}

# set first mount target for the EFS
resource "aws_efs_mount_target" "subnet_1" {
  file_system_id  = aws_efs_file_system.company_efs.id
  subnet_id       = aws_subnet.private[0].id
  security_groups = [aws_security_group.datalayer_sg.id]
}

# set second mount target for the EFS
resource "aws_efs_mount_target" "subnet_2" {
  file_system_id  = aws_efs_file_system.company_efs.id
  subnet_id       = aws_subnet.private[1].id
  security_groups = [aws_security_group.datalayer_sg.id]
}

# create access point for wordpress
resource "aws_efs_access_point" "wordpress" {
  file_system_id = aws_efs_file_system.company_efs.id

  posix_user {
    gid = 0
    uid = 0
  }

  root_directory {
    path = "/wordpress"

    creation_info {
      owner_gid   = 0
      owner_uid   = 0
      permissions = "0755"
    }
  }

  tags = merge(
    var.tags,
    {
      Name = "wordpress-access-point"
    },
  )
}

# create access point for tooling
resource "aws_efs_access_point" "tooling" {
  file_system_id = aws_efs_file_system.company_efs.id

  posix_user {
    gid = 0
    uid = 0
  }

  root_directory {
    path = "/tooling"

    creation_info {
      owner_gid   = 0
      owner_uid   = 0
      permissions = "0755"
    }
  }

  tags = merge(
    var.tags,
    {
      Name = "tooling-access-point"
    },
  )
}
```

### 8.2 — MySQL RDS

In `rds.tf`:

```hcl
# This section will create the subnet group for the RDS instance using the private subnets
resource "aws_db_subnet_group" "company-rds" {
  name       = "acs-rds"
  subnet_ids = aws_subnet.private[*].id

  tags = merge(
    var.tags,
    {
      Name = "company-rds"
    },
  )
}

# create the RDS instance with the subnet group
resource "aws_db_instance" "company-rds" {
  allocated_storage      = 20
  storage_type           = "gp2"
  engine                 = "mysql"
  engine_version         = "8.0.45"
  instance_class         = "db.t3.micro"
  db_name                = "companydb"
  username               = var.master_username
  password               = var.master_password
  parameter_group_name   = "default.mysql8.0"
  db_subnet_group_name   = aws_db_subnet_group.company-rds.name
  skip_final_snapshot    = true
  vpc_security_group_ids = [aws_security_group.datalayer_sg.id]
  multi_az               = false
}
```

---

## Part 9 — Variables and tfvars

The resources above reference many variables, so I declared them all in `variables.tf`:

```hcl
variable "region" {
  default = "us-east-1"
}


variable "vpc_cidr" {
  default = "172.16.0.0/16"
}



variable "enable_dns_support" {
  default = true
}


variable "enable_dns_hostnames" {
  default = true
}


variable "preferred_number_of_public_subnets" {
  default = 2
}


variable "public_subnet_cidrs" {
  type = list(string)
}

variable "private_subnet_cidrs" {
  type = list(string)
}

variable "availability_zones" {
  type = list(string)
}

variable "name" {
  type    = string
  default = "ACS"
}


variable "tags" {
  description = "A mapping of tags to assign to all resources."
  type        = map(string)
  default     = {}
}

variable "ami" {
  type        = string
  description = "AMI ID for the launch template"
}


variable "keypair" {
  type        = string
  description = "key pair for the instances"
}


variable "account_no" {
  type        = number
  description = "the account number"
}


variable "master_username" {
  type        = string
  description = "RDS admin username"
}


variable "master_password" {
  type        = string
  description = "RDS master password"
}
```

Then supplied values in `terraform.tfvars`:

```hcl
region = "us-east-1"

vpc_cidr = "172.16.0.0/16"

enable_dns_support = "true"

enable_dns_hostnames = "true"

enable_classiclink = "false"

enable_classiclink_dns_support = "false"

preferred_number_of_public_subnets = "2"

preferred_number_of_private_subnets = "4"

environment = "production"

ami = "ami-0b0af3577fe5e3532"

keypair = "devops"

# Ensure to change this to your acccount number
account_no = "<your-account-number>"

master_username = "<db-username>"

master_password = "<db-password>"

tags = {
  Enviroment      = "production" 
  Owner-Email     = "owner@example.com"
  Managed-By      = "Terraform"
  Billing-Account = "1234567890"
}
```

> **Note:** The variable names in `terraform.tfvars` must match the declarations in `variables.tf`. The RDS credentials are declared as `master_username` and `master_password`, so those are the names used above. Never commit real passwords, and keep real account numbers and emails out of public repos.

Two things to keep in mind before applying:

- The long list of files looks confusing, but it is fine for a start. Project 18 tidies it up using **modules**.
- The application will not fully work yet, because the user-data scripts need the RDS and EFS endpoints that do not exist when the launch templates are created. Project 19 fixes this using Ansible.

---

## Part 10 — Plan, Apply and Explore

Re-initialise with the upgraded providers, then plan and apply:

```bash
terraform init -upgrade
terraform plan
terraform apply
```

![terraform init -upgrade](./Images/Task%2017.16.png)
![terraform apply confirmation and resources being created](./Images/Task%2017.17.png)
![Apply in progress](./Images/Task%2017.18.png)
![Load balancers, certificate validation and ASG creating](./Images/Task%2017.19.png)
![Apply complete and outputs](./Images/Task%2017.20.png)

Exploring the created resources in the AWS console:

![VPC created by Terraform](./Images/Task%2017.21.png)
![Subnets](./Images/Task%2017.22.png)
![Route tables and gateways](./Images/Task%2017.23.png)
![Security groups](./Images/Task%2017.24.png)
![Load balancers](./Images/Task%2017.25.png)
![Target groups](./Images/Task%2017.26.png)
![Auto Scaling Groups](./Images/Task%2017.27.png)
![RDS, EFS and other resources](./Images/Task%2017.28.png)

---

## Part 11 — Destroy

These resources cost money, so I destroyed them straight away.

```bash
terraform destroy
```

![terraform destroy plan](./Images/Task%2017.29.png)
![Destroy complete](./Images/Task%2017.30.png)

---

## Additional Tasks

- Summarise networking concepts: IP address, subnets, CIDR notation, IP routing, Internet Gateway and NAT.
- Summarise the OSI model and the TCP/IP suite and how they relate.
- Explain the difference between an assume role policy and a role policy.

---

## Errors & Fixes

### `terraform validate`: reference to undeclared resource

**Symptom:** After adding `natgateway.tf` and `route_tables.tf`, `terraform fmt` followed by `terraform validate` failed with `Reference to undeclared resource` errors: `aws_internet_gateway.ig` and `aws_subnet.private` "has not been declared in the root module".

**Root cause:** The new files referenced resources that Terraform could not find yet. The private subnets were not declared at all (only the public subnets existed from Project 16), and the Internet Gateway resource was not yet declared in the configuration when validation ran.

**Fix:** Declared the Internet Gateway in `internet_gateway.tf`, and added a `subnets.tf` that declares both the private and public subnets with `count = length(var.<cidr list>)`, `format()` tags and CIDRs/AZs from variables. I declared the matching list variables in `variables.tf` and set their values in `terraform.tfvars`:

```hcl
public_subnet_cidrs  = ["172.16.1.0/24", "172.16.2.0/24"]
private_subnet_cidrs = ["172.16.3.0/24", "172.16.4.0/24"]
availability_zones   = ["us-east-1a", "us-east-1b"]
```

`terraform fmt` and `terraform validate` then reported `Success! The configuration is valid.`

---

*Project completed by **Adewale Oluwasemiloore***
