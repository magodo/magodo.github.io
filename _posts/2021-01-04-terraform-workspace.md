---
layout: "post"
title: "Terraform Provider Contributor Workspace Setup"
categories: "blog"
tags: ['linux', 'terraform']
published: True
comments: true
script: [post.js]
excerpted: |
    A short description...
---

* TOC
{:toc}

As a Terraform provider contributor, one will needs to test out the in-house provider. In the meanwhile, one will need to reproduce some issues reported in community, pinning to a certain version, pulled from upstream.

Since Terraform v0.13, it brings some new stuffs:

- [New filesystem layout for local copies of providers](https://www.terraform.io/upgrade-guides/0-13.html#new-filesystem-layout-for-local-copies-of-providers)
- [Explicit installation Method Configuration](https://www.terraform.io/docs/commands/cli-config.html#explicit-installation-method-configuration)

To allow developers to easily test out local providers, since v0.14 Hashicorp also [provides an option `dev_overrides` in the terraform configuration](https://www.terraform.io/docs/commands/cli-config.html#development-overrides-for-provider-developers). Since it is introduced recently and I had the needs sicne v0.13, so below I'll tell my solutions.


Let's say I always need to test against terraform-provider-azurerm in my case:

1. Prepare local mirror for terraform-provider-azurerm:

    ```
    $ mkdir -p ~/.terraform.d/plugin/registry.terraform.io/hashicorp/azurerm/99.99.99/linux_amd64
    $ ln -s ~/go/bin/terraform-provider-azurerm  ~/.terraform.d/plugin/registry.terraform.io/hashicorp/azurerm/99.99.99/linux_amd64/terraform-provider-azurerm_v99.99.99
    ```
2. Setup the terraform config:

    ```
    cat << EOF > ~/.terraformrc
    provider_installation {
        filesystem_mirror {
            path = "/home/magodo/.terraform.d/plugins"
        }
        direct {
            exclude = []
        }
    }

    plugin_cache_dir = "$HOME/.terraform.d/plugin-cache"
    EOF
    ```

3. (In case for local build) Simply `go install` the provider under *$HOME/go/bin*, which will be linked by the one in the file system mirror.
4. (In case for reproduce an upstream issue) Simply specify the provider version in the provider section of the terraform config, which will pin the version and guide terraform to download that version from upstream (to local cache dir, to void multi downloads).

> 💡 NOTE: You have to set the `direct` as above, for the reason stated in github issue [hashicorp/terraform#25985](https://github.com/hashicorp/terraform/issues/25985#issuecomment-680052845).




