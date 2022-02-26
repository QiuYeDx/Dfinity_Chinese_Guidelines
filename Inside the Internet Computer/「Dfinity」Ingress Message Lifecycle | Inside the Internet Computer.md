# 「Dfinity」Ingress Message Lifecycle | Inside the Internet Computer

> 原创，转载请注明出处及作者。
> 
> <cite>——2022.2.25 by QiuYeDx</cite>

## 前言

本文将讲述Internet Computer的消息生命周期(***Ingress Message Lifecycle***)。

阅读完本文，您将了解Internet Computer如何处理update方法调用，（从它到达一个节点直到它被执行的全过程）。我们将尤其讨论必须满足的条件以及我们如何确保对相关方的处理收费。

## 正文

Internet Computer托管包含代码和数据的容器(***Canister***)，并且用户可以通过向运行Internet Computer的协议的节点发送query和update调用来与这些Canister进行交互。

query调用可以由参与托管目标容器的子网的任何节点应答，且不会更改Canister的状态。而update调用则可以改变Canister的状态，因此，节点必须通过共识算法就它们的有效性和它们的执行顺序达成一致。

当在Internet Computer节点收到有效的update调用请求时，该节点的***HTTP Handler***将会创建一个入口消息(***Ingress Message***)，并把它提交给副本(***Replica***)处理。在到达它的目标canister之前，入口消息将被若干个软件组件处理。（如Picture1.1）

### Ingress Message Lifecycle概述

首先，P2P确保消息被广播到足够多的其他的节点中；在某一时刻，共识(***Consensus***)形成一个包含消息的区块(***Block***)；一旦经过公证和最终确认(*Notarized and Finalized*)，它就会被传递到消息路由(***Routing***)，在那里，如果它满足有效集的规则(***The Valid Set Rule***)，它将会进入到***Induction Pool***；在Induction Pool中时，入口消息可能会花费一些时间等待它的目标Canister的队列，直到调用程序(***Scheduler***)选择该消息使其执行完毕(*Executed*)。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/1.1.PNG" alt="IMG_3924" class="wp-image-947"><figcaption>Picture 1.1</figcaption></figure>

### 入口消息(*Ingress Message*)的特性

和Canister间的消息传递（从一个Canister发送到另一个Canister）相反，入口消息的发送者是用户(***User***)，而这两种情况下，目标都是一个Canister。处理消息需要花费时间，因此，用户不能直接被告知消息是否被导入(*Inducted*)并执行(*Executed*)以及执行的结果是什么。取而代之的是用户可以从入口消息历史记录(***Ingress Message History***)中获取此信息，获取结果可能额外需要一个query调用。为了防止用户无止境地等待消息被执行（花费无限长的时间），入口消息可能会超时(***Expire***)。最后很重要的是，入口消息要被签名以保证其真实性(*Authenticity*)。特性总结起来，如Picture 1.2，即：

- 发送者不是Canister而是一名用户；
- 接受者是一个Canister；
- 结果需要花费时间，需要通过消息历史获取；
- 会超时；
- 被签名。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/1.2.PNG" alt="IMG_3928" class="wp-image-951"><figcaption>Picture 1.2</figcaption></figure>

所有这些信息都是update调用请求的一部分。***HTTP Handler***将此请求转换为入口消息，然后将其传递给入口管理(***Ingress Manager***)。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/1.3.PNG" alt="IMG_3929" class="wp-image-952"><figcaption>Picture 1.3</figcaption></figure>

### 入口消息的字段(Ingress Message Fields)

入口消息具有以下字段：

- 发送者（UserID）；
- 接收者（目标CanisterID）；
- Expiry（包含该消息可以被导入到***Induction Pool***中的最晚IC时间。换句话说，一旦IC时间超过了这一时刻，用户就知道IC将永远不会处理此消息。由于操作和存储的花费将会从一个实体被收取，到期时间最多只能是个常数而不会无限大，否则将会产生无限大的成本）；
- 签名（Signature and Public Key for UserID）；
- 方法名（要调用的方法的名称）；
- 方法负载（参数）；
- Nonce（随机数，用于区分其他对相同方法的调用消息）。

此外，入口消息据有一个请求ID(***Request ID***)，该ID由除了签名以外的所有字段的哈希组成。基于此ID，用户可以向IC查询该请求的状态是什么。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/1.4.PNG" alt="IMG_3930" class="wp-image-953"><figcaption>Picture 1.4</figcaption></figure>

这些字段的详细信息在下文进行了详细描述。

### Mechanism for Canister-Controlled Rate-Limiting of Ingress Messages

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/2.1.PNG" alt="IMG_3931" class="wp-image-954"><figcaption>Picture 2.1</figcaption></figure>

Internet Computer最初使用***Canister Paste***模型。在这个模型中，Canister对所有IC所需要的资源进行“买账”来使Canister持续运作，存储、计算、网络资源等一切花费在运行Internet Computer协议上的资源都被考虑在内。由于Canister要为处理不受Canister影响的入口消息所消耗的资源进行“买账”，Internet Computer为Canister提供了限制入口消息花费的机制。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/2.2.PNG" alt="IMG_3932" class="wp-image-955"><figcaption>Picture 2.2</figcaption></figure>

在基础版本的机制中，Canister可以公开一种方法(***Method***)来检查入口消息及其字段并作出决定，看看此消息应被进一步处理还是立即被拒绝。为了作出此决定，可以考虑Canister的状态或元数据(***Metadata***)。例如，一个Canister可以仅当自身的Cycles余额大于某个值时才接受该消息；或者它可以检查消息的发送者是否是授权列表的成员。注意，执行这个用来确定是否应保留入口消息的方法基本上是一个query调用。换句话说，容器状态不会被它改变。

为了确保消息可以被执行以及有人可以成功为之“买账”，在消息执行前，必须满足以下所有条件。（如Picture 2.3）请注意，这实际上是在我们**执行**消息之前，而不是我们在子网中发送消息之前。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/2.3.PNG" alt="IMG_3935" class="wp-image-958"><figcaption>Picture 2.3</figcaption></figure>

首先，格式必须正确，所有必要的字段必须存在，它们必须是正确的类型；请求ID必须是正确的，消息大小必须在**Registry中明确的限制大小**以内。然后我们需要确保消息在被导入(***Inducted***)时没有超时，并且我们需要确保它花费的时间小于***Induction Pool***中给定的时间常量。我们不希望入口消息被意外地执行两次。所以如果我们有一条入口消息，这个入口消息的请求ID和已经被导入到***Induction Pool***中的某个请求的请求ID相同，那么这条后来的消息则不会进入***Induction Pool***。其他检查包括WASM检查等。例如，Canister真的具有入口消息想要调用的方法，并且参数格式是正确的。这里可能包含一些对特定方法的更严格的大小检查。`canister_inspect_message()`方法是非公开的，当随着消息被调用时会返回accept。为了消息能够执行，它的目标Canister必须有足够的Cycles余额，并且必须有足够的导入和执行能力(*Induction and Execution Capacity*)。最后，请求ID上的签名必须被签证。

现在，我们知道了执行消息必须满足的所有条件，那么让我们跟随一条入口消息经历一遍它从开始到最终被执行的整个过程，看一看在不同位置所进行的不同的必要检查吧。

### 入口消息验证检查(Ingress Message Validation Checks)

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/3.1.PNG" alt="IMG_3939" class="wp-image-962"><figcaption>Picture 3.1</figcaption></figure>

update调用请求被提交到的第一部分是***HTTP Handler***，这也是第一个我们可以进行一些检查的地方。如果通过了它的检查，消息将会被添加到***Artifact Pool***并且通过P2P协议进行传播。当接收到来自另一个节点的入口消息时（当它不是直接来自用户时），入口管理(***Ingress Manager***)会在将消息添加到***Artifact Pool***执勤执行一组检查。因为发送消息的节点可能是恶意的(*Malicious*)。当消息被添加到***Artifact Pool***之后，它将会被传播到其他节点。当区块创建者(***Block Maker***)创建一个新的区块提案(***Block Proposal***)时，入口消息管理选择一组适合新提案的入口消息，然后进行广播。在接收到一个提案时，节点会在对消息进行公证之前运行另一组检查。当区块已经最终确定(*Finalized*)并且它的消息被传递到消息路由组件(***Message Routing Component***)时，它们需要在被插入到***Induction Pool***之前通过有效集规则(***The Valid Set Rule***)。重要的是，最后的一组检查将会在消息被执行(*Executed*)之前进行。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/3.2.PNG" alt="IMG_3941" class="wp-image-964"><figcaption>Picture 3.2</figcaption></figure>

在我们深入了解每个检查的细节之前，我们要知道的是，消息的某些条件不可能在处理流程的早期就获得。当不得不决定入口消息是否应该被添加到***Artifact Pool***时，节点只能使用当前可用的信息，特别是节点此时无法知道消息将在哪个块中结束，并且目标Canister的状态将会被它们如何改变。例如可能发生的是，将包含消息的块之前的块中的消息正在更改容器状态，从而导致消息最终被拒绝。然而，在必须立即作出决定时，无论机制设计得有多复杂，节点都无法知道届时会发生什么。暂时不去理会其中的机制有多复杂，总是会有某些情况，在这些情况下，一个节点做出了错误的决定，或者拒绝了一条之后会被接受的消息，再或者将一条消息添加到它的池中，即使它以后无法被添加到区块中。因此，在**Gossip**之前会尽可能少地进行验证，并且将会仅丢弃那些从来不会进去区块的消息。注意，我们确保在进行选择和公证检查(*Selection and Notarization Checks*)之后，最终在区块中没有错误消息。

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/3.3.PNG" alt="IMG_3945" class="wp-image-968"><figcaption>Picture 3.3</figcaption></figure>

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/3.4.PNG" alt="IMG_3946" class="wp-image-969"><figcaption>Picture 3.4</figcaption></figure>

一些检查独立于节点的当前状态，所以总是在早期运行这些检查，例如格式化和签名验证。一些其他检查确实取决于节点当前的状态，但它们是单调的(*Monotonic*)，即如果一条消息在某个时刻是无效的，那么它在未来将会一直保持无效的状态。对于这些检查，采用本地可用状态(*The Locally Available State*)是可以的。对于剩下的检查，在两个方向上都可能是错误的(*For the remaining checks it is possible to be wrong in both directions*.)。依据Picture 3.3中列出的原则，当接收到一条来自用户的消息时，我们将会采用本地可用状态，仅仅接受这样的事实：一些决定可能是错误的。如果收到一条来自对等方(***Peer***)消息，将**不会**根据本地可用状态重复这些检查。换句话说，该节点相信其他节点的决定，并且我们接受这样一个事实，即如果我们那样做（指重复检查），我们可能会浪费一些资源。

我们现在将更详细地了解我们目前正在执行的检查集，在未来不久，我们可能会有更多的检查以加强这个平台。

## 总结

<figure class="wp-block-image size-large"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/4.1.PNG" alt="IMG_3951" class="wp-image-974"><figcaption>Picture 4.1</figcaption></figure>

C0：第一次检查是在HTTP Handler处执行。当收到用户请求时，它将会对请求格式进行初步检查。这些检查不需要其他组件重复，因为反序列化机制(***Deserialization Mechanism***)将会确保只有正确格式的消息才能传递到入口管理。此外，***HTTP Handler***处理关于目前可用的最新执行高度、时间和最新的可用***Registry***版本的检查。因此，我们检查目标Canister是否实际安装在子网上，我们检查方法和***Payload***是否正确，如果签名验证通过并且***Expiry***在不远的将来。最后我们还要确保消息不在入口历史记录(***Ingress History***)中且不在***Artifact Pool***中。如果检查方法返回"decline"，那么我们将丢弃这些消息，否则消息就通过所有这些检查，并且被添加到***Artifact Pool***，在子网中传播。

C1：如果入口管理组件(***Ingress Manager Component***)从另一个节点收到一条消息，它将会检查该消息是否适合进一步广播。当且仅当入口消息满足以下所有条件时，它才会被发送给其他节点：我们再次检查签名，以避免在出块时以及检测其他错误时浪费时间。我们检查***Expiry***并确保消息尚未在***Artifact Pool***中或在***Inductive Payloads***。

C2 = C3：当被出块组件(***Block Maker Component***)的共识所“敦促”(***Prompted***)时，入口管理(***Ingress Manager***)选择一组有效的入口消息作为区块有效负载(***Block Payloads***)的一部分。当被公证组件(*Notary Component* ***of Consensus***)触发时，这个组件也使来自对等者(***Peer***)的区块提案的入口负载(***Ingress Payload of Block Proposals***)生效。在这种情况下，提案的入口负载(***Ingress Payload of the Proposal***)和链拓展的区块需要被考虑。**出块和公证的有效条件是一样的。**这两者都查阅出存在区块中的执行高度和***Registry***版本。这些信息在所有节点上都可以用来**一致地**确认入口消息的有效性。例如，所有节点都可以依据区块中的时间确认Expiry在正确的范围内，无论本地挂钟时间(*The Local Wall Clock Time*)是多少。

C4：当一批处理从共识的最终确认组件(***The Finalization Component of Consensus***)被传递到消息路由时，这批处理的负载中的入口消息被提交给有效集规则(***Valid Set Rule***)来进行容量和Cycles的检查，之后才能进入到***Induction Pool***中。其他检查不需要重复，由于对最终确认(***Finalization***)来说必要的、节点的阈值，它们被假定保持不变。已经导入，但长时间未被处理的消息会被丢弃。

C5：当入口消息被消息路由中的消息调度程序(***Message Scheduler***)选择时，最后一次的Cycles检查将会在它们到达目标Canister之前被执行。

我们可以看到，入口消息需要通过大量必要的检查。因此，能够为Canister支付计算、通信等消耗的Cycles是必要的。并且，同时可以阻止用户和恶意节点造成过多的资源浪费。

感谢您的阅读！

<figure class="wp-block-image size-large is-style-circle-mask"><img src="src/Ingress Message Lifecycle|Inside the Internet Computer/end.PNG" alt="IMG_3957" class="wp-image-980"></figure>
