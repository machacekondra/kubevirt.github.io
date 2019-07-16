---
layout: post
author: omachace
title: KubeVirt Terraform provider
description: Managing Kubevirt using Terraform
navbar_active: Blogs
pub-date: Jul 17
pub-year: 2019
category: news
comments: true
---

Terraform enables you to safely and predictably create, change, and improve infrastructure. With Terraform KubeVirt provider
you can manage also your virtual machines using Terraform altogether with rest of your infrastructure using Terraform.

### Setup

Currently the KubeVirt provider isn't part of official Terraform providers, it's developed [here](https://github.com/machacekondra/terraform-provider-kubevirt).
To use the KubeVirt provider, compile it first:

```bash
$ mkdir -p $GOPATH/src/github.com/machacekondra; cd $GOPATH/src/github.com/machacekondra
$ git clone git@github.com:machacekondra/terraform-provider-kubevirt
$ cd $GOPATH/src/github.com/machacekondra/terraform-provider-kubevirt
$ make build
```

This creates `terraform-provider-kubevirt` binary file, which you can start using.

### Virtual machine management

Let's create a Terraform configuration file, which will manage virtual machine for us.

```
provider "kubevirt" {
}

resource "kubevirt_virtual_machine" "myvm" {
  name = "myvm"
  namespace = "default"
  labels {
    label = "mylabel"
  }
  wait = true
  running = true
  image = {
    url = "http://download.cirros-cloud.net/0.3.6/cirros-0.3.6-x86_64-disk.img"
  }
  memory {
    requests = "128M"
    limits = "256M"
  }
  cpu {
    requests = "100m"
    limits = "200m"
    cores = 1
  }
  interfaces {
    name = "nic1"
    type = "bridge"
    network = "pod"
  }
  cloud_init = <<-EOF
  #cloud-config
  password: cirros
  chpasswd: { expire: False }
  EOF
}
```

In this file we first register our KubeVirt provider, by default we use `~/.kube/config` file, same as you may be used to from kubernetes provider.
Next we define the resource `kubevirt_virtual_machine` with many different parameters. You can define virtual machine CPU, memory, image, interfaces, cloud-init, etc.


```bash
terraform plan
$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.


------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  + create

Terraform will perform the following actions:

  + kubevirt_virtual_machine.myvm
      id:                   <computed>
      cloud_init:           "#cloud-config\npassword: cirros\nchpasswd: { expire: False }\n"
      cpu.#:                "1"
      cpu.0.cores:          "1"
      cpu.0.limits:         "200m"
      cpu.0.requests:       "100m"
      ephemeral:            "false"
      image.%:              "1"
      image.url:            "http://download.cirros-cloud.net/0.3.6/cirros-0.3.6-x86_64-disk.img"
      interfaces.#:         "1"
      interfaces.0.name:    "nic1"
      interfaces.0.network: "pod"
      interfaces.0.type:    "bridge"
      labels.%:             "1"
      labels.label:         "mylabel"
      memory.#:             "1"
      memory.0.limits:      "256M"
      memory.0.requests:    "128M"
      name:                 "myvm"
      namespace:            "default"
      running:              "true"
      wait:                 "true"


Plan: 1 to add, 0 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

Check the `plan` output and run `apply` if all is fine:

```bash
$ terraform apply -auto-approve
kubevirt_virtual_machine.myvm: Creating...
  cloud_init:           "" => "#cloud-config\npassword: cirros\nchpasswd: { expire: False }\n"
  cpu.#:                "" => "1"
  cpu.0.cores:          "" => "1"
  cpu.0.limits:         "" => "200m"
  cpu.0.requests:       "" => "100m"
  ephemeral:            "" => "false"
  image.%:              "" => "1"
  image.url:            "" => "http://download.cirros-cloud.net/0.3.6/cirros-0.3.6-x86_64-disk.img"
  interfaces.#:         "" => "1"
  interfaces.0.name:    "" => "nic1"
  interfaces.0.network: "" => "pod"
  interfaces.0.type:    "" => "bridge"
  labels.%:             "" => "1"
  labels.label:         "" => "mylabel"
  memory.#:             "" => "1"
  memory.0.limits:      "" => "256M"
  memory.0.requests:    "" => "128M"
  name:                 "" => "myvm"
  namespace:            "" => "default"
  running:              "" => "true"
  wait:                 "" => "true"
kubevirt_virtual_machine.myvm: Still creating... (10s elapsed)
kubevirt_virtual_machine.myvm: Creation complete after 58s (ID: myvm)

Apply complete! Resources: 1 added, 0 changed, 0 destroyed.
```

Our first virtual machine using Terraform was created, but what if we want to update it?
Let's add new label for the virtual machine:


```
labels {
  label = "mylabel"
  label2 = "mytest"
}
```

```bash
$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

kubevirt_virtual_machine.myvm: Refreshing state... (ID: myvm)

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place

Terraform will perform the following actions:

  ~ kubevirt_virtual_machine.myvm
      labels.%:      "1" => "2"
      labels.label2: "" => "mytest"


Plan: 0 to add, 1 to change, 0 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

Terraform notice the change and updates the virtual machine on next run.
But what if we want to change the image of the virtual machine? The virtual machine must be recreated then!
Try to change the image `url` to new value.

```
image {
  url = "http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img"
}
```

```bash
$ terraform plan
Refreshing Terraform state in-memory prior to plan...
The refreshed state will be used to calculate this plan, but will not be
persisted to local or remote state storage.

kubevirt_virtual_machine.myvm: Refreshing state... (ID: myvm)

------------------------------------------------------------------------

An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
-/+ destroy and then create replacement

Terraform will perform the following actions:

-/+ kubevirt_virtual_machine.myvm (new resource required)
      id:                   "myvm" => <computed> (forces new resource)
      cloud_init:           "#cloud-config\npassword: cirros\nchpasswd: { expire: False }\n" => "#cloud-config\npassword: cirros\nchpasswd: { expire: False }\n"
      cpu.#:                "1" => "1"
      cpu.0.cores:          "1" => "1"
      cpu.0.limits:         "200m" => "200m"
      cpu.0.requests:       "100m" => "100m"
      ephemeral:            "false" => "false"
      image.%:              "1" => "1"
      image.url:            "http://download.cirros-cloud.net/0.3.6/cirros-0.3.6-x86_64-disk.img" => "http://download.cirros-cloud.net/0.4.0/cirros-0.4.0-x86_64-disk.img" (forces new resource)
      interfaces.#:         "1" => "1"
      interfaces.0.name:    "nic1" => "nic1"
      interfaces.0.network: "pod" => "pod"
      interfaces.0.type:    "bridge" => "bridge"
      labels.%:             "1" => "1"
      labels.label:         "mylabel" => "mylabel"
      memory.#:             "1" => "1"
      memory.0.limits:      "256M" => "256M"
      memory.0.requests:    "128M" => "128M"
      name:                 "myvm" => "myvm"
      namespace:            "default" => "default"
      running:              "true" => "true"
      wait:                 "true" => "true"


Plan: 1 to add, 0 to change, 1 to destroy.

------------------------------------------------------------------------

Note: You didn't specify an "-out" parameter to save this plan, so Terraform
can't guarantee that exactly these actions will be performed if
"terraform apply" is subsequently run.
```

From the `plan` output we can read, that the Terraform will re-create the virtual machine, because
updating the image wouldn't have any effect.

You can see whole recorded example as asciinema recording here: [![asciicast](https://asciinema.org/a/257590.svg)](https://asciinema.org/a/257590)
