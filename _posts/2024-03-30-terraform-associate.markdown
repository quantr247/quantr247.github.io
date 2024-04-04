---
layout: post
title:  "Terraform: Associate"
date:   2024-03-30 11:00:00 +0700
tags: terraform
categories: terraform
---

* IaC makes changes idempotent, consistent, repeatable and predictable.
* IaC: allow infrastructure to be versioned, code can easily be shared and reused, creates a blue print of your data center.
* IaC enables API-driven workflows for deploying resources in public clouds, private infrastructure, and other SaaS and PaaS services.

* Terraform is indeed an **immutable and declarative** Infrastructure as Code provisioning language.
* Terraform Core is a statically-compiled binary written in the **Go** programming language.
* Most Terraform configurations are written in the native **Terraform language (HCL)** syntax. Terraform also supports an alternative syntax that is **JSON-compatible**.
* Terraform by default provisions **10 resources** concurrently during a `terraform apply` command to speed up the provisioning process and reduce the overall time taken.
* 3 stage, workflow steps: **write --> plan --> apply**

## **Provider**
---
* Provider is a plugin to manage a specific cloud provider or service.
* Providers can be installed using multiple methods, including downloading from a Terraform public or private registry, the official HashiCorp releases page, a local plugins directory, or even from a plugin cache. Terraform **cannot**, however, install directly from the source code.
* A provider block may be **omitted** if its contents would otherwise be empty.
* Each provider be unique.
* **alias**: multiple provider
{% highlight terraform %}
provider "aws" {
  region = "us-east-1"
}

provider "aws" {
  alias  = "west"
  region = "us-west-2"
}
{% endhighlight %}

## **Data Types**
---
* Terraform automatically converts number and bool values to strings when needed.

## **Variables**
---
* The name of a variable can be any valid identifier **except** the following: **source, version, providers, count, for_each, lifecycle, depends_on, locals**.
* Environment variables --> tfvars file --> auto.tfvars file --> command line
```
    TF_VAR_name
```

## **Registry**
---
### **Terraform Registry**
* Terraform Registry is integrated directly into Terraform.

syntax of module: **NAMESPACE/NAME/PROVIDER**\
example: hashicorp/consul/aws

**Benefits**:
* support versioning
* show examples and README
* automatically generated documentation
* allow browsing version histories

### **Private Registry**
* While fetching a module, having a version is required.

syntax of module: **HOSTNAME/NAMESPACE/NAME/PROVIDER**

{% highlight terraform %}
module "vpc" {
  source  = "aws.terraform.abc/corp_example/vpc/aws"
  version = "v1.0.1"
}
{% endhighlight %}

## **Debugging**
---
* TF_LOG: TRACE, DEBUG, INFO, WARN or ERROR
* TF_LOG_PATH
* When TF_LOG_PATH is set, TF_LOG must be set in order for any logging to be enabled.

## **Terraform Plan**
---
* Terraform Plan will show output to remove the manually created resources.

```
    $ terraform plan
```

## **Terraform Import**
---
```
    $ terraform import aws_instance.example i-abcd1234
```

## **Terraform Workspace**
---
* By default, when you **create a new workspace** you are **automatically switched** to it.
* Terraform Workspace feature in Terraform Cloud **NOT** same as the Terraform workspace feature that is present in the free open-source version of Terraform.
* local state file: `terraform.tfstate.d/workspace_name`

```
    $ terraform workspace list
    $ terraform workspace show
    $ terraform workspace new [name]
    $ terraform workspace select [name]
```

## **Terraform State**
---
**Purpose of state**
* Mapping to the Real World
* Increased performance
* Determining the correct order to destroy resources

## **Terraform Functions**
---
* The Terraform language does **not** support user-defined functions, only the Terraform functions.

```
    $ terraform console
    > max(5, 12, 9)
    12
```

## **Count and Count Index**
---

{% highlight terraform %}
resource "aws_iam_user" "example" {
  name  = "example-name${count.index}"
  count = 3
}
{% endhighlight %}

## **Terraform Lock**
---
* Terraform lock state for all operations that could write state.
* force-unlock command to manually unlock the state if unlocking failed.
>$ terraform force-unlock LOCK_ID DIR
* State locking in Terraform is only required for write operations. 
* Read operations, such as terraform state list, can be performed even if the state file is locked.
* Not all backends supports locking functionally. **Support: s3, azurerm, consul**

## **Backend**
---
* Backend type: consul, s3, local, remote

## **Provisioner**
---
* remote-exec

{% highlight terraform %}
provisioner "remote-exec" {
    inline = [
      "puppet apply",
      "consul join ${aws_instance.web.private_ip}",
    ]
}
{% endhighlight %}

* local-exec

{% highlight terraform %}
provisioner "local-exec" {
    command = "echo ${self.private_ip} >> private_ips.txt"
  }
{% endhighlight %}

* **connection** block allows setting the credentials required. Supports both **ssh** and **winrm**.

## **Sentinel**
---
* An embedded policy-as-code framework to enforce standard and cost controls.
* Use-cases: verify if EC2 instance has tags, verify if the S3 bucket has encryption enabled.
* Sentinel Policies are checked when a run is performed, after the terraform plan but before it can be confirmed or the terraform apply is executed.

## **Dynamic Block**
---
* Dynamically construct repeatable nested blocks which is supported inside resource, data, provider, and provisioner blocks.

{% highlight terraform %}
dynamic "ingress" {
  for_each = var.ingress_ports
  content {
    from_port = ingress.value
    to_port   = ingress.value
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
{% endhighlight %}

### **Iterators**
* Iterator argument (optional) is a temporary variable that represents the current element of the complex value.
* If omitted, the name of the variable defaults to the label of the dynamic block ("ingress" in the example above).

{% highlight terraform %}
dynamic "ingress" {
  for_each = var.ingress_ports
  iterator = port
  content {
    from_port = port.value
    to_port   = port.value
    protocol  = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }
}
{% endhighlight %}

## **Terraform Graph**
---
* Command is used to generate a visual representation of either a configuration or execution plan.
* Output of terraform graph is DOT file (DOT format), which can convert to image.

## **Terraform Console**
---
* The terraform console command provides an interactive command-line console for evaluating and experimenting with expressions.
* Useful for testing interpolations before using them in configurations

## **Publishing Modules**
---
### **Requirement**
* Github with public repo
* Named: terraform-PROVIDER-NAME
* Repository description
* Standard module structure
* x.y.z tags for releases

## **Version Control System**
---
* Azure DevOps Server
* Azure DevOps Services
* Bitbucket Cloud
* Bitbucket Data Center
* GitHub.com
* GitHub Enterprise
* GitHub App for TFE
* GitLab EE and CE
* GitLab.com

## **Terraform Enterprise**
---
* Data storage requirement: PostgreSQL
* Linux Instance: Ubuntu, Oracle Linux, Red Hat Enterprise Linux, CentOS, Amazon Linux, Debian

## **OS Terraform supported**
---
* Linux
* macOS
* Solaris
* Windows

## **Terraform Cloud**
---
* A workspace can be mapped to 1 VCS repo, multi workspace can use the same repo
* Multiple workspaces using a variable set
* All current and future workspaces in a project using a variable set
* A specific Terraform run in a single workspace
* Terraform Cloud always encrypts state at rest
* Can be managed form the CLI but requires **an API token**
* Before use need run `terraform login`

## **Air Gap**
---
* Offline install Terraform Enterprise, Terraform Provider Plugins
* Execute `./install.sh` air gap command to begin the install

## **Vault**
---
* Vault Provider allows to read and write to HashiCorp Vault from Terraform.