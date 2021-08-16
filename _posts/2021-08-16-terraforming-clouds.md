---
title: Terraforming Clouds
description: First impressions of Terraform in Azure
category: programming
---

[Terraform](https://www.terraform.io/) is a software tool for "infrastructure as
code". "Infrastructure as code" just means that you deploy cloud resources
(databases, applications, and how they're wired together) with a configuration
file. Terraform is a tool which reads its brand of configuration file and can
interact with several different clouds (Azure, Google Cloud, AWS, etc).

## The good

Once you learn how to write Terraform configuration files you can put them into
source control and create a continuous integration pipeline so that:

 * You can fearlessly change cloud resources
 * Cloud resources are always in sync with source code
 * You can easily revert to a previous version of source code _and_ a previous
   cloud setup

These are real benefits. You can have a continuous integration pipeline that
builds and tests your application, and then deploys it to the cloud while making
sure all the necessary infrastructure is in place and configured properly.

And it's awesome being able to put all the infrastructure in source control. It
feels a lot less fragile.

## The bad

You can't do everything with Terraform; in theory it lags behind. When Azure
comes out with something new then someone from Microsoft has to update the
Terraform
["plugin" for Azure](https://registry.terraform.io/providers/hashicorp/azurerm/latest/docs).
To be honest this isn't a problem for me.

Terraform will not show mercy to cloud resources that it controls. If you remove
a database from your configuration file then Terraform will not think twice
about deleting it. So make sure you keep regular backups (you would be doing
that anyway, right?) and in a storage container that isn't under Terraform's
control (otherwise your fat fingers will make it delete all your backups, too!)

There is a learning curve.

## The ugly

I'm not really sure how I would improve Terraform. The configuration language is
okay. The tooling feels complete. There is documentation for everything.

## The point

I think I'm going to use infrastructure as code techniques in all my future
cloud projects. You should, too!

Would I use Terraform? Probably. I need to finish learning about
[Azure Resource Manager Templates](https://docs.microsoft.com/en-us/dotnet/architecture/cloud-native/infrastructure-as-code#azure-resource-manager-templates).
Apparently Microsoft has a native solution for infrastructure as code, and I
might prefer that over Terraform.