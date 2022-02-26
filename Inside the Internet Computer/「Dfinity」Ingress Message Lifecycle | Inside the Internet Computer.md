# 「Dfinity」Ingress Message Lifecycle | Inside the Internet Computer

> 原创，转载请注明出处及作者。
> 
> <cite>——2022.2.25 by QiuYeDx</cite>

## 前言

本文将讲述Internet Computer的消息生命周期(***Ingress Message Lifecycle***)。

阅读完本文，您将了解Internet Computer如何处理`update`方法调用，（从它到达一个节点直到它被执行的全过程）。我们将尤其讨论必须满足的条件以及我们如何确保对相关方的处理收费。

## 正文

Internet Computer托管包含代码和数据的容器(***Canister***)，并且用户可以通过向运行Internet Computer的协议的节点发送query和update调用来与这些Canister进行交互。

query调用可以由参与托管目标容器的子网的任何节点应答，且不会更改Canister的状态。而update调用则可以改变Canister的状态，因此，节点必须通过共识算法就它们的有效性和它们的执行顺序达成一致。

当在Internet Computer节点收到有效的update调用请求时，该节点的***HTTP Handler***将会创建一个入口消息(***Ingress Message***)，并把它提交给副本(***Replica***)处理。在到达它的目标canister之前，入口消息将被若干个软件组件处理。（如Picture1.1）

### Ingress Message Lifecycle概述

首先，P2P确保消息被广播到足够多的其他的节点中；在某一时刻，共识(***Consensus***)形成一个包含消息的区块(***Block***)；一旦经过公证和最终确认(*Notarized and Finalized*)，它就会被传递到消息路由(***Routing***)，在那里，如果它满足有效集的规则(***The Valid Set Rule***)，它将会进入到***Induction Pool***；在Induction Pool中时，入口消息可能会花费一些时间等待它的目标Canister的队列，直到调用程序(***Scheduler***)选择该消息使其执行完毕(*Executed*)。

<figure class="wp-block-image size-large"><img src="https://qiuyedx.com/wp-content/uploads/2022/01/IMG_3924-1024x573.png" alt="IMG_3924" class="wp-image-947"><figcaption>Picture 1.1</figcaption></figure>

### 入口消息(*Ingress Message*)的特性

和Canister间的消息传递（从一个Canister发送到另一个Canister）相反，入口消息的发送者是用户(***User***)，而这两种情况下，目标都是一个Canister。处理消息需要花费时间，因此，用户不能直接被告知消息是否被导入(*Inducted*)并执行(*Executed*)以及执行的结果是什么。取而代之的是用户可以从入口消息历史记录(***Ingress Message History***)中获取此信息，获取结果可能额外需要一个query调用。为了防止用户无止境地等待消息被执行（花费无限长的时间），入口消息可能会超时(***Expire***)。最后很重要的是，入口消息要被签名以保证其真实性(*Authenticity*)。特性总结起来，如Picture 1.2，即：

- 发送者不是Canister而是一名用户；
- 接受者是一个Canister；
- 结果需要花费时间，需要通过消息历史获取；
- 会超时；
- 被签名。

<figure class="wp-block-image size-large"><img src="https://qiuyedx.com/wp-content/uploads/2022/01/IMG_3928-1024x573.png" alt="IMG_3928" class="wp-image-951"><figcaption>Picture 1.2</figcaption></figure>

所有这些信息都是update调用请求的一部分。***HTTP Handler***将此请求转换为入口消息，然后将其传递给入口管理(***Ingress Manager***)。

<figure class="wp-block-image size-large"><img src="https://qiuyedx.com/wp-content/uploads/2022/01/IMG_3929-1024x573.png" alt="IMG_3929" class="wp-image-952"><figcaption>Picture 1.3</figcaption></figure>

### 入口消息(*Ingress Message*)的字段(*Fields*)

入口消息具有以下字段：

- 发送者（UserID）；
- 接收者（目标CanisterID）；
- Expiry（包含该消息可以被导入到***Induction Pool***中的最晚IC时间。换句话说，一旦IC时间超过了这一时刻，用户就知道IC将永远不会处理此消息。由于操作和存储的花费将会从一个实体被收取，到期时间最多只能是个常数而不会无限大，否则将会产生无限大的成本）；
- 签名（Signature and Public Key for UserID）；
- 方法名（要调用的方法的名称）；
- 方法负载（参数）；
- Nonce（随机数，用于区分其他对相同方法的调用消息）。

此外，入口消息据有一个请求ID(***Request ID***)，该ID由除了签名以外的所有字段的哈希组成。基于此ID，用户可以向IC查询该请求的状态是什么。

<figure class="wp-block-image size-large"><img src="https://qiuyedx.com/wp-content/uploads/2022/01/IMG_3930-1024x573.png" alt="IMG_3930" class="wp-image-953"><figcaption>Picture 1.4</figcaption></figure>

这些字段的详细信息在下文进行了详细描述。

### Mechanism for Canister-Controlled Rate-Limiting of Ingress Messages

<figure class="wp-block-image size-large"><img src="https://qiuyedx.com/wp-content/uploads/2022/01/IMG_3931-1024x573.png" alt="IMG_3931" class="wp-image-954"><figcaption>Picture 2.1</figcaption></figure>

Internet Computer最初使用***Canister Paste***模型。在这个模型中，Canister对所有IC所需要的资源进行“买账”来使Canister持续运作，存储、计算、网络资源等一切花费在运行Internet Computer协议上的资源都被考虑在内。由于Canister要为处理不受Canister影响的入口消息所消耗的资源进行“买账”，Internet Computer为Canister提供了限制入口消息花费的机制。

<figure class="wp-block-image size-large"><img src="https://qiuyedx.com/wp-content/uploads/2022/01/IMG_3932-1024x573.png" alt="IMG_3932" class="wp-image-955"><figcaption>Picture 2.2</figcaption></figure>
