---
title: "Terraform 0.14 - Error: Invalid legacy provider address"
date: 2021-02-10T17:15:21Z
author: "Carlos Castro"
tags: ["terraform"]
categories: ["terraform"]
description: "How to fix this error"
draft: false
showToc: false
TocOpen: false
hidemeta: false
comments: false
disableHLJS: true 
disableShare: true
disableHLJS: true
searchHidden: true
---

When upgrading to Terraform 0.14 directly from versions under Terraform 0.13, one the errors that you will see are related to the Terraform providers.

```sh
Error: Invalid legacy provider address

This configuration or its associated state refers to the unqualified provider
"aws".

You must complete the Terraform 0.13 upgrade process before upgrading to later
versions.
```

This specifc example show the error for the `aws` provider.

Before run with Terraform 0.14, if you run the code again with, at least once, Terraform 0.13, this error will disappear. If that is not an option, then run the following command:

```sh
terraform state replace-provider "registry.terraform.io/-/aws" "hashicorp/aws"
```

It will ask you for accepting the changes in the updated files.

**Note**: same applies for local or remote state.

```sh
terraform {
    backend "s3" {
        bucket = "my-remote-state-bucket"
        key    = "my-remote-state-key"
        region = "eu-west-1"
        dynamodb_table = "my.terraform.lock" # for state lock
        encrypt = true
    }
}
```

The next `terraform init` should run successfully. 
