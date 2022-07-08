# Searport Analysis

## 概述

Seaport 的交易模型跟之前的 Wyvern Protocol 一样,依旧是中央订单簿的交易模型。都是由链下的中心化的订单簿和链上的交易组成。

![order-book](order.png)

其中链下的订单簿负责存储用户的挂单信息，并对订单进行撮合。最终的成交和转移 NFT 是由 Seaport Protocol 来负责的。

### 对比

我们都知道 Opensea 之前用的合约是 Wyvern Protocol。但是随着生态的发展 Wyvern Protocol 的各种问题开始暴露了出来。

1. 多年前开发的，代码最后维护的时间都是4年前了。
2. 不是专门为 Opensea 专门量身定做的，有大量的资源浪费。
3. gas 消耗多，280000左右（单个）。
4. 不支持直接使用代币购买 NFT，购买的时候需要转换。进一步提高了gas消耗和购买成本。

也正是为了解决这些问题 Seaport Protocol 被开发出来了。他有以下特性：

1. 支持批量购买，批量卖出。
2. 支持不同币种来购买。
3. 支持混合支付，比如订单可以设置成以一些 ERC20 代币加上某些 ERC721 代币来支付。主要目的是去除不同币种相互转换带来的损耗，进一步降低用户的额外费用。
4. 使用内联汇编来降低 gas 消耗，目前测试来看是 134000左右（单个）

我们可以看出 Seaport Protocol 最重要的目的就是降低用户的 gas 消耗并且降低购买流程的复杂度。

## 结构

```mermaid
graph TD
    Offer & Consideration --> Order
    zone & conduitKey --> Order

    subgraph Seaport[ ]
    Order --> Fulfill & Match
    Order --> Validate & Cancel
    end

    Validate --> Verify
    Cancel --> OrderStatus

    Fulfill & Match --> OrderCombiner --> OrderFulfiller

    OrderCombiner --> BasicOrderFulfiller --> OrderValidator
    OrderCombiner --> FulfillmentApplier

    OrderFulfiller --> CriteriaResolution
    OrderFulfiller --> AmountDeriver
    OrderFulfiller --> OrderValidator
    
    OrderValidator --> ZoneInteraction
    OrderValidator --> Executor --> TokenTransferrer
    Executor --> Conduit --> TokenTransferrer
    
    Executor --> Verify
    

    subgraph Verifiers[ ]
    Verify --> Time & Signature & OrderStatus
    end
```

上图是官方给出的结构解析图。

大致可以分为以下几个部分：

1. Order，订单相关数据结构
2. Order Fulfillment，订单撮合成交的方法
3. Verify，订单的校验
4. Transferrer，token 的转移

我们下面按照这个顺序来进行分析。

## Order

这里先分析 order 的数据结构，一些功能的具体实现后面再详细叙述。

![Seaport](Seaport.order.drawio.svg)

### 1. offer 和 consideration

目前大多数 NFT 市场只允许一方同意提供 NFT，另一方同意提供支付代币的 listing。Seaport 采取了一种不同的方法：`offerer` 可以同意提供一定数量的 ETH/ERC20/ERC721/ERC1155 项目--这就是 `offer`。为了使该`offer`被接受，`offerer`必须收到一定数量的物品，这就是 `consideration`。

也就是说 Seaport 中没有明确的买家和卖家的概念了。

1. 当 offer 是 ERC721/ERC1155 的时候，offerer 就是卖家，卖出 ERC721/ERC1155 来换取 ETH/ERC20。
2. 当 offer 是 ETH/ERC20 的时候，offerer 就是买家，买入 ERC721/ERC1155，支付 ETH/ERC20。
3. 当 offer 和 consideration 都是 是 ERC721/ERC1155 的时候就是 NFT 的互换了，这种情况下就无所谓买家和卖家了。

需要注意的是 **ETH/ERC20 在 Seaport 中充当的是货币（currency），ERC721/ERC1155 充当的是商品。** 这一点很重要。

还有一点需要注意的是 order 中 offer 和 consideration 都是数组类型。数组的元素个数可以是 0 。也就是说 订单中 offer 或者 consideration 中的一个可以是空的。

```solidity
// ConsiderationStructs.sol => OfferItem

struct OfferItem {
    ItemType itemType;
    address token;
    uint256 identifierOrCriteria;
    uint256 startAmount;
    uint256 endAmount;
}

struct ConsiderationItem {
    ItemType itemType;
    address token;
    uint256 identifierOrCriteria;
    uint256 startAmount;
    uint256 endAmount;
    address payable recipient;
}
```

#### 1) itemType

```solidity
// ConsiderationEnums.sol => ItemType

enum ItemType {
    // 0: ETH on mainnet, MATIC on polygon, etc.
    NATIVE,

    // 1: ERC20 items (ERC777 and ERC20 analogues could also technically work)
    ERC20,

    // 2: ERC721 items
    ERC721,

    // 3: ERC1155 items
    ERC1155,

    // 4: ERC721 items where a number of tokenIds are supported
    ERC721_WITH_CRITERIA,

    // 5: ERC1155 items where a number of ids are supported
    ERC1155_WITH_CRITERIA
}
```

需要注意的是 ERC721_WITH_CRITERIA 和 ERC1155_WITH_CRITERIA。表示基于标准的 ERC721/ERC1155。

配合 identifierOrCriteria 可以用来表示一个标准，代表一个或者多个 NFT的集合。

还有一点需要注意的是在 `OrderFulfilled` 事件中 `ERC721_WITH_CRITERIA` 和 `ERC1155_WITH_CRITERIA` 类型的元素会被转换为对应的 `ERC721` 和 `ERC1155` 类型。

#### 2) token

token 的合约地址，空地址表示原生代币。

#### 3) identifierOrCriteria

对于 ETH/ERC20 类型的 token，这个属性会被忽略。

对于 ERC721/ERC1155 类型的项目，表示 token id。

对于 ERC721_WITH_CRITERIA/ERC1155_WITH_CRITERIA 类型的项目， 该值表示的是一个 merkle tree 的 root 值。这个 merkle tree 由一个指定的 token id 集合来生成。

简单来说就是卖家拥有一个 collection 的多个 NFT。这时候他就可以提供一个 offer，这个 offer 里的 identifierOrCriteria 是通过他所拥有的 NFT 对应的 token id 组成的集合生成的。这样这个 offer 就包含了他拥有的所有 NFT 对应的 token id 的信息。买家在交易的时候可以选择某一个 token id 的 NFT 进行交易，如果验证通过就能进行成交。具体实现过程后面再详细说明。

identifierOrCriteria 可以为 0，表示买家提供该 collection 下所有 NFT 的 offer。买家在成交的时候可以选择该 collection 下 offerer 拥有的任意 NFT。

需要注意的是，这个特性在 Opensea 目前的前端界面中还不支持。

#### 4) startAmount 和 endAmount

普通情况下这两个值是相同的。表示 token 的数量。

在进行荷兰式拍卖和英格兰式拍卖的时候，这两个值是不同的。然后订单根据当前这个两个属性并结合当前区块时间以及订单的 startTime、endTime 来确定当前的价格。

```solidity
// AmountDeriver.sol =>  _locateCurrentAmount()

// Derive the duration for the order and place it on the stack.
duration = endTime - startTime;

// Derive time elapsed since the order started & place on stack.
elapsed = block.timestamp - startTime;

// Derive time remaining until order expires and place on stack.
remaining = duration - elapsed;
```

#### 5) recipient

consideration 需要指明 recipient。代表 token 的接收地址。

### 2. offerer

order 里的 offer 的提供者。

需要注意的是，订单要是想要成交或者说想要转移 offerer 提供的 offer，必须是下面三个情况中的一个。

1. msg.sender == offerer，也就是 offerer 自己操作
2. 通过签名进行授权，标准的 65-byte EDCSA, 64-byte EIP-2098, 或者通过 EIP-1271 isValidSignature 的校验。大部分情况下是这种。
3. offerer 通过调用 validate() 进行链上挂单的。这种情况下订单成交的时候可以跳过签名的校验。

```solidity
// Consideration.sol => validate()

function validate(Order[] calldata orders)
        external
        override
        returns (bool validated)
    {
        // Validate the orders.
        validated = _validate(orders);
    }
```

### 3. startTime 和 endTime

订单的开始和结束时间。还可以配合 offer 或者 consideration 的 startAmount/endAmount 来确定拍卖的价格。

### 4. orderType

订单有四种类型，是两种类型选项的组合。

#### 1) FULL 和 PARTIAL

`FULL` 表示订单不支持部分填充，而 `PARTIAL` 允许填充订单的某些部分。部分填充一般用于 ERC1155 的 NFT。买方可以购买 offer 中的一部分。

#### 2) OPEN 和 RESTRICTED

`OPEN` 表示执行订单的调用可以由任何帐户提交，而 `RESTRICTED` 是限制性的订单，要求由 offerer 或订单 zone 执行订单，或者调用 zone 的 `isValidOrder`或 `isValidOrderIncludingExtraData` 方法来查看返回的 magic value。这个 magic value 表示订单是否已被批准。

```solidity
// ConsiderationEnums.sol => OrderType

enum OrderType {
    // 0: no partial fills, anyone can execute
    FULL_OPEN,

    // 1: partial fills supported, anyone can execute
    PARTIAL_OPEN,

    // 2: no partial fills, only offerer or zone can execute
    FULL_RESTRICTED,

    // 3: partial fills supported, only offerer or zone can execute
    PARTIAL_RESTRICTED
}
```

### 5. zone

zone 是可选的辅助帐户，一般情况下是一个合约。

具有两个附加权限：

1. zone 可以通过调用 `cancel` 来取消其命名为 zone 的订单。（请注意，offerer 也可以取消自己的订单，可以单独取消，也可以通过调用 `incrementCounter` 立即取消与当前计数器签署的所有订单）。
2. `RESTRICTED` 类型的订单必须由 zone 或 offerer 执行，或者调用 zone 的 `isValidOrder`或 `isValidOrderIncludingExtraData` 方法来查看返回的 magic value。这个 magic value 表示订单是否已被批准。

简单来说 zone 是在成单前做额外校验的，并且可以取消 offerer 的 listing。offerer 可以利用 zone 来做一些交易过滤相关的操作。

### 6. zoneHash

zoneHash 代表一个任意的 32 字节的值，在履行限制性订单时，该值将被提供给 zone ，zone 在决定是否授权该订单时可以利用该值。

### 7. conduitKey

conduitKey 是一个 bytes32 的值，表示在执行代币转移时，应该利用什么 conduit（渠道） 作为代币批准的来源。默认情况下（即当 conduitKey 被设置为0时），offerer 将直接向 Seaport 授权 ERC20、ERC721和ERC1155代币，这样它就可以在执行期间执行订单指定的转移。相反，offerer 将代币批准给对应的 conduit，然后 Seaport 将指示该 conduit 转移各自的代币。

conduitKey 跟 conduit 是密切关联的。项目方，拥有者或者平台可以通过 conduit 来管理 nft 的交易。提供了更大的灵活性。

### 8. counter

计数器，要与 offerer 的计数器相同。

offerer 可以通过调用 `incrementCounter` 立即取消与当前计数器签署的所有订单。

## Order Fulfillment

在 Wyvern Protocol 中所有的订单都是通过 `orderMatch` 方法来最终成交的。Seaport 与之不同，提供了 7 个方法来执行交易。

这些方法可以分为两类：

### fulfill

包括所有以 fulfill 开头的方法。

1. fulfillBasicOrder                    => 0xfb0f3ee1
2. fulfillOrder                         => 0xb3a34c4c
3. fulfillAdvancedOrder                 => 0xe7acab24
4. fulfillAvailableOrders               => 0xed98a574
5. fulfillAvailableAdvancedOrders       => 0x87201b41

调用这些方法的时候，第二个隐含订单将被构建，调用者为 offerer，将要履行的订单的 offer 为 consideration，将要履行订单的 consideration 为 offer。将要履行订单的 offer 项目将从订单的 offerer 转移到履行者，然后所有 consideration 项目将从履行者转移到指定的接受者。

目前大部分成交都调用的这些方法。

### match

包括 match 开头的方法。

1. matchOrders                  => 0xa8174404
2. matchAdvancedOrders          => 0x55944a42

这些方法将多个订单（大于等于 2）匹配起来，Seaport 将简单地确保每个订单的需求重合。

以这种方式履行的订单没有一个明确的履行者。交易成功的事件中 recipient 为空。

目前发现的有两种使用场景：

1. 在挂单的时候指定某个地址购买。这个指定的地址在购买的时候。
2. 在某个 NFT 的 offer 列表里点击 counter 进行议价的时候。

后面应该会有更多的使用场景。

### Advanced

Advanced 分两个维度

一个是订单：订单分为 Order / AdvancedOrder

二是方法：方法也有 普通方法 和 Advanced 方法

总结下来 Advanced 主要体现在下面几点：

#### 1. 部分填充（Partial fills）

上面我们了解到 orderType 中有 FULL 和 PARTIAL 的分别。 FULL 表示完全填充，也就是买方必须购买全部的 offer。 PARTIAL 则表示允许买方购买部分的 offer。

因此在使用部分填充的时候 orderType 必须是 1 或者 3。

AdvancedOrder 相较于 Order 多了 numerator（分子）和 denominator（分母）两个参数。通过他们计算出来一个比率，这个比率就是买方希望获取数量占 offer 总数量的比率。分子、分母都是 1 则表示要全部购买。（其实在 Advanced 的方法中 Order 都会被转化为 AdvancedOrder，分子、分母都是 1）

一个 PARTIAL 类型的 offer 支持多次部分填充。也就是说如果一次交易没有完全卖出，订单还是有效的。后续还可以进行购买。

有几点需要注意：

1. 部分填充的比率是原订单 offer 数量为基准的。
   1. 比如说 offer 中有 10 个。
   2. 第一次成交 3/10，还剩余 7 个。
   3. 第二次成交 1/2 ，买方获得的数量是 `1 / 2 * 10 = 5`。
2. 如果总数量是 amount，那么 `amount * (numerator / denominator)` 必须是个整数，否则会报 `InexactFraction` 的错误。
3. 如果买方想要获取的比率乘以总数量大于剩余数量的话将会获取剩余的全部。
4. 如果 numerator > denominator 或者 numerator == 0，会报 `BadFraction` 的错误。

#### 2. 基于标准的订单（Criteria-based orders）

前面介绍 `identifierOrCriteria` 的时候，我们大致了解了基于标准的订单的基本逻辑。现在看看是怎么实现的。

这里面涉及到一个我们都很熟悉的数据结构： Merkle Tree。

![merkle](Seaport.merkle.drawio.svg)

我们通过一个例子来具体说明实现过程。

1. 首先一个 offerer 拥有某个 collection 的三个 ERC721 token。tokend id 分别是 1、2、6。

2. offerer 挂出他的 token。挂单的时候要生成一个 Merkle Tree。Data1 = 1， Data1 = 2， Data1 = 6。最终得到 Root 的值。然后生成订单信息，offer 的 `identifierOrCriteria` = `Root`。
3. 某个买家想要购买 tokend id 为 1 的 token。需要调用 Advanced 方法。
4. Advanced 方法需要传入 criteriaResolvers 参数。具体类型如下

```solidity
// ConsiderationStructs.sol => CriteriaResolver

struct CriteriaResolver {
    uint256 orderIndex; // 用于一次成交多个订单的情况，表明需要校验哪个订单
    Side side;  // offer 或者 consideration，具体订单中的哪一方
    uint256 index; // offer 或者 consideration 中的元素索引，找出具体哪个元素
    uint256 identifier; // 想要成交的 tokend id， 上面例子中就是 1
    bytes32[] criteriaProof; // 证明所需的数据，比如上面的例子中就是 Node12 和 Node22 组成的集合。
}
```

5. 执行订单的时候要先进行证明，证明买家要买的 token 包含在 offerer 的 offer 之中。

```solidity
// CriteriaResolution.sol => _verifyProof()

/**
     * @dev Internal pure function to ensure that a given element is contained
     *      in a merkle root via a supplied proof.
     *
     * @param leaf  The element for which to prove inclusion. 上面的例子中就是 1
     * @param root  The merkle root that inclusion will be proved against. offer 中的 identifierOrCriteria 
     * @param proof The merkle proof. 买方成单的时候传入的 criteriaProof
     */
    function _verifyProof(
        uint256 leaf,
        uint256 root,
        bytes32[] memory proof
    ) 
    // 省略具体代码
    // 大致证明逻辑如下
    // 1. 根据 leaf 生成 Node11
    // 2. 根据 Node11 和传入的 Node12 生成 Node21
    // 3. 根据 Node21 和传入的 Node22 生成 Root
    // 4. 如果 Root == root 就说明通过了证明
```

6. 将 offerer 对应 token id 的 token 转移给买家，完成交易。

还有一点需要注意的是 如果 offerer 的 offer 中 identifierOrCriteria 是 0 的话，表示买家可以购买 offerer 的任意 token id 的 token。

#### 3. 受限订单和额外数据（restricted order and extra data）

所谓受限制（RESTRICTED）的订单就是在成单前检查订单的时候需要进行额外的校验，只有通过校验的订单才能进行成交。

受限订单是所有方法都有的，额外数据是 Advanced 特有的。我们看看具体实现。

```solidity
// ZoneInteraction.sol => _assertRestrictedAdvancedOrderValidity()

    function _assertRestrictedAdvancedOrderValidity(
        AdvancedOrder memory advancedOrder,
        CriteriaResolver[] memory criteriaResolvers,
        bytes32[] memory priorOrderHashes,
        bytes32 orderHash,
        bytes32 zoneHash,
        OrderType orderType,
        address offerer,
        address zone
    ) internal view {
        // Order type 2-3 require zone or offerer be caller or zone to approve.
        // 如果是受限订单且不是由 offerer 或者 zone 执行的时候需要进行校验
        if (
            uint256(orderType) > 1 &&
            msg.sender != zone &&
            msg.sender != offerer
        ) {
            // If no extraData or criteria resolvers are supplied...
            // 如果不是基于标准处理的订单且 extraData 为空
            if (
                advancedOrder.extraData.length == 0 &&
                criteriaResolvers.length == 0
            ) {
                // Perform minimal staticcall to the zone.
                // 调用 zone 的 isValidOrder 方法，进行校验
                _callIsValidOrder(zone, orderHash, offerer, zoneHash);
            } else {
                // Otherwise, extra data or criteria resolvers were supplied; in
                // that event, perform a more verbose staticcall to the zone.
                // 其他情况下：基于标准处理的订单或者 extra data 存在。调用 zone 的 isValidOrderIncludingExtraData 方法
                bool success = _staticcall(
                    zone,
                    abi.encodeWithSelector(
                        ZoneInterface.isValidOrderIncludingExtraData.selector,
                        orderHash,
                        msg.sender,
                        advancedOrder,
                        priorOrderHashes,
                        criteriaResolvers
                    )
                );

                // Ensure call was successful and returned correct magic value.
                // 确保调用成功且返回的 magic value 是正确的。否则报错。
                _assertIsValidOrderStaticcallSuccess(success, orderHash);
            }
        }
    }
```

#### 4. 可以指定 recipient

fulfillOrder 和 fulfillAvailableOrders 方法只能指定 `msg.sender` 作为 recipient。fulfillAdvancedOrder 和 fulfillAvailableAdvancedOrders 可以指定任意的 recipient。

### Conduit（管道）

Conduit 是一个合约，发售者通过他来设置代币授权。Conduit 的所有者可以为 Conduit 添加和删除 "channel"，而注册的 channel 可以指示 Conduit 如何转移代币。Conduit 以完全 "选择 "的方式实现了可扩展性和可升级性，给创造者、收集者和平台提供了额外的能力，使他们能够利用 Seaport 做出自己的选择，同时保持与协议上其他挂单的广泛兼容性。

order 中的 conduitKey 就与此相关。我们看看具体是怎么实现的。

#### ConduitController 和 Conduit

首先来了解一下 ConduitController 和 Conduit 这两个合约。

![Conduit](Seaport.drawio.svg)

#### ConduitController

首先是 ConduitController（[00000000F9490004C11Cef243f5400493c00Ad63](https://etherscan.io/address/0x00000000F9490004C11Cef243f5400493c00Ad63#code)）。

用来管理所有 Conduit 的合约。可以创建和查询对应的 Conduit。

所有被管理的 Conduit 都放在 _conduits 这一个 map 类型的属性里。key 是 Conduit 的地址， value 是 ConduitProperties 类型。包含了 Conduit 的基本信息。

##### ConduitProperties

```solidity
// ConduitControllerInterface.sol => ConduitProperties

struct ConduitProperties {
        bytes32 key; // conduitKey
        address owner; // 当前的管理者
        address potentialOwner; // 将要转移给的管理者
        address[] channels; // 管理的 channel
        mapping(address => uint256) channelIndexesPlusOne; // channel 在 channels 中的indexe + 1
    }
```

然后就是创建 Conduit 的方法 `createConduit()`。

`conduitKey` 是 bytes32 类型的数据。前20个字节是该方法的调用者的地址。后面的字节补 0。也就是说 conduitKey 其实就是由创建 Conduit 的账户地址转换而来的。

`initialOwner` 是要创建的 Conduit 的 Owner。不能为空。

```solidity
// ConduitController.sol => createConduit()

function createConduit(bytes32 conduitKey, address initialOwner)
        external
        override
        returns (address conduit)
    {
        // Ensure that an initial owner has been supplied.
        // initialOwner 不能为空
        if (initialOwner == address(0)) {
            revert InvalidInitialOwner();
        }

        // If the first 20 bytes of the conduit key do not match the caller...
        // conduitKey 必须与 msg.sender 想匹配，否则报错
        if (address(uint160(bytes20(conduitKey))) != msg.sender) {
            // Revert with an error indicating that the creator is invalid.
            revert InvalidCreator();
        }

        // Derive address from deployer, conduit key and creation code hash.
        // 推算 conduit 的地址
        conduit = address(
            uint160(
                uint256(
                    keccak256(
                        abi.encodePacked(
                            bytes1(0xff),
                            address(this),
                            conduitKey,
                            _CONDUIT_CREATION_CODE_HASH
                        )
                    )
                )
            )
        );

        // If derived conduit exists, as evidenced by comparing runtime code...
        // 如果 conduit 已经存在，报错
        if (conduit.codehash == _CONDUIT_RUNTIME_CODE_HASH) {
            // Revert with an error indicating that the conduit already exists.
            revert ConduitAlreadyExists(conduit);
        }

        // Deploy the conduit via CREATE2 using the conduit key as the salt.
        // 使用 CREATE2 方法创建合约
        new Conduit{ salt: conduitKey }();

        // Initialize storage variable referencing conduit properties.
        // 创建 conduitProperties
        ConduitProperties storage conduitProperties = _conduits[conduit];

        // Set the supplied initial owner as the owner of the conduit.
        conduitProperties.owner = initialOwner;

        // Set conduit key used to deploy the conduit to enable reverse lookup.
        conduitProperties.key = conduitKey;

        // Emit an event indicating that the conduit has been deployed.
        // 发送 NewConduit 事件
        emit NewConduit(conduit, conduitKey);

        // Emit an event indicating that conduit ownership has been assigned.
        // 发送 OwnershipTransferred 事件
        emit OwnershipTransferred(conduit, address(0), initialOwner);
    }
```

通过调用 updateChannel() 来管理 Conduit 的 channel。需要注意的是虽然 Conduit 合约有 `updateChannel()` 方法，但是 Conduit 的 channel 必须由 ConduitController 来管理，不能直接调用。

```solidity
// ConduitController.sol => updateChannel()

function updateChannel(
        address conduit,
        address channel,
        bool isOpen
    ) external override {
        // Ensure the caller is the current owner of the conduit in question.
        // 只有 owner 才能调用
        _assertCallerIsConduitOwner(conduit);

        // Call the conduit, updating the channel.
        // 调用 conduit 的 updateChannel 方法
        ConduitInterface(conduit).updateChannel(channel, isOpen);

        // Retrieve storage region where channels for the conduit are tracked.
        // 查找 conduit 对应的 conduitProperties 信息
        ConduitProperties storage conduitProperties = _conduits[conduit];

        // Retrieve the index, if one currently exists, for the updated channel.
        // 查找 channel 的 索引，如果不存在默认是是 0
        uint256 channelIndexPlusOne = (
            conduitProperties.channelIndexesPlusOne[channel]
        );

        // Determine whether the updated channel is already tracked as open.
        // 判断 channel 状态是否是 open，
        bool channelPreviouslyOpen = channelIndexPlusOne != 0;

        // If the channel has been set to open and was previously closed...
        // 如果是要打开且当前状态不是打开的
        if (isOpen && !channelPreviouslyOpen) {
            // Add the channel to the channels array for the conduit.
            // 更新 conduitProperties 里 channels 的数据
            conduitProperties.channels.push(channel);

            // Add new open channel length to associated mapping as index + 1.
            // 更新 conduitProperties 里 channelIndexesPlusOne 的数据
            conduitProperties.channelIndexesPlusOne[channel] = (
                conduitProperties.channels.length
            );
        } else if (!isOpen && channelPreviouslyOpen) { 
            // 如果是要关闭且当前状态是打开

            // Set a previously open channel as closed via "swap & pop" method.
            // Decrement located index to get the index of the closed channel.
            uint256 removedChannelIndex;

            // Skip underflow check as channelPreviouslyOpen being true ensures
            // that channelIndexPlusOne is nonzero.
            // 跳过下溢检查，因为 channelPreviouslyOpen 为真已经确保了 channelIndexPlusOne 不为零。
            unchecked {
                removedChannelIndex = channelIndexPlusOne - 1;
            }

            // Use length of channels array to determine index of last channel.
            // 获得 channels 数组中最后一个元素的索引
            uint256 finalChannelIndex = conduitProperties.channels.length - 1;

            // If closed channel is not last channel in the channels array...
            // 如果要关闭的 channel 不是最后一个元素
            if (finalChannelIndex != removedChannelIndex) {
                // Retrieve the final channel and place the value on the stack.
                // 获得最后一个 channel 的值
                address finalChannel = (
                    conduitProperties.channels[finalChannelIndex]
                );

                // Overwrite the removed channel using the final channel value.
                // 将最后一个 channel 的值赋值给要关闭的 channel 的位置
                conduitProperties.channels[removedChannelIndex] = finalChannel;

                // Update final index in associated mapping to removed index.
                // 更新 conduitProperties 数据
                conduitProperties.channelIndexesPlusOne[finalChannel] = (
                    channelIndexPlusOne
                );
            }

            // Remove the last channel from the channels array for the conduit.
            // 删除最后一个元素
            conduitProperties.channels.pop();

            // Remove the closed channel from associated mapping of indexes.
            // 删除 channelIndexesPlusOne 中的数据
            delete conduitProperties.channelIndexesPlusOne[channel];
        }
    }
```

涉及到权限管理的方法有三个 `transferOwnership()`、`cancelOwnershipTransfer()` 和  `acceptOwnership()`。

`transferOwnership()` 必须是对应 Conduit 的 owner 发起的。发起后并不会直接将权限给对应的地址。而是将对应的地址赋值给 ConduitProperties 中的 potentialOwner。 对应的地址调用 `acceptOwnership()` 后才会成为对应 Conduit 的 owner。

#### Conduit

Conduit 合约继承自 TokenTransferrer。也就是说 Conduit 合约负责的是 Token 的转移。他里面有各种转移 token 的方法。

其实在订单中如果不使用 Conduit 的话，最终调用的转移 token 的方法就是 TokenTransferrer 合约里的方法。使用 Conduit 的目的就是控制代币的转移。可以通过设置 channel 和 channel 的状态来控制 token 的转移。

这一切个关键就在于 onlyOpenChannel 这个函数修饰器上。通过它来确保调用者是一个注册在 Conduit 上的 channel， 并且该 channel 是打开的。

```solidity
// Conduit.sol => onlyOpenChannel

modifier onlyOpenChannel() {
        // Utilize assembly to access channel storage mapping directly.
        // 直接访问存储 channel 的 mapping
        assembly {
            // Write the caller to scratch space.
            // 将 caller 写入
            mstore(ChannelKey_channel_ptr, caller())

            // Write the storage slot for _channels to scratch space.
            mstore(ChannelKey_slot_ptr, _channels.slot)

            // Derive the position in storage of _channels[msg.sender]
            // and check if the stored value is zero.
            // 推导出 _channels[msg.sender] 的存储位置，并检查是否是 0， 如果是 0 则表示 channel 不是打开的，报 ChannelClosed 的错误
            if iszero(
                sload(keccak256(ChannelKey_channel_ptr, ChannelKey_length))
            ) {
                // The caller is not an open channel; revert with
                // ChannelClosed(caller). First, set error signature in memory.
                mstore(ChannelClosed_error_ptr, ChannelClosed_error_signature)

                // Next, set the caller as the argument.
                mstore(ChannelClosed_channel_ptr, caller())

                // Finally, revert, returning full custom error with argument.
                revert(ChannelClosed_error_ptr, ChannelClosed_error_length)
            }
        }
```

#### 订单成交中涉及的逻辑



### fulfillBasicOrder

```solidity
// 说明：为了更容易查看，我将方法的 ABI 中的 tuple 标识符删除了，换成这个 tuple 对应的参数名称。
// 比如正常情况下 parameters 的位置应该是 tuple ，parameters 在最后面。
// 下同。

function fulfillBasicOrder(
    parameters(
        address considerationToken, 
        uint256 considerationIdentifier, 
        uint256 considerationAmount, 
        address offerer, 
        address zone, 
        address offerToken, 
        uint256 offerIdentifier, 
        uint256 offerAmount, 
        uint8 basicOrderType, 
        uint256 startTime, 
        uint256 endTime, 
        bytes32 zoneHash, 
        uint256 salt, 
        bytes32 offererConduitKey, 
        bytes32 fulfillerConduitKey, 
        uint256 totalOriginalAdditionalRecipients, 
        additionalRecipients(
            uint256 amount, 
            address recipient
            )[] , 
        bytes signature
        ) 
    ) 
    
    payable 
    
    returns (bool fulfilled)
```

#### 通过 fulfillBasicOrder 执行的订单要满足的条件

1. 该订单只包含一个 offer 项目，并且至少包含一个 consideration 项目。
2. 该订单只包含一个 ERC721 或 ERC1155 项目，并且该项目不是基于标准的（Criteria-based）。
3. 该订单的 offerer 会收到 consideration 中第一个项目。
4. 用作货币（currency）的 token 必须是同一种。也就是说要么是原生代币作为支付货币 要么是 ERC20 的 token 作为支付货币，不能混合支付。
5. offer 不能是原生代币。
6. 每个项目的 startAmount 必须与该项目的 endAmount 一致（即项目不能有升/降金额）。
7. 所有 "被忽略 "的项目字段（即本地项目的token和identifierOrCriteria以及ERC20项目的identifierOrCriteria）被设置为空地址或零。
8. 原生货币项目上的token需要设置为空地址，货币上的标识符需要为 0，ERC721项目上的数量要为 1。
9. 如果订单有多个对价项目，并且除第一个对价项目外的所有对价项目与被提供的项目类型相同，则被提供的项目金额不低于除第一个对价项目金额外的所有对价项目金额之和。

#### BasicOrderRouteType and BasicOrderType

```solidity
// ConsiderationEnums.sol => BasicOrderRouteType

enum BasicOrderRouteType {
    // 0: provide Ether (or other native token) to receive offered ERC721 item.
    ETH_TO_ERC721,

    // 1: provide Ether (or other native token) to receive offered ERC1155 item.
    ETH_TO_ERC1155,

    // 2: provide ERC20 item to receive offered ERC721 item.
    ERC20_TO_ERC721,

    // 3: provide ERC20 item to receive offered ERC1155 item.
    ERC20_TO_ERC1155,

    // 4: provide ERC721 item to receive offered ERC20 item.
    ERC721_TO_ERC20,

    // 5: provide ERC1155 item to receive offered ERC20 item.
    ERC1155_TO_ERC20
}
```

这里有个 `BasicOrderRouteType` 的概念。也就是交换路径。表示支付什么类型的代币获取什么类型的代币。

注意：上面说过 offer 不能是原生代币，因此没有 `ERC721_TO_ETH` 或者 `ERC1155_TO_ETH` 的类型。

还有一点 BasicOrder 的订单的类型不是最上面我们讲到的 `OrderType`。 他有自己的 `BasicOrderType`。

`basicOrderType = orderType + (4 * basicOrderRoute)`

与 `OrderType` 类似。

```solidity
// ConsiderationEnums.sol => BasicOrderType

enum BasicOrderType {
    // 0: no partial fills, anyone can execute
    ETH_TO_ERC721_FULL_OPEN,

    // 1: partial fills supported, anyone can execute
    ETH_TO_ERC721_PARTIAL_OPEN,

    // 2: no partial fills, only offerer or zone can execute
    ETH_TO_ERC721_FULL_RESTRICTED,

    // 3: partial fills supported, only offerer or zone can execute
    ETH_TO_ERC721_PARTIAL_RESTRICTED,

    // 4: no partial fills, anyone can execute
    ETH_TO_ERC1155_FULL_OPEN,

    // 5: partial fills supported, anyone can execute
    ETH_TO_ERC1155_PARTIAL_OPEN,

    // 6: no partial fills, only offerer or zone can execute
    ETH_TO_ERC1155_FULL_RESTRICTED,

    // 7: partial fills supported, only offerer or zone can execute
    ETH_TO_ERC1155_PARTIAL_RESTRICTED,

    // 8: no partial fills, anyone can execute
    ERC20_TO_ERC721_FULL_OPEN,

    // 9: partial fills supported, anyone can execute
    ERC20_TO_ERC721_PARTIAL_OPEN,

    // 10: no partial fills, only offerer or zone can execute
    ERC20_TO_ERC721_FULL_RESTRICTED,

    // 11: partial fills supported, only offerer or zone can execute
    ERC20_TO_ERC721_PARTIAL_RESTRICTED,

    // 12: no partial fills, anyone can execute
    ERC20_TO_ERC1155_FULL_OPEN,

    // 13: partial fills supported, anyone can execute
    ERC20_TO_ERC1155_PARTIAL_OPEN,

    // 14: no partial fills, only offerer or zone can execute
    ERC20_TO_ERC1155_FULL_RESTRICTED,

    // 15: partial fills supported, only offerer or zone can execute
    ERC20_TO_ERC1155_PARTIAL_RESTRICTED,

    // 16: no partial fills, anyone can execute
    ERC721_TO_ERC20_FULL_OPEN,

    // 17: partial fills supported, anyone can execute
    ERC721_TO_ERC20_PARTIAL_OPEN,

    // 18: no partial fills, only offerer or zone can execute
    ERC721_TO_ERC20_FULL_RESTRICTED,

    // 19: partial fills supported, only offerer or zone can execute
    ERC721_TO_ERC20_PARTIAL_RESTRICTED,

    // 20: no partial fills, anyone can execute
    ERC1155_TO_ERC20_FULL_OPEN,

    // 21: partial fills supported, anyone can execute
    ERC1155_TO_ERC20_PARTIAL_OPEN,

    // 22: no partial fills, only offerer or zone can execute
    ERC1155_TO_ERC20_FULL_RESTRICTED,

    // 23: partial fills supported, only offerer or zone can execute
    ERC1155_TO_ERC20_PARTIAL_RESTRICTED
}
```

fulfillBasicOrder 方法更像是为了兼容 Wyvern Protocol。

### fulfillOrder

在测试用例中 fulfillOrder 和 fulfillAdvancedOrder 被称作标准方法（Stand）。也是使用最多的方法。

```solidity
function fulfillOrder(
    order(
        parameters(
            address offerer, 
            address zone, 
            offer(
                uint8 itemType, 
                address token, 
                uint256 identifierOrCriteria, 
                uint256 startAmount, 
                uint256 endAmount
            )[] , 
            consideration(
                uint8 itemType, 
                address token, 
                uint256 identifierOrCriteria, 
                uint256 startAmount, 
                uint256 endAmount, 
                address recipient
            )[] , 
            uint8 orderType, 
            uint256 startTime, 
            uint256 endTime, 
            bytes32 zoneHash, 
            uint256 salt, 
            bytes32 conduitKey, 
            uint256 totalOriginalConsiderationItems
        ) , 
        bytes signature
    ) , 
    bytes32 fulfillerConduitKey
    ) 
    
    payable 
    
    returns (bool fulfilled)
```

从上面的 ABI 上，我们可以看出大部分的内容前面都做过解析。只有两个我们现在还不了解。一个是 `conduitKey，` 一个是 `fulfillerConduitKey。` BasicOrder 里面也有类似的 `offererConduitKey` 和 `fulfillerConduitKey。` 他们的作用都是一样的。

### fulfillAdvancedOrder

```solidity
function fulfillAdvancedOrder(
    advancedOrder(
        parameters(
            address offerer,
            address zone,
            offer(
                uint8 itemType,
                address token,
                uint256 identifierOrCriteria,
                uint256 startAmount,
                uint256 endAmount
            )[],
            consideration(
                uint8 itemType,
                address token,
                uint256 identifierOrCriteria,
                uint256 startAmount,
                uint256 endAmount,
                address recipient
            )[],
            uint8 orderType,
            uint256 startTime,
            uint256 endTime,
            bytes32 zoneHash,
            uint256 salt,
            bytes32 conduitKey,
            uint256 totalOriginalConsiderationItems
        ),
        uint120 numerator, // 分子
        uint120 denominator, // 分母
        bytes signature,
        bytes extraData
    ),
    criteriaResolvers(
        uint256 orderIndex,
        uint8 side,
        uint256 index,
        uint256 identifier,
        bytes32[] criteriaProof
    )[],
    bytes32 fulfillerConduitKey,
    address recipient
)

payable

returns(bool fulfilled)
```

### fulfillAvailableOrders

```solidity

function fulfillAvailableOrders(
    orders(
        parameters(
            address offerer, 
            address zone, 
            offer(
                uint8 itemType, 
                address token, 
                uint256 identifierOrCriteria, 
                uint256 startAmount, 
                uint256 endAmount
                )[] , 
            consideration(
                uint8 itemType, 
                address token, 
                uint256 identifierOrCriteria, 
                uint256 startAmount, 
                uint256 endAmount, 
                address recipient
                )[] , 
            uint8 orderType, 
            uint256 startTime, 
            uint256 endTime, 
            bytes32 zoneHash, 
            uint256 salt, 
            bytes32 conduitKey, 
            uint256 totalOriginalConsiderationItems
            ) , 
        bytes signature
        )[] , 
    offerFulfillments(
        uint256 orderIndex, 
        uint256 itemIndex
        )[][] , 
    considerationFulfillments(
        uint256 orderIndex, 
        uint256 itemIndex
        )[][] , 
    bytes32 fulfillerConduitKey, 
    uint256 maximumFulfilled
    ) 
    
    payable 
    
    returns (
        bool[] availableOrders, 
        executions(
            item(
                uint8 itemType, 
                address token, 
                uint256 identifier, 
                uint256 amount, 
                address recipient
                ) , 
            address offerer, 
            bytes32 conduitKey
            )[] 
        )
```

### fulfillAvailableAdvancedOrders

```solidity
function fulfillAvailableAdvancedOrders(
    advancedOrders(
        parameters(
            address offerer,
            address zone,
            offer(
                uint8 itemType,
                address token,
                uint256 identifierOrCriteria,
                uint256 startAmount,
                uint256 endAmount
            )[],
            consideration(
                uint8 itemType,
                address token,
                uint256 identifierOrCriteria,
                uint256 startAmount,
                uint256 endAmount,
                address recipient
            )[],
            uint8 orderType,
            uint256 startTime,
            uint256 endTime,
            bytes32 zoneHash,
            uint256 salt,
            bytes32 conduitKey,
            uint256 totalOriginalConsiderationItems
        ),
        uint120 numerator,
        uint120 denominator,
        bytes signature,
        bytes extraData
    )[],
    criteriaResolvers(
        uint256 orderIndex,
        uint8 side,
        uint256 index,
        uint256 identifier,
        bytes32[] criteriaProof
    )[],
    offerFulfillments( // 所有的提供详情：根据地址划分
        uint256 orderIndex,
        uint256 itemIndex
    )[][],
    considerationFulfillments( // 所有的收取详情：根据地址划分
        uint256 orderIndex,
        uint256 itemIndex
    )[][],
    bytes32 fulfillerConduitKey,
    address recipient,
    uint256 maximumFulfilled
)

payable

returns(
    bool[] availableOrders,
    executions(
        item(
            uint8 itemType,
            address token,
            uint256 identifier,
            uint256 amount,
            address recipient
        )[],
        address offerer,
        bytes32 conduitKey
    )
)
```

<!-- ![Seaport](Seaport.drawio.svg) -->
