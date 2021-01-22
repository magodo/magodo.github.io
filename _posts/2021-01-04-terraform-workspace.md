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

    > 💡 NOTE: You have to set the `direct` as above, for the reason stated in github issue [hashicorp/terraform#25985](https://github.com/hashicorp/terraform/issues/25985#issuecomment-680052845).

3. (In case for local build) Before v0.14, simply `go install` the provider under *$HOME/go/bin*, which will be linked by the one in the file system mirror. Since Terraform v0.14, there is a [provider dependency lockfile](https://www.hashicorp.com/blog/announcing-hashicorp-terraform-0-14-general-availability) that will checksum the provider under used agaisnt a certain version. Imagine you are terraform init with a local build, do some fix on the provider, build and init again. Terraform will complain the newer built provider has different checksum than the prior one. A simple way to workaround is to simply remove the *.terraform.lock.hcl*. E.g. I used following shell function as an `terraform init` alternatives:

    ```shell
    tfinit () {
        rm .terraform.lock.hcl 2> /dev/null
        terraform init "$@"
    }
    ```

    > 💡 NOTE: To allow developers to easily test out local providers, since v0.14 Hashicorp also [provides an option `dev_overrides` in the terraform configuration](https://www.terraform.io/docs/commands/cli-config.html#development-overrides-for-provider-developers). It works great if you stick to using local build for a named provider, however, it doesn't work well when switching between local build and the upstream one. There are other tools complement this limitation, e.g. [tfdev](https://github.com/tombuildsstuff/tfdev), that provide another workflow for this use case.

4. (In case for reproduce an upstream issue) Simply specify the provider version in the provider section of the terraform config, which will pin the version and guide terraform to download that version from upstream (to local cache dir, to void multi downloads).
