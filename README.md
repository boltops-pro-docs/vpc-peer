<!-- note marker start -->
**NOTE**: This repo contains only the documentation for the private BoltsOps Pro repo code.
Original file: https://github.com/boltopspro/vpc-peer/blob/master/README.md
The docs are publish so they are available for interested customers.
For access to the source code, you must be a paying BoltOps Pro subscriber.
If are interested, you can contact us at contact@boltops.com or https://www.boltops.com

<!-- note marker end -->

# VPC Peer Blueprint

[![Watch the video](https://img.boltops.com/boltopspro/video-preview/multiple/vpc-peer.png)](https://youtu.be/0se1qMQ1Vv8)

[![BoltOps Badge](https://img.boltops.com/boltops/badges/boltops-badge.png)](https://www.boltops.com)

This blueprint peers VPCs across different AWS accounts.

* Creates the necessary IAM Roles in the management control AWS account.
* Creates the VPC Peering connections in the requesting AWS account.
* Sets up the routes tables to route the VPCs to each other. The route tables to use are all configurable.

If you are interested in peering VPCs in the same account, refer to [boltopspro/vpc-peer-one](https://github.com/boltopspro-docs/vpc-peer-one)

## Prerequisite

* Setup config/settings.yml: [Settings Setup](https://github.com/boltopspro-docs/reference-architecture/blob/master/docs/settings-setup.md)

## Usage

1. Add blueprint to Gemfile
2. Configure and Deploy Each Template

## Add

Add the blueprint to your lono project's `Gemfile`.

```ruby
gem "vpc-peer", git: "git@github.com:boltopspro/vpc-peer.git"
```

## Configure and Deploy Each Template

We'll use multiple templates in this blueprint to peer VPCs across multiple AWS accounts. Here's a summary:

1. [VPC Peer Roles](app/templates/vpc-peer-roles.rb) - Create IAM roles with peering permission.
2. [VPC Peer](app/templates/vpc-peer.rb) - Peer on the development and production VPC side.
3. [Management Routes](app/templates/management-routes.rb) - Peer on the management VPC side.

We'll go through each step next.

## 1. VPC Peer Roles

For cross-account VPC peering, we first we need IAM roles. The IAM roles live in the management AWS account. They provide the development and production account access to peer to the management account.  The [vpc-peer-roles](app/templates/vpc-peer-roles.rb) template creates the roles required.

Use [lono seed](https://lono.cloud/reference/lono-seed/) to create a starter variables file.

    LONO_ENV=management lono seed vpc-peer

The generated variables file look something like this:

configs/vpc-peer/variables/management.rb

    # Quick way to get account ids:
    #     AWS_PROFILE=dev aws sts get-caller-identity
    #     AWS_PROFILE=prd aws sts get-caller-identity
    @account_ids = {
      development: "112233445566",
      production: "778899001122",
    }

Fill in your actual development and production AWS account ids. For now, you can ignore the other variables. They are used for the Management Routes later.

Create the roles with `lono cfn deploy`:

    LONO_ENV=management lono cfn deploy vpc-peer-roles --blueprint vpc-peer --template vpc-peer-roles --sure --no-wait

The CloudFormation Outputs has the Role ARNs. They'll look something like this:

![](https://img.boltops.com/boltopspro/blueprints/vpc-peer/vpc-peer-roles-outputs.png)

You'll need the Role ARNs for the next template.

## 2. VPC Peer

The vpc-peer template does most of the heavy lifting. It'll create the peering connection, and also add the routes on the requester VPC side, IE: development and production.

Use [lono seed](https://lono.cloud/reference/lono-seed/) to create a starter variables file.

    LONO_ENV=development lono seed vpc-peer
    LONO_ENV=production  lono seed vpc-peer

The `configs/vpc-peer/` will look like this:

    configs/vpc-peer/
    ├── params
    │   ├── development.txt
    │   └── production.txt
    └── variables
        ├── development.rb
        └── production.rb

The config files look like this:

configs/vpc-peer/params/development.txt:

    # Required parameters:
    AccepterAccountId=112233445566 # management AWS account. AWS_PROFILE=mgt aws sts get-caller-identity
    RequesterRouteOutCdir=10.21.0.0/16 # route to management VPC CIDR
    PeerRoleArn=arn:aws:iam::112233445566:role/vpc-peer-roles-PeerRoleDevelopment-RY7RMMNAWGNL # Find in vpc-peer-roles CloudFormation Outputs on mgt account
    AccepterVpc=vpc-111 # management vpc. Find in vpc CloudFormation Outputs in mgt account
    RequesterVpc=vpc-222 # development or production. Find in vpc CloudFormation Outputs in dev and prd account

configs/vpc-peer/variables/development.rb:

    # Comma-separated lists. Get values from the VPC Outputs in the CloudFormation console
    @requester_route_tables="rtb-111,rtb-222,rtb-333" # development or production. IE: AllRouteTables

Set the config values.  You can grab the config values from the CloudFormation Outputs of the VPC stack the different AWS accounts. Here's a screenshot from a development account as an example:

![](https://img.boltops.com/boltopspro/demo-apps/backend/dev-vpc-outputs.png)

Once you're ready, you can deploy:

    LONO_ENV=development lono cfn deploy vpc-peer --sure --no-wait
    LONO_ENV=production  lono cfn deploy vpc-peer --sure --no-wait

After the deploy completes, the peering connections have been created. The routes on the development and production side have also been setup.  Here's a screenshot of the development account so you know what it looks like:

![](https://img.boltops.com/boltopspro/blueprints/vpc-peer/dev-vpc-peer-resources.png)

## 3. Management Routes

The vpc-peer stack that you deployed in the previous step created the routes on the development and production VPC side.  The routes on the management VPC side has not been created yet. The management-routes.rb template creates the routes on the management VPC side.

You should already have a variables file. It should look something like this:

configs/vpc-peer/variables/management.rb

```ruby
# Look for VpcPeeringConnection in the vpc-peer stacks.
@peer_connections = {
  development: {
    peer_connection: "pcx-111", # Find at vpc-peer CloudFormation Outputs VpcPeeringConnection
    vpc_cidr: "10.20.0.0/16",
  },
  production: {
    peer_connection: "pcx-222", # Find at vpc-peer CloudFormation Outputs VpcPeeringConnection
    vpc_cidr: "10.22.0.0/16",
  },
}
# Grab the routes table from the vpc stack Outputs in the management account: IE: AllRouteTables
@management_route_tables = "rtb-111,rtb-222,rtb-333"
```

Set the `@peer_connections` and `@management_route_tables` variables.  You can find the config values on the CloudFormation console on each account and vpc-peer and vpc stacks.

Example of vpc-peer output on development AWS account:

![](https://img.boltops.com/boltopspro/blueprints/vpc-peer/dev-vpc-peer-outputs.png)

Example of vpc output with AllRoutesTable on management AWS account:

![](https://img.boltops.com/boltopspro/blueprints/vpc/mgt-vpc-outputs.png)

After setting the configs, you can deploy:

    LONO_ENV=management lono cfn deploy vpc-peer-routes --blueprint vpc-peer --template management-routes --sure --no-wait

That creates all the routes on the management VPC back to the development and production VPCs.

## IAM Permissions

The IAM permissions required for this stack are described below.

Service | Description
--- | ---
cloudformation | To launch the CloudFormation stack.
ec2 | VPC Peering Connections
s3 | Lono managed s3 bucket

## Back to Reference Architecture

That's it. Go back to the main [boltopspro/reference-architecture](https://github.com/boltopspro-docs/reference-architecture/blob/master/README.md)

