---
title: Quick Start Connections With Terraform and Kubernetes
author: Adron Hall
date: 2017-09-22 10:40:19
template: article.jade
---
Friday is here... a quick sitrep on my practices around Terraform connections with Azure, AWS, and GCP. In the past week I've been working on getting Terraform deploying a clean Kubernetes Cluster, then getting connected to that cluster and getting some resources deployed to the cluster. This article I'm just going to cover the practices I'm using to secure connections with the respective cloud providers, but more is coming soon.

<span class="more"></span>

# Kubernetes and Terraform

I've used Azure and Google Compute Platform (GCP) to accomplish the initial Kubernetes Cluster deployments. The following is the path I followed to get each cluster running on the respective platform.

Before diving directly into the deployments I've setup with Terraform, I'll cover a little of how I've setup Terraform to authenticate and such. First, Azure and then GCP.

### Azure Terraform Setup

My Terraform configuration file for connections with Azure looks like this.

```
provider "azurerm" {
  subscription_id = "${var.subscription_id}"
  client_id       = "${var.client_id}"
  client_secret   = "${var.client_secret}"
  tenant_id       = "${var.tenant_id}"
}
```

What I've done is setup a variable for each of the connection requirements. These variables are then set via environment variables. The environment variables I set via my ~/.bash_profile or ~/.bashrc file depending on if I'm working on the System76 Ubuntu machine, Mac Book Pros, or iMacs I regularly work on.

```
export TF_VAR_subscription_id="90a9b1d7-1529-4ec4-abd4-e2ad84bfadf3"
export TF_VAR_client_id="e705e58a-8a2b-4b87-b69a-c3eee633b8e6"
export TF_VAR_client_secret="c4f13f94-4d1f-42c4-bfd5-cd55a3620d1b"
export TF_VAR_tenant_id="7325bc0f-1842-4126-8109-7c12773e7d30"
```

(Don't worry, those are fakey UUIDs, but they're the same type)

Once those are setup this way, I can easily connect to the Azure account without having any of my UUIDs (security IDs?, Key IDs?, whatever they're called) sitting in any actual repositories. It's all just variables.

The way Terraform works with environment variables is pretty straight forward, yet not intuitive without some solid [RTFM](https://en.wikipedia.org/wiki/RTFM) time. The short description is Terraform picks up any prepended environment variables starting with `TF_VAR` and using the name after that as the variable name. Thus, above, `TF_VAR_subscription_id` become simply `subscription_id` in the example above.

It's also important to note that Terraform pulls these environment variables in without the need for them to be declared in configuration. This can be somewhat confusing since other variables need declared before use regardless of being passed in at execution time or assigned in a *tfvars* file.

### GCP Terraform Setup

For GCP the connection I set it up using the *account.json* file. What I do is place the *account.json* file in a parallel *security* directory and simply reference it in the connection. Again, nothing with security information goes into the repository. The configuration looks like this.

```
provider "google" {
  credentials = "${file("../../secrets/account.json")}"
  project     = "mythrashingawesomeprojectname"
  region      = "us-west1"
}
```

With GCP there are not any magical environment variables. This is something I actually like, which makes it a little easier to make it all inclusive within a repository and a security package that I can securely move from machine to machine, instead of needing to re-implement more junk loading as environment variables. Those ~/.bash_profile and ~/.bashrc files get messy enough without additional development environment specific repository specific variables!

### AWS Setup

AWS is absurdly simple, yet it requires you've setup all the default connection and configuration for the AWS CLI. The only connection info for AWS is as shown below.

```
provider "aws" {}
```

In the end, with all three of these added my ~/.bashrc and ~/.bash_profile files look like this.

```
# Azure Variables
export TF_VAR_subscription_id="90a9b1d7-1529-4ec4-abd4-e2ad84bfadf3"
export TF_VAR_client_id="e705e58a-8a2b-4b87-b69a-c3eee633b8e6"
export TF_VAR_client_secret="c4f13f94-4d1f-42c4-bfd5-cd55a3620d1b"
export TF_VAR_tenant_id="7325bc0f-1842-4126-8109-7c12773e7d30"
# AWS Variables
export TF_VAR_aws_region="us-west-2"
```

Note the only AWS variable I specifically add, is the region, the CLI adds a bunch of others or sets/assumes environment defaults in the `~/.aws/` directory. It also sets the region, but I've found *some* tooling doesn't seem to use the set one, so I add the variable here so Terraform doesn't get confused and has it set specifically.

That's the way I find myself setting up connections for Terraform and AWS, Google Cloud Platform, and Azure. If you've got suggestions, questions, or thoughts on better ways to do this I'd love to hear about them. Ping me on Twitter [@Adron](https://twitter.com/Adron) and we'll discuss.

Alright, so now it's Kubernetes time. Once the above connection is worked out for the respective cloud providers, I can get a Kubernetes Cluster running in each. I'll tackle the creation of each below, broken out to GCP and Azure, with a follow up blog post on getting Terraform to launch a Kubernetes Cluster in AWS at a later time.

## GCP Kubernetes Cluster

For GCP I setup my resource as shown.

```
resource "google_container_cluster" "googlykuby" {
  name               = "${var.cluster_name}"
  zone               = "us-west1-a"
  initial_node_count = "${var.gcp_cluster_count}"

  additional_zones = [
    "us-west1-b",
    "us-west1-c",
  ]

  master_auth {
    username = "${var.linux_admin_username}"
    password = "${var.linux_admin_password}}"

  }

  node_config {
    oauth_scopes = [
      "https://www.googleapis.com/auth/compute",
      "https://www.googleapis.com/auth/devstorage.read_only",
      "https://www.googleapis.com/auth/logging.write",
      "https://www.googleapis.com/auth/monitoring",
    ]

    labels {
      this-is-for = "dev-cluster"
    }

    tags = ["dev", "work"]
  }
}
```

I setup the two variables; `cluster_name` and `gcp_cluster_count` to pull some of the settings into the variables file. Currently, my default setup is also to have two additional zones too, which means that a node will be setup in each of these zones. So expect the VM count to be `gcp_cluster_count` multiplied by the number of zones that are being used. In this case, there are two additional zones plus the zone the cluster is being created in, times let's say 2 for `gcp_cluster_count`, which will give me 6 VM instances running. This is respectively 6 nodes in the cluster.

The `master_auth` is then also setup, similarly to the way I setup the connection variables previously, in which they're environment variables or I pass them in via CLI parameters.

Then, just for good measure I've added some labels and tags. Partly as an example of how to use them but also just to differentiate this cluster versus other clusters I might have running.

This might be surprising, but that's it for the resource in Kubernetes. We're now ready to launch our cluster in GCP.

## Azure Kubernetes Cluster

The Azure setup is a bit different, and in some ways more extensive in demand than the GCP setup. I've tested out a few configurations and have ended up, generally, with configuration that looks like this.

```
resource "azurerm_container_service" "bluekuby" {
  name = "bluekubyhouse"
  location = "${azurerm_resource_group.blue_kuby_group.location}"
  resource_group_name = "${azurerm_resource_group.blue_kuby_group.name}"
  orchestration_platform = "Kubernetes"

  master_profile {
    count = 1
    dns_prefix = "${var.azure_cluster_prefix}"
  }

  linux_profile {
    admin_username = "${var.linux_admin_username}"

    ssh_key {
      key_data = "${var.ssh_key}"
    }
  }

  agent_pool_profile {
    name = "default"
    count = "${var.azure_node_count}"
    dns_prefix = "kuby.house"
    vm_size = "Standard_A0"
  }

  service_principal {
    client_id = "${var.client_id}"
    client_secret = "${var.client_secret}"
  }

  diagnostics_profile {
    enabled = false
  }

  tags {
    Environment = "Production"
  }
}
```

The location and resource group values passed in the `azurerm_resource_group.blue_kuby_group.location` and `azurerm_resource_group.blue_kuby_group.name` are created in the following resource.

```
resource "azurerm_resource_group" "blue_kuby_group" {
  name = "bluekubygroup"
  location = "West US"
}
```

One of the differences I've noticed with the Azure setup vs. the GCP setup is that it appears, the Azure setup needs an ssh key to work with for authentication purposes while the GCP setup doesn't particularly need this.

*Or does it?* I'll talk more about that in a future post. For now I'll leave you to ponder what one is doing versus the other! #shock #amaze #ohnoez!

This provides a simple summary of the two Kubernetes setups for GCP and Azure. In a subsequent post I'll dive into AWS Kubernetes setup via Terraform and also elaborate on some of the specifics that are going on when using Terraform to setup Kubernetes in any of these environments. Stay tuned, subscribe via [rss]() or for lower volume, subscribe to [Thrashing Code News](/)

**References:**

* My current [working repo](https://github.com/Adron/blue-world-making) of config & code related to [upcoming talks](/articles/technology-and-upcoming-talks/) and their respective examples.
* [Terraform AWS Provider Documentation](https://www.terraform.io/docs/providers/aws/index.html)
* [Terraform GCP Provider Documentation](https://www.terraform.io/docs/providers/google/index.html)
* [Terraform Azure Provider Documentation](https://www.terraform.io/docs/providers/azurerm/index.html)
