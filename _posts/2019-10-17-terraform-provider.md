---
layout: "post"
title: "Terraform Provider Tips"
categories: "blog"
tags: ['terraform-provider']
published: True
comments: true
script: [post.js]
excerpted: |
    关于Terraform及其Provider编写的概念和Tips...
---

* TOC
{:toc}

# 1. Terraform language fomat

[官网](https://www.terraform.io/docs/configuration/index.html#arguments-blocks-and-expressions)中对于terraform的语言格式描述如下：

```
<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block body
  <IDENTIFIER> = <EXPRESSION> # Argument
}
```

在 [官网的另一个页面](https://www.terraform.io/docs/configuration/syntax.html)，有更为详细的语法说明。其中需要注意的是 `block body` 中的内容除了 `argument`（ [某些地方](https://www.terraform.io/docs/extend/terraform-0.12-compatibility.html#configuration-syntax-changes) 也称为 `attribute` ）以外，也允许是内嵌的 `block`。

因此，上面这个spec应该更新为：

```
<BLOCK TYPE> "<BLOCK LABEL>" "<BLOCK LABEL>" {
  # Block body

  <IDENTIFIER> = <EXPRESSION> # Argument

  OR

  <IDENTIFIER> {              # Nested Block
      # Block body
  }
}
```

与之前不同的是，我在 `block body` 中加入了 `Nested Block`。需要注意的是这里 `argument` 的syntax（有 `=` 号） 和 `block` 的syntax（没有 `=` ）不同。这个限制在tf 0.12 以后是[强制](https://www.terraform.io/docs/extend/terraform-0.12-compatibility.html#configuration-syntax-changes)的。

另外， `argument` 除了Provider定义的以外，不同的 `block type` 还会有一些 [`meta argument`](https://www.terraform.io/docs/configuration/resources.html#meta-arguments).

建议，为了区分这两个 `block body` 中仅有的两类元素，遵循官方以及源码中的命名，称呼它们为：`attribute`/`argument` 和 `block`.

# 98 How to implement Azure Resource 

## 98.1 Managed Resource

Resource 基本只要：

- 实现 onCreate()
- 实现 onRead()
- 实现 onUpdate()
- 实现 onDelete()
- Schema 定义

### 98.1.1 onCreate/onUpdate

在AZureRM的SDK中，Client大多会将create和update实现在一个叫做`CreateOrUpdate()`的函数中。因此，`onCreate()`和`onUpdate()`赋值为同一个callback。

**注意**: `Resource.Update` callback是可选参数，如果没有被实现，那么说明该资源不支持update

这个callback接收两个参数：

- `d schema.ResourceData`: 代表tf接管的state的对象(包括定义在configuration中的部分)，在这个callback中需要读取configuration中的内容，然后组合成SDK的输入参数调用client
- `meta interface{}`: provider初始化的时候定义的，在AzureRM中是一个Client的集合对象(`ArmClient`)

这个callback主要做的事情是：

1. 读取configuration中的设置
1. 检查指定的resource是否已经在azure存在，但是没有被tf接管：

    这部分代码在azure中大概结构如下：

    ```go
    // 如果当前被apply的resource tf没有接管，那么 `d.IsNewResource()` 返回 `true`
    // `features.ShouldResourcesBeImported()`： 用户可以选择opt-in（AzureRM 2.0以后默认为true），
    // 如果opt-in，代表对于已经在azure存在的资源用户必须先import才可以通过apply修改它。否则，下面这段代码
    // 就会被用于阻止用户apply一个已经存在，但tf未接管的资源。
    if features.ShouldResourcesBeImported() && d.IsNewResource() {
      resp, err := client.Get(ctx, resourceGroup, serverName, name)
      if err != nil {
        if !utils.ResponseWasNotFound(resp.Response) {
          return fmt.Errorf("xxx", name, resourceGroup, serverName, err)
        }
        // * MARK
      }

      if resp.ID != nil && *resp.ID != "" {
        return tf.ImportAsExistsError("xxx", *resp.ID)
      }
    }
    ```

    上面注释的 `Mark` 的这段代码的错误处理由于azure历史问题，是有违Go语言惯例的。在Go中，当一个函数返回时，例如 `result, err := Foo()`，一般 `result`的值仅当 `err == nil` 的时候才会有意义。
    
    而这里azure的go sdk代码中，`client.Get()` 返回的 `err` 的语义为：仅当请求返回200，才认为 `err == nil`。例如下面的定义：

    ```go
    func (client FirewallRulesClient) Get(ctx context.Context, resourceGroupName string, serverName string, firewallRuleName string) (result FirewallRule, err error) {
      //...
      req, err := client.GetPreparer(ctx, resourceGroupName, serverName, firewallRuleName)
      if err != nil {
        err = autorest.NewErrorWithError(err, "sql.FirewallRulesClient", "Get", nil, "Failure preparing request")
        return
      }

      //...
      resp, err := client.GetSender(req)
      if err != nil {
        result.Response = autorest.Response{Response: resp}
        err = autorest.NewErrorWithError(err, "sql.FirewallRulesClient", "Get", resp, "Failure sending request")
        return
      }

      //...
      result, err = client.GetResponder(resp)
      if err != nil {
        err = autorest.NewErrorWithError(err, "sql.FirewallRulesClient", "Get", resp, "Failure responding to request")
      }

      return
    }
    
    // ...

    func (client FirewallRulesClient) GetResponder(resp *http.Response) (result FirewallRule, err error) {
      err = autorest.Respond(
        resp,
        client.ByInspecting(),
        azure.WithErrorUnlessStatusCode(http.StatusOK), // DEFINED HERE!!!
        autorest.ByUnmarshallingJSON(&result),
        autorest.ByClosing())
      result.Response = autorest.Response{Response: resp}
      return
    }
    ```

    个人认为将 `err` 的语义改为只要服务端正确接收了请求并且返回，那么`err`就应该置为`nil`。至于请求的结果则由`result`定义。

1. 将读取的config转换成sdk client的参数，调用client的 `CreateOrUpdate()`

    **注意**

    - `CreateOrUpdate()`的参数构造的过程称为 `expand`，这个过程简单来说就是: `configuration -> sdk struct` 的过程。可能只是简单的转换，也可能包含额外的 `client.Get()` 调用 （例如： [virtual network](https://github.com/terraform-providers/terraform-provider-azurerm/blob/master/azurerm/resource_arm_virtual_network.go#L303)）
    - `CreateOrUpdate()`函数在某些实现中返回一个future对象，需要等待

1. 调用client的`Get()`函数，从response中获取ID并且设置ID(`d.SetId()`)
1. `return onRead()`

### 98.1.2 onRead

在Azure中，`client.Get()`返回的ID，也即记录在tf state中的ID是resource的长ID，记录了包括：subscription, resource group, resource name 等信息。在Azure Provider中有一个叫做 `ParseAzureResourceID()` 的函数用来将这个长ID解析为一个`ResourceID`的结构体：

```go
type ResourceID struct {
	SubscriptionID string
	ResourceGroup  string
	Provider       string
	Path           map[string]string
}  
```

Azure的资源的长ID总是有偶数个字段，因为每个component都是以 `key/value` 的形式表现在ID中。`ResourceID.Path` 的map中存的内容为 长ID中除了 `SubscriptionID`, `ResourceGroup`, `Provider` 以外的那些component（例如：资源的name）。

说了这些背景以后再来看看这个`onRead()` callback需要做哪些事情：

1. 从 `ResourceData` 中读取长ID，并解析得到 `ResourceID`
1. 调用 `client.Get()` 获取该当前信息

    **注意**: 如果返回的resp是`http.StatusNotFound`，那么需要将 `ResourceData`的`Id`置空，这会通知terrafrom该资源在azure中已经被删除，terraform于是也会从它的state中删除该资源([refer](https://www.terraform.io/docs/extend/writing-custom-providers.html#implementing-read))

1. 将获取的信息转换为tf configuration的格式(这个过程称为 `flatten`)，用来更新`ResourceData`


### 98.1.3 onDelete

主要步骤：

1. 从 `ResourceData` 中读取长ID，并解析得到 `ResourceID`
1. 调用`client.Delete()`将资源同时从azure和tf state删除 (这个函数在某些实现中返回一个future对象，需要等待)
   
    **注意**: 由于terrafrom对`Destroy`函数的定义是：
  
    > - If the Destroy callback returns without an error, the resource is assumed to be destroyed, and all state is removed.
    > - If the Destroy callback returns with an error, the resource is assumed to still exist, and all prior state is preserved.

  
    因此，如果`client.Delete()`返回的 `err != nil`，需要特殊地判断资源是否已经从azure删除了，如果已经删除，则直接返回 `nil`.



## 98.2 Data Resource

data resource 本质上就是一个只有`Read()`的managed resource，实现的唯一区别在于 data resource 的 `Read()` 需要设置`ResourceData`的ID，而这个ID实际就是提供这些data的后端resource的长ID。

## 98.3 Azure Virtual Resource

在Azure中有一些resource，它们的后端并不是该resource特定的service，甚至有一些resource没有后端service提供其CRUD。我将这些service称为 Virtual Resource.

### 98.3.1 Association Resource

例如：`azurerm_network_interface_application_gateway_association` 是一个 **Association Resource**, 它将 `azurerm_network_interface` 和 `azurerm_application_gateway`关联起来（通过各自的ID）。这种情况一般属于两个已经存在的资源，在后续的迭代中发现某些use case下需要两者关联使用。

这时候，有两种关联的做法：

1. 将其中一个resource的ID加入到另一个resource的schema。这样有个弊端：将两个资源一定程度耦合
2. 定义一个association resource，将两个资源关联

Azure中几乎全部的 association resource 都没有实现 `Resource.Update` callback。这表明这些resource一旦修改，就需要删除并重建。由于它们本来就是定义association这种关联关系的，所以这是可以接受的。

### 98.3.2 Attaching Resource

例如：`azurerm_servicebus_event_authorization_rule` 是一个 **Attaching Resource**，这个resource实际上作为另一个有后端service，`azurerm_servicebus_event`，对应的resource的一部分存在。在这个后端service的API spec中，attaching resource实际上对应的是API的某个sub-path。因此，在其tf resource的实现中，它使用的 azure go sdk的client就是那个实际的service（即，`azurerm_servicebus_event`）。

可见，这种resource在tf中实现的前提是后端service的API对其有直接的支持。

## 98.4 Test

## 98.5 Document

# 99 TIP

## 99.1 block body nested map

在 [官网](https://www.terraform.io/docs/extend/writing-custom-providers.html#implementing-a-more-complex-read)有提到：

> Due to the limitation of [tf-11115](https://github.com/hashicorp/terraform/issues/11115) it is not possible to nest maps. So the workaround is to let only the innermost data structure be of the type `TypeMap`

这里的workaround在tf 0.12版本以后就被修复了，具体可以看上面那个issue。同时，有另外一个与之关联的[issue](https://github.com/hashicorp/terraform-website/issues/908)要求把上面这个章节删除。So...如果你打不开上面那个官网的章节的话就代表已经被删除了，如果你依然可以访问，请无视 （我在这里看了好一会儿 😶）

## 99.2 tf provider 没有输出

在provider中不晓得为啥输出到标准输出和错误都没用（即使export `TF_LOG`），打log的方法只能通过写文件
