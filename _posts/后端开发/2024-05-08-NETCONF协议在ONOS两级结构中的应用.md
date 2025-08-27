---
layout: post
title: NETCONF协议在ONOS两级结构中的应用
date: 2024-05-08
tags: ONOS, NETCONF
categories:
  - 后端开发
---

## 1 什么是NETCONF

### 1.1 背景
传统命令行CLI和SNMP的缺陷：
- 命令行是人机接口，配置过程复杂，厂商差异大。
- 命令行输出内容是非结构化的，不可预测、容易变化，导致解析复杂，CLI脚本很难实现自动化解析。
- 命令行出错率高，不适用当代网络特点。
- SNMP可读性差。（1.3.6.1.2.1.2.2.1.4）
- SNMP配置效率低，不支持事务机制，更多被用来做监控类协议。
- SNMP采用UDP传输协议，无法提供可靠的、有序的数据传输，缺乏有效的安全性。
- SNMP缺乏配置事务提交机制，只能对一个一个对象单独配置，而不是面向一个业务。
- 模型兼容性差，MIB 库混乱，无法适配所有厂商，导致定义各种私有 MIB 库。
![](/images/20240508/netconf-1.png)

> 一句话总结：一帮网管大佬在IETF开了个会，分析了当时网络管理技术的优缺点（主要是吐槽缺点），然后发了个会议纪要，我们要搞个新协议，搞个新的数据建
> 模语言，让网管能管理配置下发，能监控设备，能各种远程操作，还要能跟CLI行为一致。

### 1.2 定义
NETCONF（Network Configuration Protocol）是基于XML的网络配置和管理协议。

### 1.3 协议框架
NETCONF协议也采用了分层结构。每层分别对协议的某一方面进行包装，并向上层提供相关服务。分层结构使每层只关注协议的一个方面，实现起来更简单，同时使各层之间的依赖、内部实现的变更对其他层的影响降到最低。
- **传输层**  
  传输层为NETCONF客户端（Client）和服务器端（Server）之间交互提供通信路径。对承载协议基本要求如下：
  - 面向连接，Client和Server之间必须建立持久的连接，连接建立后，必须提供可靠的序列化的数据传输服务。
  - 用户认证，数据完整、安全加密，NETCONF协议的用户认证，数据完整，安全保密全部依赖传输层提供。
  - 承载协议必须向NETCONF协议提供区分会话类型（Client或Server）的机制。
- **消息层**  
  消息层也叫RPC层，提供了一种简单的、不依赖于传输协议的消息请求和响应机制，以<rpc>和<rpc-reply>进行请求数据和响应数据的封装。
- **操作层**  
  操作层定义了一系列在RPC中应用的基本操作，基于xml编码，这些操作组成了NETCONF基本能力。
- **内容层**  
  内容层就是管理设备的配置数据，是唯一没有被标准化的层，没有标准的NETCONF数据建模语言和数据模型。
```text
        Layer                 Example
       +-------------+      +-----------------+      +----------------+
   (4) |   Content   |      |  Configuration  |      |  Notification  |
       |             |      |      data       |      |      data      |
       +-------------+      +-----------------+      +----------------+
              |                       |                      |
       +-------------+      +-----------------+              |
   (3) | Operations  |      |  <edit-config>  |              |
       |             |      |                 |              |
       +-------------+      +-----------------+              |
              |                       |                      |
       +-------------+      +-----------------+      +----------------+
   (2) |  Messages   |      |     <rpc>,      |      | <notification> |
       |             |      |   <rpc-reply>   |      |                |
       +-------------+      +-----------------+      +----------------+
              |                       |                      |
       +-------------+      +-----------------------------------------+
   (1) |   Secure    |      |  SSH, TLS, BEEP/TLS, SOAP/HTTP/TLS, ... |
       |  Transport  |      |                                         |
       +-------------+      +-----------------------------------------+
```

### 1.4 协议特点
- netconf是面向连接的，需要对等体之间的持久连接。
- 配置数据和状态数据分离。
- netconf实现必须支持SSH传输协议（RFC6241）。
- 标准的数据模型，统一的数据格式，所有厂商都可以识别。
- 支持网络级的配置和多设备配置协同。

## 2. 什么是yang

### 2.1 背景
NETCONF协议只是一个配置管理管理协议，NETCONF协议标准化，但是并没有统一定义各个设备厂商的API接口描述语言。  
YANG是专门为NETCONF协议设计的数据建模语言，用来为NETCONF协议设计可操作的配置数据、状态数据模型、远程调用（RPCs）模型和通知机制等。  
YANG数据模型定位为一个面向机器的模型接口，明确定义数据结构及其约束，可以更灵活、更完整的进行数据描述。

### 2.2 定义
YANG是NETCONF协议的一种面向管理对象的数据建模语言。YANG可以对NETCONF客户端和服务器端之间发送的所有数据进行一个完整的描述。从设备管理对象数据建
模的视角而言，YANG是对本设备上业务对象的一种层次化数据建模。

YANG用于为网络配置管理协议使用的配置数据、状态数据、远程过程调用和通知进行模型化。通过YANG描述数据结构、数据完整性约束、数据操作，形成了一个个
YANG模型（或者叫YANG文件）。

### 2.3 特点
- 简单易懂、层次化、模块化、可复用、可扩展。
- 支持丰富的基础数据类型定义和数据属性定义。
- 结构化定义，支持定义约束条件，机器直接识别，不需要人工干预。 

一个简单的interface定义的yang文件：
```
module example-interfaces {
  namespace "urn:example:interfaces";
  prefix "if";

  container interfaces {
    list interface {
      key "name";
      leaf name {
        type string;
      }
      leaf type {
        type string;
      }
      leaf enabled {
        type boolean;
      }
      leaf ip-address {
        type string;
      }
    }
  }
}
```
### 2.4 yin与yang
解析模型时用YIN模型文件。YIN是XML表达方式的YANG，YIN与YANG之间使用不同的表达方法但包含等价的信息。用YIN是为了利用各编程语言中现有的XML解析器
等工具。这些工具可用来进行数据过滤和验证，自动生成代码和文件或者其他任务，这样可以提升设备解析YANG模型的效率。
![](/images/20240508/netconf-2.png)

## 3 NETCONF能力与操作

<table>
  <tr>
    <th>RFC</th>
    <th>说明</th>
  </tr>
  <tr>
    <td><a href="https://datatracker.ietf.org/doc/html/rfc6241">RFC6241</a></td>
    <td>定义 NETCONF 的基本操作</td>
  </tr>
  <tr>
    <td rowspan="8"><a href="https://datatracker.ietf.org/doc/html/rfc6241">RFC6241</a></td>
    <td>Writable-Running 能力</td>
  </tr>
  <tr><td>Candidate Configuration 能力</td></tr>
  <tr><td>Confirmed Commit 能力</td></tr>
  <tr><td>Rollback-on-Error 能力</td></tr>
  <tr><td>Validate 能力</td></tr>
  <tr><td>Startup 能力</td></tr>
  <tr><td>URL 能力</td></tr>
  <tr><td>XPath 能力</td></tr>
  <tr>
    <td rowspan="2"><a href="https://datatracker.ietf.org/doc/html/rfc5277">RFC5277</a></td>
    <td>Notification 能力</td>
  </tr>
  <tr><td>Interleave 能力</td></tr>
</table>

### 3.1 RPC模型
rpc请求必须由受管设备串行处理，受管设备必须按照接收请求的顺序发送响应。

#### 3.1.1 &lt;rpc&gt;
`rpc`元素用于封装从客户端发送到服务器的NETCONF请求。`<rpc>`元素有一个强制属性`“message-id”`，它是由RPC的发送者选择的一个字符串，它通常编码一个单
调递增的整数。 RPC的接收者不解码或解释这个字符串，只是简单地保存它在任何生成的`<rpc-reply>`消息中用作`“message-id”`属性。
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <some-method>
        <!-- method parameters here... -->
    </some-method>
</rpc>

```

#### 3.1.2 &lt;rpc-reply&gt;
响应`<rpc>`操作发送`<rpc-reply>`消息。`<rpc-reply>`元素有一个强制属性`“message-id”`，它等于这是一个响应的`<rpc>`的`“message-id”`属性。
```xml
<rpc-reply message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <data>
        <!-- contents here... -->
    </data>
</rpc-reply>
```

#### 3.1.3 &lt;rpc-error&gt;
如果在处理`<rpc>`请求期间发生错误，`<rpc-error>`元素将在`<rpc-reply>`消息中发送。`<rpc-error>`元素元素包含以下信息：
- error-type：定义错误发生的概念层
  - transport (layer: Secure Transport)
  - rpc (layer: Messages)
  - protocol (layer: Operations)
  - application (layer: Content)
- error-tag: 包含识别错误条件的字符串，可以参考RFC6241附录A中的定义。
- error-severity：包含标识错误严重性的字符串，由服务端确定。
  - waring
  - error
- error-app-tag：包含标识数据模型特定或特定于实现的错误条件（如果存在）的字符串。
- error-path：包含绝对XPath表达式，用于标识<rpc-error>元素中报告的错误关联的节点的元素路径。
- error-message：包含一个适合人类阅读的字符串，用于描述错误情况。
- error-info：包含协议或数据模型特定的错误内容。 如果没有为特定的错误条件提供这样的错误内容，则该元素将不存在。
```xml
<rpc-reply  xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <rpc-error>
    <error-severity>error-severity</error-severity>
    <error-path>error-path</error-path>
    <error-message>error-message</error-message>
    <error-info>...</error-info>
  </rpc-error>
</rpc-reply> 
```

#### 3.1.4 &lt;ok&gt;
如果在处理`<rpc>`请求期间没有发生错误或警告，`<ok>`元素将在`<rpc-reply>`消息中发送。
```xml
<rpc-reply message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <ok/>
</rpc-reply>
```

### 3.2 基本操作

#### 3.2.1 &lt;get&gt;
查询运行时配置和设备状态信息。

参数：
- filter：此参数指定要查询的系统配置和状态数据的部分。 如果此参数不存在，则返回所有设备配置和状态信息。<filter>元素可以包含一个“type”属性。 该属性指示<filter>元素中使用的过滤语法的类型。（例如：subtree，xpath等）

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <get>
        <filter type="subtree">
            <top xmlns="http://example.com/schema/1.2/stats">
                <interfaces>
                    <interface>
                        <ifName>eth0</ifName>
                    </interface>
                </interfaces>
            </top>
        </filter>
    </get>
</rpc>
```

#### 3.2.2 &lt;get-config&gt;
获取全部或部分指定的数据集中的配置数据。  
参数：  
- source：被查询的配置数据集的名称，例如<running/>。
- filter：此参数标识要查询的设备配置数据集的各个部分。 如果此参数不存在，则返回整个配置。

示例：
```xml
<rpc message-id="101"
    xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <get-config>
        <source>
            <running/>
        </source>
        <filter type="subtree">
            <top
                xmlns="http://example.com/schema/1.2/config">
                <users/>
            </top>
        </filter>
    </get-config>
</rpc>
```

#### 3.2.3 &lt;edit-config&gt;
该操作将全部或部分指定配置加载到指定的目标数据集中进行存储。如果目标数据集中不存在，则新建一个。

参数：
- target：正在编辑的配置数据存储的名称，例如<running/>或<candidate />。
- default-operation：选择此<edit-config>请求的默认操作。该参数是可选。其值默认是"merge"，可取值：
  - merge： <config>参数中的配置数据与目标数据集中相应级别的配置合并。
  - replace： <config>参数中的配置数据完全替换了目标数据集中的配置。
  - none：目标数据存储不受 <config>参数中的配置影响。
- test-option：只有当设备公布支持:validate:1.1能力时才能指定<test-option>元素。可选值如下：
  - test-then-set：在尝试设置之前执行验证测试。
  - set：先执行一个没有验证测试的设置。
  - test-only：仅执行验证测试，而不尝试设置。
- error-option：表示此次请求发生错误时，执行的动作，取值如下：
  - stop-on-error：停止第一个错误的<edit-config>操作。默认值。
  - continue-on-error：继续处理错误的配置数据。
  - rollback-on-error：发生错误时回滚配置。
- config：配置数据的层次结构，由服务端的一个数据模型定义。配置内容必须放置在适当的命名空间中，以允许服务端检测对应的数据模型。

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <edit-config>
        <target>
            <running/>
        </target>
        <config>
            <interface>
                <name>Ethernet0/0</name>
                <mtu>1500</mtu>
            </interface>
        </config>
    </edit-config>
</rpc>
```

#### 3.2.4 &lt;copy-config&gt;
使用另一个完整数据集中的内容创建或替换整个数据集的内容。

参数：
- target：用作<copy-config>操作的目标数据集的名称。
- source：用作<copy-config>操作源数据集的名称，或包含要复制的完整配置的<config>元素。

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <copy-config>
        <target>
            <running/>
        </target>
        <source>
            <url>https://user:password@example.com/cfg/new.txt</url>
        </source>
    </copy-config>
</rpc>
```

#### 3.2.5 &lt;delete-config&gt;
删除指定数据集的配置数据。<running/>数据集中的配置不允许删除。

参数：
- target：要删除的配置数据集的名称。

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <delete-config>
        <target>
            <startup/>
        </target>
    </delete-config>
</rpc>
```

#### 3.2.6 &lt;lock&gt;
该操作允许客户端锁定设备的整个配置数据存储系统。

参数：
- target：要锁定的配置数据集的名称。

示例：
```xml
<rpc message-id="101"  xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <lock>
        <target>
            <running/>
        </target>
    </lock>
</rpc>
```

#### 3.2.7 &lt;unlock&gt;
用于释放以前通过<lock>操作获得的配置锁。

参数：
- target：要解锁的配置数据集的名称。NETCONF客户端不允许解锁未锁定的配置数据集。

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <unlock>
        <target>
            <running/>
        </target>
    </unlock>
</rpc>
```

#### 3.2.8 &lt;close-session&gt;
客户端请求正常终止一个NETCONF会话。当服务端收到该请求时，会优雅的关闭session，并释放资源。在`<close-session>`请求之后收到的任何NETCONF请
求将被忽略。

参数：无

示例：
```xml
<rpc message-id="101"
    xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <close-session/>
</rpc>
```

#### 3.2.9 &lt;kill-session&gt;
当服务端收到该请求时，中止当前正在进行的任何操作，释放与该会话关联的任何锁和资源，并关闭任何关联的连接。

参数：
- session-id：终止的NETCONF会话的会话标识。如果该值等于当前会话ID，则返回"invalid-value"错误。

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <kill-session>
        <session-id>4</session-id>
    </kill-session>
</rpc>
```

#### 3.2.10 &lt;notification&gt;
可以通过NETCONF协议的Notification能力向客户端上报告警和事件，以便客户端及时感知服务端配置等的变更。NETCONF Client接收Server发送的Notifications的前提是向Server发送create-subscription消息
能力标识：
```xml
<capability>urn:ietf:params:netconf:capability:notification:1.0</capability>
```
请求订阅：`<create-subscription>`，订阅请求是在`<rpc>`中发起订阅。

参数：
- stream ：可选。指示感兴趣的事件流。如果不存在，将发送默认NETCONF流中的事件。
- filter：可选。定于具体的事件或通知内容。
- startTime：起始时间。
- stopTime：结束时间。

示例：
```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <create-subscription xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
    <stream>NETCONF</stream>
    <filter type="subtree">
       ...
    </filter>
    <startTime>2016-10-20T14:50:00Z</startTime>
    <stopTime>2016-10-23T06:22:04Z</stopTime>
  </create-subscription>
 </rpc>
```
订阅通知：`<notification>`

参数：
- eventTime：表示事件发生的时间。
- 通知内容，RFC不规定通知格式，由实现者定义。

示例：
```xml
<notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
  <eventTime>2016-11-29T11:57:15Z</eventTime>
  <device xmlns="http://www.seanet.com/netconf/device">
        <!-- contents here... -->
  </device>
</notification>
```

### 3.3 能力集
- **Writable-Running能力**
> urn:ietf:params:netconf:capability:writable-running:1.0

可写可运行能力(Writable-Running Capability)表示设备支持直接写入`<running>`配置数据存储。 换句话说，该设备支持以`<running>`配置为
目标的`<edit-config>`和`<copy-config>`。

- **Candidate Configuration能力**
> urn:ietf:params:netconf:capability:candidate:1.0

支持候选配置数据存储，用于保存可以在不影响设备当前配置的情况下操作的配置数据。候选配置是一个完整的配置数据集，用作创建和操作配置数据的工作场所。
可以对此数据进行添加，删除和更改以构建所需的配置数据。可以随时执行<commit>操作，使设备的运行配置设置为候选配置的值。

- **Confirmed Commit能力**
> urn:ietf:params:netconf:capability:confirmed-commit:1.1

表示服务器将支持`<commit>`操作的`<cancel-commit>`操作和`<confirmed>`，`<confirm-timeout>`，`<persist>`和`<persist-id>`参数

- **Rollback-on-Error能力**
> urn:ietf:params:netconf:capability:rollback-on-error:1.0

表示服务器将支持`<edit-config>`操作的`<error-option>`参数中的`"roll-on-error"`值。

- **Validate能力**
> urn:ietf:params:netconf:capability:validate:1.1

将配置应用于设备之前检查语法和语义错误的完整配置

- **Startup能力**
> urn:ietf:params:netconf:capability:startup:1.0

支持单独的运行和启动配置数据存储。 启动配置在启动时由设备加载。 影响运行配置的操作不会自动复制到启动配置

- **URL能力**
> urn:ietf:params:netconf:capability:url:1.0?scheme={name,…}

- **XPath能力**
> urn:ietf:params:netconf:capability:xpath:1.0
### 3.4 基本流程
场景：修改接口IP地址，并采用两阶段生效模式。  
前提：客户端触发NETCONF会话建立，完成SSH连接、完成认证和授权。
![](/images/20240508/netconf-3.png)

## 4 NETCONF在ONOS中的应用

### 4.1 层级结构
中心控制器作为NETCONF客户端，区域控制器作为NETCONF服务端，中心控制器根据配置文件config.json中配置的区域信息，定时检测区域存活状态。
当区域不存活时，中心控制器会对区域进行重连，按照先尝试连接区域控制器1，连接失败后尝试连接区域控制器2，连接失败后尝试连接区域控制器3的顺序
进行连接，如果三个都失败，则等待下一个周期进行重连。
![](/images/20240508/netconf-4.png)

### 4.2 消息处理
中心控制器与区域控制器使用同一套代码进行大包部署，在配置文件config.json中配置角色区分是中心控制器还是区域控制器。NETCONF消息处理如下：
![](/images/20240508/netconf-5.png)

### 4.3 配置下发
#### 4.3.1 配置文件下发配置
- 应用注册`listener`到`ConfigManager`；
- 中心控制器`ConfigManager`监听`config.json`变化通过`listener`回调给应用，`config.json`中的内容会缓存到cache；
- `netconf-sync`应用注册了一个`listener`，所有应用配置变化都会通知到该`listener`，该`listener`收到变化后，会组装成netconf消息，发送到区域控制器；
- `netconf-sync`应用注册了一个区域控制器上线的`listener`，收到上线后会调用`ConfigManager`的`getConfig`接口，获取所有应用配置，并下发到区域控制器；
- `netconf-sync`应用注册了监听`common`的接口，监听到`common`配置变化，会将`common`配置写入mongodb；
- 区域控制器`ConfigManager`中有一个netconf消息的`listener`，监听netconf请求，此`listener`处理`edit-config`请求；将收到的应用配置，
  通过`ConfigManager`中的`listener`通知到区域控制器的应用，并将应用配置更新到`cacheConfig`缓存的json节点中。

![](/images/20240508/netconf-6.png)

netconf消息定义：
1. 下发配置  

```xml
<rpc message-id="60" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
  <edit-config>
    <target>
      <running/>
    </target>
    <default-operation>replace</default-operation>
    <error-option>rollback-on-error</error-option>
    <config>
      <items>
        <item name="应用名称">
          应用全量json配置信息，CDATA
        </item>
      </items>
    </config>
  </edit-config>
</rpc>
```

2. 创建订阅

```xml
<rpc message-id="101" xmlns="urn:ietf:params:xml:ns:netconf:base:1.0">
    <create-subscription
        xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
        <stream>NETCONF</stream>
        <filter type="subtree">
            <items>
                <item name="订阅名称" xmlns="订阅名称所在的命名空间">

                <item/>           
            </items>
        </filter>
    </create-subscription>
</rpc>
```

3. 事件通知

```xml
<notification xmlns="urn:ietf:params:xml:ns:netconf:notification:1.0">
    <eventTime>2024-11-26T13:51:00Z</eventTime>
    <订阅名称 xmlns="订阅名称所在的命名空间"> 
        <!--content here    -->
    </订阅名称>
</notification>
```
#### 4.3.2 CLI下发配置
中心控制器通过CLI配置的信息，由中心控制器通过CLI代理，下发到区域控制。
![](/images/20240508/netconf-7.png)
