+++
categories = ["docker", "aws"]
date = "2018-02-07T20:10:02-05:00"
description = "How to migrate Docker for AWS from a managed VPC to a pre-existing VPC"
keywords = ["docker", "aws", "novpc"]
title = "Whipping tablecloths with Docker for AWS, CloudFormation, and Terraform"
+++

I've been using [Docker for AWS][] (D4AWS) ever since I was accepted into the 
public beta, and have been running it in production for a while. One of the
best features of D4AWS is that you can deploy it without needing any
client-side tooling, due to its use of AWS [CloudFormation][] _(CFN)_.

This simplicity can also be a problem, since CloudFormation can be quite
inflexible. In the early days, if you wanted to add a D4AWS stack to an
existing VPC, for example, you were out of luck. Luckily, Docker soon added
a second [CloudFormation template][novpc template] that you can use, given
you have an existing VPC, 3 subnets, and appropriate route tables.

Personally, I prefer using [Terraform][] to provision and manage infrastructure,
and my D4AWS stacks are no exception - I use the [`aws_cloudformation_stack`][]
resource for this. And in the cases where I've added D4AWS to an existing VPC,
it's easy with Terraform to plug the right values into the right places.

This all means you have a _choice_ when deploying Docker for AWS: do you want
the whole infrastructure created and managed for you, with no effort? Or will
you choose the more complex route of provisioning your own networking, for more
flexibility down the road?

I've learned the importance of _making the right choice_ the hard way, and now
I'm faced with the situation that I need to move the VPC (and subnets) out from
CloudFormation's control, and into (in my case) Terraform's.

## The tablecloth trick

The key here is that I need to make this migration _without downtime_. Simply
tearing down one stack and bringing up another would be simple, but it would
also be a _real hassle_.

Fortunately, a number of features of CloudFormation and Terraform can help me
perform this magic trick.

### Setting the table

The goal here is to migrate between the two CFN templates, so let's start by
downloading both of them:

```console
$ curl -sSLO https://editions-us-east-1.s3.amazonaws.com/aws/stable/Docker.tmpl
$ curl -sSLO https://editions-us-east-1.s3.amazonaws.com/aws/stable/Docker-no-vpc.tmpl
$ ls -1sh *.tmpl
 92K Docker-no-vpc.tmpl
108K Docker.tmpl
```

We can see right away that there's about 16KB of difference between the two. Makes sense - the `-no-vpc` variant is missing a number of resources that are created by the other one.

If you `diff` them, these are the main differences:
  
1. `Docker-no-vpc.tmpl` adds 5 new _Parameters_: `Vpc`, `VpcCidr`, and `PubSubnetAz1` through `3`
2. `Docker.tmpl` has an _Output_ named `VPCID`
3. `Docker.tmpl` includes a number of resources:
    - `AttachGateway`
    - `Vpc`
    - `PubSubnetAz1` (through `3`)
    - `InternetGateway`
    - `PubSubnet1RouteTableAssociation` (through `3`)
    - `PublicRouteViaIgw`
    - `RouteViaIgw`

There is also a Lambda and some related resources which we won't need - it's
used for getting information about availability zones at deploy time.

The plan is to liberate 11 resources from the `Docker.tmpl` stack, and bring
them into Terraform's control:

1. first we'll use CloudFormation's [`DeletionPolicy`][] resource attribute to
  retain deleted resources
2. then we'll use Terraform's [import][Terraform Import] functionality to bring
  those into our control

When we're done, we'll end up having morphed from `Docker.tmpl` to `Docker-no-vpc.tmpl`.

### First course

The first step is to add a [`DeletionPolicy`][] entry to each of the network
resources we'll be moving.

For each of the resources that `Docker-no-vpc.tmpl` isn't going to be 
managing, we need to add a `"DeletionPolicy": "Retain"` entry.

I've done the changes, and the diff is:

{{< gist hairyhenderson 0676eb235c8ecf09d548ba7d6341ca84 "deletion_policies.diff" >}}

At this point, it's important to actually apply these changes. Since we're using
Terraform, this is as easy as:

```console
$ terraform apply
...
Apply complete! Resources: 0 added, 1 changed, 0 destroyed.
...
```

### Second course

Now the _fun_ part - importing resources into the Terraform state and configuration.
For now (as of v0.11.3) Terraform doesn't support generating configuration on
`import`, so that part must be done first, and by hand.

We'll need to create 10 Terraform resources:

- `aws_vpc`
- `aws_internet_gateway`
- `aws_subnet` (times `3`)
- `aws_route_table`
- `aws_route_table_association` (times `3`)
- `aws_route`

The new Terraform resources look roughly like this:

{{< gist hairyhenderson 0676eb235c8ecf09d548ba7d6341ca84 "new_resources.tf" >}}

A few details:

- `AWS::EC2::VPCGatewayAttachment` resources are only implicitly modeled by
Terraform, so we won't be explicitly creating a replacement for `AttachGateway`.
- the value that `AWS::StackName` resolves to is put in a variable named
`stack_name`
- some additional variables `VpcCidr`, `PubSubnetCidrs`, and `Azs` help reduce
hard-coding a bit
- note that the `aws_route_table` is named `RouteViaIgw`, as well as the `aws_route`,
and the `aws_route_table_association`s - this is due to how Terraform will import these
  - also note that the names have no correspondence to the subnets they map to - the names
  will have to be tweaked after running `terraform import` until the `terraform plan` is clean

#### Finding resource IDs

Before we can import the AWS resources into Terraform, we need to find their IDs.
It's fairly straightforward to do this through the AWS console UI, but in my
opinion it's faster to use the AWS CLI:

```console
$ aws ec2 describe-vpcs --output text --filters 'Name=tag:Name,Values="docker4aws-VPC"' --query Vpcs[].VpcId
vpc-12348765
$ aws ec2 describe-internet-gateways --output text --filters 'Name=tag:Name,Values="docker4aws-IGW"' --query InternetGateways[].InternetGatewayId
igw-18273645
$ aws ec2 describe-subnets --output text --filters 'Name=tag:Name,Values="docker4aws-Subnet1"' --query Subnets[].SubnetId
subnet-87654321
$ aws ec2 describe-subnets --output text --filters 'Name=tag:Name,Values="docker4aws-Subnet2"' --query Subnets[].SubnetId
subnet-87654322
$ aws ec2 describe-subnets --output text --filters 'Name=tag:Name,Values="docker4aws-Subnet3"' --query Subnets[].SubnetId
subnet-87654323
$ aws --profile developer --region us-east-1 ec2 describe-route-tables --output text --filters 'Name=tag:Name,Values="docker4aws-RT"' --query RouteTables[].RouteTableId
rtb-12344321
```

Notes:

- `Route`s and `RouteTableAssociation`s aren't _real_ resource type in the AWS API - 
they're properties of `RouteTable`, so we don't need to find IDs for those.

#### Importing resources

Now the fun starts. Starting with the VPC, we need to import each resource
into the terraform state, using the IDs discovered above:

```console
$ terraform import aws_vpc.Vpc vpc-12348765
...
$ terraform import aws_internet_gateway.InternetGateway igw-18273645
...
$ terraform import aws_subnet.PubSubnetAz1 subnet-87654321
...
$ terraform import aws_subnet.PubSubnetAz2 subnet-87654322
...
$ terraform import aws_subnet.PubSubnetAz3 subnet-87654323
...
$ terraform import aws_route_table.RouteViaIgw rtb-12344321
...
```

At this point, if you run `terraform plan`, the command should return with:

```
No changes. Infrastructure is up-to-date.
```

### Removing the tablecloth without disturbing the dishes...

Right now we're in a bit of an odd state - both Terraform and CloudFormation
think they're managing the same resources. The next step is to remove them from
the CloudFormation template.

#### Altering the CFN template

At this point we can start morphing `Docker.tmpl` to look more like
`Docker-no-vpc.tmpl`.

First off, we can simply delete the affected resources from `Docker.tmpl`. We'll
also need to delete the `VPCID` output, as well as add in the missing Parameters.

The easiest way to do this is simply to diff the two files (I like using `diff -u`),
and copy/paste the relevant pieces.

The diff should look something like this:

{{< gist hairyhenderson 0676eb235c8ecf09d548ba7d6341ca84 "removed_resources.diff" >}}

Note that this isn't _quite_ the same as `Docker-no-vpc.tmpl` yet. A few unnecessary
resources are still hanging around (Lambda-related). We'll clean them up later.

#### Passing in the parameters

The Terraform configuration now needs to be altered to pass in the appropriate IDs
as parameters. The `aws_cloudformation_stack` resources should look like this:

```
resource "aws_cloudformation_stack" "docker4aws" {
  template_body = "${file("Docker.tmpl")}"
  parameters {
    ...

    Vpc = "${aws_vpc.Vpc.id}"
    VpcCidr = "${aws_vpc.Vpc.cidr_block}"
    PubSubnetAz1 = "${aws_subnet.PubSubnetAz1.id}"
    PubSubnetAz2 = "${aws_subnet.PubSubnetAz2.id}"
    PubSubnetAz3 = "${aws_subnet.PubSubnetAz3.id}"
  }
}
```

#### Applying!

Now if we run `terraform plan` we should see 1 change - the `aws_cloudformation_stack`
resource. It's difficult to validate this since it's just a blob of encoded JSON,
but if we've been diligent above, we can do a `terraform apply`!

It's probably wise at this point to watch the update in the AWS Console UI, so we
can make sure nothing unexpected is happening.

The update shouldn't take too long (~1-2mins), and should contain a number of lines
where the status is `DELETE_SKIPPED`.

### Clearing the dishes

Now we can clean everything up and swap the `Docker.tmpl` for the full
`Docker-no-vpc.tmpl` file. The only differences should be related to the Lambda
resource that's no longer in use:

```
resource "aws_cloudformation_stack" "docker4aws" {
  template_body = "${file("Docker-no-vpc.tmpl")}"
  ...
}
```

At this point we can `terraform apply` again, and after ~30s the update should
be complete.

## Dessert

Migrating resources from CloudFormation's control to Terraform's control is not
too painful. The `DeletionPolicy` attribute makes this possible. We could
accomplish the transition by manually altering the Terraform state (either by
hand, or with the new `terraform state` commands), but the `terraform import`
command makes life so much easier!

[Docker for AWS]: https://web.archive.org/web/20180314143906/https://docker.com/docker-aws
[CloudFormation]: https://aws.amazon.com/cloudformation
[novpc template]: https://web.archive.org/web/20181116035529/https://docs.docker.com/docker-for-aws/#deployment-options
[Terraform]: https://terraform.io
[`aws_cloudformation_stack`]: https://registry.terraform.io/providers/hashicorp/aws/latest/docs/resources/cloudformation_stack.html
[`DeletionPolicy`]: https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-attribute-deletionpolicy.html
[Terraform Import]: https://developer.hashicorp.com/terraform/cli/import

