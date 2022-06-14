---
layout: post
title: "Manage ThingsBoard with Terraform"
categories: blog
tags: ["terraform", "IoT"]
published: true
comments: true
excerpted: |
  Discussing how to manage ThingsBoard with Terraform Restful Provider
script: [post.js]
---

{: toc}

ThingsBoard is an open-source server-side platform that allows you to monitor and control your IoT devices. It is free for both personal and commercial usage and you can deploy it anywhere.

This blog will use the community edition of ThingsBoard, to create a dashboard similar as the [hello world demo](https://thingsboard.io/docs/getting-started-guides/helloworld/), totally via [Terraform](https://terraform.io/).

## Why Terraform

The first benfit of using Terraform configuration to describe the infra is that the configuration can be used to reproduce the ThingsBoard infra easily.

The approach provided by ThingsBoard is [builk provisioning](https://thingsboard.io/docs/user-guide/bulk-provisioning/) (mainly for devices and assets) and other export/import features for different components. This is useful in the existing environment. However, once you want to use these artifacts to reproduce the same infrastructure, it will likely fail. The reason is that there are quite a lot cross component dependencies, which are referencing each other via a UUID. The UUID of each resource will be generated with a new value in a new provisioning, which causes the existing hardcoded references to be invalid.

On the other hand, Terraform resources can referencing each other's UUID in a dynamic way, that the referenced UUID is only known after the originating resource is provisioned.

Another major benefit of using Terraform is that Terraform not only makes Day 0 support for the infra, it is also a good experience for Day N maintainance. That means you can use Terraform (together with its existing ecosystem) to maintain the whole infrastructure, including adding new resources, updating and deleting existing resources.

Sharing reusable building blocks (e.g. dashboard, rule chains) with ThingsBoard users is another place where Terraform can help. Terraform [module](https://www.terraform.io/language/modules) is a container for multiple resources that are used together, which is the main way to package and reuse resource configurations with Terraform. With some good abstractions and designs, it is easy to create high quality Terraform modules for ThingsBoard.

## Terraform Plugins

Terraform is logically split into two main parts:

- Terraform Core: This is the Terraform binary that communicates with plugins to manage infrastructure resources. It provides a common interface that allows you to leverage many different cloud providers, databases, services, and in-house solutions.
- Terraform Plugins: These are executable binaries written in Go that communicate with Terraform Core over an RPC interface. Each plugin exposes an implementation for a specific service, such as the AWS provider or the cloud-init provider. Terraform currently supports one type of Plugin called providers.

Typically, a first class Terraform provider is needed for interacting with the target platform (e.g. ThingsBoard). This needs a Go SDK of that platform, then build the provider on top of the SDK.

Recently, I created a special Terraform provider: [magodo/restful](https://registry.terraform.io/providers/magodo/restful), which changes above fact a bit: This is a general Terraform provider aims to work for any platform as long as it exposes a RESTful API.

As ThingsBoard exposes its API in RESTful, means technically the restful provider can be able used for ThingsBoard. However, there are several assumptions about the API are required by the restful provider, where the ThingsBoard API doesn't meet:

- The OAuth2 using password credential grant should follow https://datatracker.ietf.org/doc/html/rfc6749#section-4.3
- Only supports `PUT` to update a resource

These violations causes:

- We have to retrieve the `Bearer` token via manual login, and configure the restful provider to using `http.Bearer` auth method
- Only Creating/Reading/Deleteing a resource is supported, no update is allowed

These are definitely not ideal. To resolve these issues, I've created a simple API proxy: https://github.com/magodo/thingsboard-api-proxy, which:

- Turns the authentication request from the Terraform Restful Provider in the urlencoded style, to its equivalent in body style, which is expected by the thingsboard API
- Turns PUT API request that is used by the Terraform Restful Provider to update a resource, to its equivalent POST form, which is expected by the thingsboard API

With this API proxy, we can now manage fully manage ThingsBoard reosurces via Terraform!


## "Hello World"

Now lets work through a "Hello World" example, step by step. Most of the setup steps are same as is stated in the [official get started](https://thingsboard.io/docs/getting-started-guides/helloworld/) page.

First thing first, let's start the thingsboard via docker:

```
mkdir -p ~/.mytb-data && sudo chown -R 799:799 ~/.mytb-data
mkdir -p ~/.mytb-logs && sudo chown -R 799:799 ~/.mytb-logs
docker run -it -p 9090:9090 -p 7070:7070 -p 1883:1883 -p 5683-5688:5683-5688/udp -v ~/.mytb-data:/data \
-v ~/.mytb-logs:/var/log/thingsboard --name mytb --restart always thingsboard/tb-postgres
```
These commands install ThingsBoard and load demo data and accounts (optionally, we can clean up all the demo data via the ThingsBoard UI)

ThingsBoard UI will be available using the URL: http://localhost:9090. You may use username `tenant@thingsboard.org` and password `tenant`.

Next, create a working directory to hold Terraform files:

```
mkdir ~/tf4tb && cd ~/tf4tb
```

Create a file named `main.tf` with following content:

```hcl
# This is the Terraform setting, that is required when you are using a 3rd party terraform provider (i.e. magodo/restful).
# See: https://www.terraform.io/language/settings#specifying-a-required-terraform-version.
terraform {
  required_providers {
    restful = {
      source = "magodo/restful"
      version = "= 0.2.0"
    }
  }
}

# Following are user defined variable, which are input as parameter when you run Terraform

# `base_url` is the URL of the thingsbaord-api-proxy (see: https://github.com/magodo/thingsboard-api-proxy), e.g. http://localhost:12345/api
variable "base_url" {
  type = string
}

# `username` is the username used when login in from ThingsBoard UI
variable "username" {
  type    = string
  default = "tenant@thingsboard.org"
}

# `password` is the password used when login in from ThingsBoard UI
variable "password" {
  type    = string
  default = "tenant"
}

# This is the provider config for the magodo/restful provider.
# See: https://www.terraform.io/language/providers/configuration for a general description of this block.
# See: https://registry.terraform.io/providers/magodo/restful/latest/docs for the meaning of each attribute.
provider "restful" {
  base_url = var.base_url
  security = {
    oauth2 = {
      token_url = format("%s/auth/login", var.base_url)
      username  = var.username
      password  = var.password
    }
  }
}

# This is a Data Source for querying user information.
# See: https://www.terraform.io/language/data-sources for a introduction of Data Source.
# See: https://registry.terraform.io/providers/magodo/restful/latest/docs/data-sources/restful_resource for the meaning of each attribute.
data "restful_resource" "user" {
  id = "/users"
  query = {
    pageSize = [10]
    page     = [0]
  }
  selector = format("data.#(name==%s)", var.username)
}

# This is a Resource for managing a ThingsBoard customer.
# See: https://www.terraform.io/language/resources for a introduction of  Resource.
# See: https://registry.terraform.io/providers/magodo/restful/latest/docs/resources/restful_resource for the meaning of each attribute.
# Note that this resource implicitly depends on the data source `restful_resource.user` for the tenant Id.
resource "restful_resource" "customer" {
  path      = "/customer"
  name_path = "id.id"
  body = jsonencode({
    title = "Example Company"
    tenantId = {
      id         = jsondecode(data.restful_resource.user.output).tenantId.id
      entityType = "TENANT"
    }
    country  = "US"
    state    = "NY"
    city     = "New York"
    address  = "addr1"
    address2 = "addr2"
    zip      = "10004"
    phone    = "+1(415)777-7777"
    email    = "example@company.com"
  })
}

# This is a Resource for managing a ThingsBoard device profile.
# Note that this resource implicitly depends on the data source `restful_resource.user` for the tenant Id.
resource "restful_resource" "device_profile" {
  path      = "/deviceProfile"
  name_path = "id.id"
  body = jsonencode({
    tenantId = {
      id         = jsondecode(data.restful_resource.user.output).tenantId.id
      entityType = "TENANT"
    }
    name               = "My Profile"
    description        = "Example device profile"
    type               = "DEFAULT"
    transportType      = "DEFAULT"
    defaultRuleChainId = null
    defaultDashboardId = null
    defaultQueueName   = null
    profileData = {
      configuration = {
        type = "DEFAULT"
      }
      transportConfiguration = {
        type = "DEFAULT"
      }
      provisionConfiguration = {
        type                  = "DISABLED"
        provisionDeviceSecret = null
      }
      alarms = null
    }
    provisionDeviceKey = null
    firmwareId         = null
    softwareId         = null
    default            = false
  })
}

# This is a Resource for managing a ThingsBoard device.
# Note that this resource implicitly depends on the data source `restful_resource.user`, resource `restful_resource.customer` and `restful_device_profile`.
resource "restful_resource" "device" {
  path      = "/device"
  name_path = "id.id"
  body = jsonencode({
    tenantId = {
      id         = jsondecode(data.restful_resource.user.output).tenantId.id
      entityType = "TENANT"
    }
    customerId = {
      id         = jsondecode(restful_resource.customer.output).id.id
      entityType = "CUSTOMER"
    }
    name  = "My Device"
    label = "Room 123 Sensor"
    deviceProfileId : {
      id         = jsondecode(restful_resource.device_profile.output).id.id
      entityType = "DEVICE_PROFILE"
    }
  })
}

# This is a Data Source for retrieving the device credential of the resource created via `restful_resource.device` .
data "restful_resource" "device_credential" {
  id = format("%s/credentials", restful_resource.device.id)
}

# These two resources are used to generate the UUID of a device entity (entity alias for the single device created above) and a device widget (which is created below).
resource "random_uuid" "my_device_entity" {}
resource "random_uuid" "my_device_widget" {}

# User defined local variables.
# See: https://www.terraform.io/language/values/locals. 
locals {
  # A local variable holds the definition of the device entity, which is actually a single entity maps the device we created above.
  # Note that it implicitly depends on the `restful_resource.device` and `random_uuid.my_device_entity`.
  my_device_entity = {
    alias = "MyDevice"
    filter = {
      resolveMultiple = false
      singleEntity = {
        entityType = "DEVICE"
        id         = jsondecode(restful_resource.device.output).id.id
      }
      type = "singleEntity"
    }
    id = random_uuid.my_device_entity.id
  }

  # A local variable holds the definition of a device widget, which is actually a "Card" widget.
  # Note that it implicitly depends on the `local.my_device_entity` and `random_uuid.my_device_widget`.
  my_device_widget = {
    bundleAlias = "cards"
    col         = 0
    config = {
      actions         = {}
      backgroundColor = "#ff5722"
      color           = "rgba(255, 255, 255, 0.87)"
      datasources = [
        {
          dataKeys = [
            {
              _hash    = 0.009193323503694284
              color    = "#2196f3"
              label    = "temperature"
              name     = "temperature"
              settings = {}
              type     = "timeseries"
            },
          ]
          entityAliasId = local.my_device_entity.id
          filterId      = null
          name          = null
          type          = "entity"
        },
      ]
      decimals         = 0
      dropShadow       = true
      enableFullscreen = true
      padding          = "16px"
      settings = {
        labelPosition = "top"
      }
      showLegend = false
      showTitle  = false
      timewindow = {
        realtime = {
          timewindowMs = 60000
        }
      }
      title = "New Simple card"
      titleStyle = {
        fontSize   = "16px"
        fontWeight = 400
      }
      units                  = "°C"
      useDashboardTimewindow = true
      widgetStyle            = {}
    }
    description  = null
    id           = random_uuid.my_device_widget.id
    image        = null
    isSystemType = true
    row          = 0
    sizeX        = 5
    sizeY        = 3
    title        = "New widget"
    type         = "latest"
    typeAlias    = "simple_card"
  }
}

# This is a Resource for managing a ThingsBoard dashboard.
# Note that this resource implicitly depends on a couple of resources/local variables we defined above.
resource "restful_resource" "dashboard" {
  path      = "/dashboard"
  name_path = "id.id"
  body = jsonencode({
    tenantId = {
      id         = jsondecode(data.restful_resource.user.output).tenantId.id
      entityType = "TENANT"
    }
    title = "My Dashboard"
    configuration = {
      entityAliases = {
        (local.my_device_entity.id) = local.my_device_entity
      }
      filters = {}
      settings = {
        showDashboardExport     = true
        showDashboardTimewindow = true
        showDashboardsSelect    = true
        showEntitiesSelect      = true
        showTitle               = false
        stateControllerId       = "entity"
        toolbarAlwaysOpen       = true
      }
      states = {
        default = {
          layouts = {
            main = {
              gridSettings = {
                backgroundColor    = "#eeeeee"
                backgroundSizeMode = "100%"
                columns            = 24
                margin             = 10
              }
              widgets = {
                (local.my_device_widget.id) = {
                  col   = 0
                  row   = 0
                  sizeX = 5
                  sizeY = 3
                }
              }
            }
          }
          name = "My Dashboard"
          root = true
        }
      }
      widgets = {
        (local.my_device_widget.id) = local.my_device_widget
      }
    }
  })
}

# A Terraform output variable, that output the device credential that is retrieved via the Data Source `restful_resource.device_credential`.
# See: https://www.terraform.io/language/values/outputs
output "device_token" {
  value     = jsondecode(data.restful_resource.device_credential.output).credentialsId
  sensitive = true
}
```

With above file created, we need to initialize Terraform, which will setup everything and download the used providers for us (i.e. the magodo/restful and hashicorp/random_uuid providers):

```
terraform init
```

Before we move on provinisioning everything via `terraform apply`, we need to start a local API proxy to adapt the ThingsBoard API with the magodo/restful provider:

```
# Install the proxy
# Note: Go environment (>= v1.16.0) is needed for below install command
go install github.com/magodo/thingsboard-api-proxy@latest

# Launch the proxy
thingsboard-api-proxy -taddr http://localhost:9090
```

With the proxy running, we can now open another terminal to run the Terraform command to provision everything:

```
# Note the port 12345 below is the default listening port of thingsboard-api-proxy
terraform apply -var=base_url=http://localhost:12345/api
```

Then Terraform will provide you an execution plan, telling what is gonna happen:

```
Terraform used the selected providers to generate the following execution plan. Resource actions are indicated with the following symbols:
  + create
 <= read (data resources)

Terraform will perform the following actions:

  # data.restful_resource.device_credential will be read during apply
  # (config refers to values not yet known)
 <= data "restful_resource" "device_credential"  {
      + header   = (known after apply)
      + id       = (known after apply)
      + output   = (known after apply)
      + query    = (known after apply)
      + selector = (known after apply)
    }

  # random_uuid.my_device_entity will be created
  + resource "random_uuid" "my_device_entity" {
      + id     = (known after apply)
      + result = (known after apply)
    }

  # random_uuid.my_device_widget will be created
  + resource "random_uuid" "my_device_widget" {
      + id     = (known after apply)
      + result = (known after apply)
    }

  # restful_resource.customer will be created
  + resource "restful_resource" "customer" {
      + body           = jsonencode(
            {
              + address  = "addr1"
              + address2 = "addr2"
              + city     = "New York"
              + country  = "US"
              + email    = "example@company.com"
              + phone    = "+1(415)777-7777"
              + state    = "NY"
              + tenantId = {
                  + entityType = "TENANT"
                  + id         = "96b97770-eada-11ec-94dd-db8b4f328f37"
                }
              + title    = "Example Company"
              + zip      = "10004"
            }
        )
      + create_method  = (known after apply)
      + header         = (known after apply)
      + id             = (known after apply)
      + ignore_changes = []
      + name_path      = "id.id"
      + output         = (known after apply)
      + path           = "/customer"
      + query          = (known after apply)
    }

  # restful_resource.dashboard will be created
  + resource "restful_resource" "dashboard" {
      + body           = (known after apply)
      + create_method  = (known after apply)
      + header         = (known after apply)
      + id             = (known after apply)
      + ignore_changes = []
      + name_path      = "id.id"
      + output         = (known after apply)
      + path           = "/dashboard"
      + query          = (known after apply)
    }

  # restful_resource.device will be created
  + resource "restful_resource" "device" {
      + body           = (known after apply)
      + create_method  = (known after apply)
      + header         = (known after apply)
      + id             = (known after apply)
      + ignore_changes = []
      + name_path      = "id.id"
      + output         = (known after apply)
      + path           = "/device"
      + query          = (known after apply)
    }

  # restful_resource.device_profile will be created
  + resource "restful_resource" "device_profile" {
      + body           = jsonencode(
            {
              + default            = false
              + defaultDashboardId = null
              + defaultQueueName   = null
              + defaultRuleChainId = null
              + description        = "Example device profile"
              + firmwareId         = null
              + name               = "My Profile"
              + profileData        = {
                  + alarms                 = null
                  + configuration          = {
                      + type = "DEFAULT"
                    }
                  + provisionConfiguration = {
                      + provisionDeviceSecret = null
                      + type                  = "DISABLED"
                    }
                  + transportConfiguration = {
                      + type = "DEFAULT"
                    }
                }
              + provisionDeviceKey = null
              + softwareId         = null
              + tenantId           = {
                  + entityType = "TENANT"
                  + id         = "96b97770-eada-11ec-94dd-db8b4f328f37"
                }
              + transportType      = "DEFAULT"
              + type               = "DEFAULT"
            }
        )
      + create_method  = (known after apply)
      + header         = (known after apply)
      + id             = (known after apply)
      + ignore_changes = []
      + name_path      = "id.id"
      + output         = (known after apply)
      + path           = "/deviceProfile"
      + query          = (known after apply)
    }

Plan: 6 to add, 0 to change, 0 to destroy.

Changes to Outputs:
  + device_token = (sensitive value)

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Type `yes` and press `enter` to continue, and you'll see:

```
random_uuid.my_device_widget: Creating...
random_uuid.my_device_entity: Creating...
random_uuid.my_device_widget: Creation complete after 0s [id=a74636a4-a8d5-b7b4-158e-e335447b2483]
random_uuid.my_device_entity: Creation complete after 0s [id=35d982d9-b43c-5344-4ed0-d43a7041999a]
restful_resource.customer: Creating...
restful_resource.device_profile: Creating...
restful_resource.device_profile: Creation complete after 1s [id=/deviceProfile/d2989ec0-ebdd-11ec-990a-59b15ed7052d]
restful_resource.customer: Creation complete after 1s [id=/customer/d298c5d0-ebdd-11ec-990a-59b15ed7052d]
restful_resource.device: Creating...
restful_resource.device: Creation complete after 0s [id=/device/d29a7380-ebdd-11ec-990a-59b15ed7052d]
data.restful_resource.device_credential: Reading...
data.restful_resource.device_credential: Read complete after 0s [id=/device/d29a7380-ebdd-11ec-990a-59b15ed7052d/credentials]
restful_resource.dashboard: Creating...
restful_resource.dashboard: Creation complete after 0s [id=/dashboard/d29c4840-ebdd-11ec-990a-59b15ed7052d]

Apply complete! Resources: 6 added, 0 changed, 0 destroyed.

Outputs:

device_token = <sensitive>
```

This just means everything is created. The last bit of the `Outputs` is redacted with `<sensitive>` is because we defined the output variable `device_token` as `sensitive` in the `main.tf`. To show the exact value of it, just run:

```
terraform output device_token
"M5avg0eWpdThOTMeEGD3"
```

Now with this token, let's send some telemetry to our device:

```
curl -v -X POST -d "{\"temperature\": 35}" http://localhost:9090/api/v1/M5avg0eWpdThOTMeEGD3/telemetry --header "Content-Type:application/json"
```

OK, let's open the ThingsBoard UI: http://localhost:9090 and validate everything just works:

![dashboard](/assets/img/thingsboard-terraform/dashboard.png)

This is not the end. From this point, you can totally manage these resource via Terraform. E.g. let's say you want to update the background color of the card widget, simply change the color definition of the card widget in `main.tf`:

```
-      backgroundColor = "#ff5722"
+      backgroundColor = "#0066ff"
```

Then run `terraform apply` again. This time, Terraform can still successfully generate the execution plan for you:


```
  ~ resource "restful_resource" "dashboard" {
      ~ body           = jsonencode(
          ~ {
              ~ configuration = {
                  ~ widgets       = {
                      ~ a74636a4-a8d5-b7b4-158e-e335447b2483 = {
                          ~ config       = {
                              ~ backgroundColor        = "#ff5722" -> "#0066ff"
                                # (16 unchanged elements hidden)
                            }
                            id           = "a74636a4-a8d5-b7b4-158e-e335447b2483"
                            # (11 unchanged elements hidden)
                        }
                    }
                    # (4 unchanged elements hidden)
                }
                # (2 unchanged elements hidden)
            }
        )

... (simplified)

Plan: 0 to add, 1 to change, 0 to destroy.

Do you want to perform these actions?
  Terraform will perform the actions described above.
  Only 'yes' will be accepted to approve.

  Enter a value:
```

Type `yes` and press `enter` to continue, and you'll get:

![dashboard](/assets/img/thingsboard-terraform/dashboard-blue.png)

Last step, you can destroy everything via Terraform with:

```
terraform destroy -var=base_url=http://localhost:12345/api
```

## Conclusion

Using Terraform to manage ThingsBoard is a new way for the ThingsBoard users. At least it serves me well for perfectly reproducing a whole provisioning of ThingsBoard project. Hope it can also serves the others!

As is demoed above, currently, we are still using a plain `curl` call (or via other protocols) for sending telemetry to the devices. Whilst, potentially we can integrate that part into Terraform also, with the ThingsBoard integration feature, e.g. https://thingsboard.io/docs/user-guide/integrations/azure-iot-hub/. Let's see!

## Reference

- ThingsBoard repository: https://github.com/thingsboard/thingsboard
- Terraform repository: https://github.com/hashicorp/terraform
- magodo/restful repository: https://github.com/magodo/terraform-provider-restful
- magodo/thingsboard-api-proxy repository: https://github.com/magodo/thingsboard-api-proxy