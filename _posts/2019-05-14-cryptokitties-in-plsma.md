---
layout: post
title:   CryptoKitties in Plasma EVM (KR)
date:   2019-5-14 00:00:00 +09:00
author: "박주형(Carl)"
categories: ethereum dapp
tag: [ethereum, scalability, plasma, plasmaEVM, dapp]
youtubeId :
slideWebId :
---

# CryptoKitties in Plasma EVM (KR)

Carl Park([4000D](https://medium.com/@4000d),
[github](https://github.com/4000d))

KR / EN

![](https://cdn-images-1.medium.com/max/1600/1*c0kAJf0LAbiUSwtJeDuUFw.png)

CryptoKitties는 가장 성공적인 ERC721 토큰 중 하나로 한 때 이더리움 네트워크를 상당수 잡아먹으며 성공적인 Dapp으로
떠올랐습니다. 하지만 많은 Dapp들이 그러했듯 높아진 트래픽을 감당하지 못하는 이더리움 확장성 문제와 그에 따라 증가한 트랜잭션 수수료 부담은
사용성을 저해하는 가장 큰 문제로 떠올랐습니다. 가장 많이 쓰이는 ERC20 토큰의 경우 UTXO 기반의 플라즈마, state channel
등을 이용하여 충분히 이더리움 성능의 한계를 극복할 수 있지만, 로직이 복잡한 CryptoKitties, MakerDAO, Aragon 같은
경우 동일한 방식을 적용할 수 없습니다. 본 글은 이러한 문제를 Plasma EVM을 통해 해결할 수 있는 방향을 CryptoKitties을
예시로 제시합니다.

### 1. CryptoKitties Overview

우선 CryptoKitties가 어떻게 구성되어있는지 살펴봅시다. 컨트랙트는 크게 Auction과 KittyCore 파트로 나누어져있습니다.

![](https://cdn-images-1.medium.com/max/1600/1*8GIz9Ovmdq-bQRMjQDkrIw.png)
<span class="figcaption_hack">Figure 1. Contract Diagram</span>

위 그림에서 실선은 상속 관계를, 점선은 참조 관계를 표현합니다. 실선의 방향은 derived 컨트랙트에서 base 컨트랙트로 향합니다. 점선의
방향은 참조 하는 컨트랙트(referrer)에서 참조 당하는 컨트랙트(referee)로 향합니다.

이더리움에 배포되는 컨트랙트는 `SaleClockAuction`, `SiringClockAuction`, `KittyCore`,
`ERC721Metadata` 입니다. 두 옥션 컨트랙트가 참조하는 ERC721는 결국 `KittyCore`를 가리키며, `KittyCore`는
`KittyOwnership`을 상속하여 ERC721를 구현합니다.

*****

#### Auctions

    contract ClockAuctionBase
    contract ClockAuction is Pausable, ClockAuctionBase
    contract SaleClockAuction is ClockAuctionBase
    contract SiringClockAuction is ClockAuctionBase

* **Pausable**: 위급한 상황에 컨트랙트를 중지시킬 수 있는 OpenZeppelin의 구현체.
* **ClockAuctionBase**: `ClockAuction`에서 사용하는 기능들의 internal function, `Auction`
구조체, 상태 변수를 구현한 컨트랙트. ERC721 토큰에 대한 판매가격이 시간이 지날수록 가격이 선형적으로 증감하는 경매을 생성하고 유저가
bid 하는 로직을 구현.<br> 각 `Auction`은 `seller`, `startingPrice`, `endingPrice`,
`duration`, `startedAt`을 멤버로 가지며 `tokenId(kittyId)`로 식별.
* **ClockAuction**: 옥션 생성, 삭제 및 조회와 bid 할 수 있는 external function을 구현. 추가적으로
Auction fee를 출금할 수 있는 함수 구현.
* **SaleClockAuction**: 키티의 소유권을 이전할 수 있는 옥션을 생성하는 컨트랙트.
* **SiringClockAuction**: Sire 키티와 번식할 수 있는 옥션 기능 제공. Siring auction은 오직
`KittyCore` 컨트랙트를 통해서 생성 가능.

*****

#### KittyCore

    contract KittyAccessControl
    contract KittyBase is KittyAccessControl
    contract KittyOwnership is KittyBase, ERC721
    contract KittyBreeding is KittyOwnership
    contract KittyAuction is KittyBreeding
    contract KittyMinting is KittyAuction
    contract KittyCore is KittyMinting

    contract GeneScience

* **KittyAccessControl**: CEO / CFO / COO 등의 주소 및 권한을 설정하는 컨트랙트. C-Level 이 컨트랙트
동작을 멈출 수 있음 (Pausable).
* **KittyBase**: `Kitty` 구조체를 구현한 컨트랙트. 키티를 전송하거나 생성하는 internal functions 구현.
* **KittyOwnership**: 키티의 소유권을 전달하는 기능을 `KittyBase`를 상속하여 ERC721 표준에 따른 함수 구현.
* **KittyBreeding**: `GeneScience`를 이용하여 번식(breeding)하는 기능 구현. 자신이 가진 두 키티를 이용해
각각을 matron, sire로 사용하던가 SireAuction을 이용하여 자신의 matron을 다른 유저가 소유한 sire와 번식할 수 있음.
<br> 번식 후 matron 키티는 임신한 상태가 되며, 특정 블록 넘버까지 cooldown 상태가 유지되는데 cooldown이 종료된 이후
임의의 유저가 `KittyBreeding.giveBirth(matronId)` 함수를 호출하여 임신한 matron에서 새로운 자식 키티를 생성할
수 있음.<br> `giveBirth`를 통해 키티를 생성하는 유저에게 인센티브로 `autoBirthFee(0.008 ETH)`를 전달.
* **KittyAuction**: `SaleClockAuction`, `SiringClockAuction` 컨트랙트 주소를 지정하고 각 옥션을
생성하는 로직 구현.
* **KittyMinting**: PromoKitty, 0세대 키티 발행 기능 구현. 0세대 키티는 발행과 동시에 `SiringAuction`이
자동으로 생성됨. COO만이 해당 컨트랙트를 통해 키티들을 생성할 수 있으며 gene을 임의로 지정함.
* **KittyCore**: 이더리움에 배포되는 최종 CrptoKitties 컨트랙트. Kitty 정보를 조회하거나 Auction fee를
출금하는 기능 구현.
* **GeneScience**: 두 부모(matron, sire)의 유전자를 이용해 자식 키티의 유전자를 생성하는 컨트랙트. 두 부모의 **유전적
특성(traits)**과 **랜덤성**(특정 블록의 해시)를 기반으로 자식 키티의 유전적 특성이 결정됨.

*****

#### Contract Interaction

두 종류의 세일 및 번식 행위를 어떻게 수행할 수 있는지 아래 다이어그램으로 살펴보겠습니다. 사용자가 보낸 트랜잭션 및 message-call이
실선으로 표현되어있습니다.

<span class="figcaption_hack">Figure 2. SaleAuction Diagram</span>

1.  Seller가 `SaleClockAuction` 컨트랙트가 자신의 키티를 가져갈 수 있도록 approve 합니다.
1.  Seller가 `KittyCore` 컨트랙트를 통해 sale auction을 생성합니다. 이 과정에서 `SaleClockAuction`
컨트랙트가 판매를 원하는 키티의 소유권을 가져옵니다 (2.2).
1.  Buyer가 옥션 진행중인 특정 키티에 입찰(bid)합니다. 판매 가격은 `startingPrice + (endingPrice —
startingPrice) * (block.timestamp — startedAt/duration)` 으로 결정됩니다. 판매가보다 높은 금액을
전달했다면 즉시 키티의 소유권을 받습니다.<br> 판매 가격의 97.5%를 판매자가, 나머지 3.25%를 `KittyCore`가 가지며
`KittyCore`에 있는 잔액들은 CFO가 회수할 수 있습니다.

*****

<span class="figcaption_hack">Figure 3. SiringAuction DIagram</span>

1.  Seller가 `SiringClockAuction` 컨트랙트가 자신의 키티를 가져갈 수 있도록 approve 합니다.
1.  Seller가 `KittyCore` 컨트랙트를 통해 siring auction을 생성합니다. 이 과정에서 `SiringClockAuction`
컨트랙트가 판매를 원하는 키티의 소유권을 가져옵니다 (2.2).
1.  Buyer가 `KittyCore` 컨트랙트를 통해 옥션 진행중인 특정 키티에 입찰합니다. 판매 가격은 sale auction과 동일하게
결정됩니다. 하지만 `autoBirthFee`를 `KittyCore`컨트랙트가 지불해야되기 때문에, `KittyCore` 컨트랙트는 입찰
트랜잭션의 `msg.value`에서 `autoBirthFee`를 삭감한 금액으로 `SiringClockAuction.bid() `함수를
호출합니다(3.1).<br> 입찰에 성공한다면, 판매 금액의 일부를 판매자에게 전달하고 (3.2), `SiringClockAuction`
컨트랙트는 키티의 소유권을 다시 판매자에게 전달합니다 (3.3). 그리고 `KittyCore` 컨트랙트가 sire와 matron간의 번식을
수행합니다.
1.  누군가 matron의 cooldown 기간이 끝난 후 자식 키티의 생성을 위해 `KittyCore.giveBirth()` 함수를 호출합니다. 이
때 `GeneScience `컨트랙트를 통해 자식 키티의 유전자를 계산합니다 (4.1). 자식 키티의 id는 가장 `마지막 키티 id + 1`로
결정됩니다. <br> 마지막으로 `giveBirth()`를 호출한 어카운트에게 `autoBirthFee`를 제공합니다.

*****

<span class="figcaption_hack">Figure 4. Auto-Breeding Diagam (w/o siring aucton)</span>

자신이 가진 두 키티 혹은 siring을 approve받은 키티와 자신의 matron 키티를 이용하여 siring auction없이 번식하는
것은 위와 같습니다.

1.  Alice가 Bob에게 자신의 키티를 sire로서 번식하는 것을 approve 합니다. 만약 Alice와 Bob이 동일하다면 생략할 수
있습니다.
1.  Bob이 자신의 matron과 Alice의 sire를 지정하여 번식을 수행합니다. 이 때 autoBirthFee를 함께 지불합니다.
1.  matron의 cooldown 기간이 종료되면 자식 키티를 생성합니다. 해당 트랜잭션으로 발생한 수수료를 보상하기 위하여
`autoBirthFee`를 제공합니다.

*****

### 2. Make CryptoKitties Requestable

이제 CryptoKitties가 어떻게 구성되고 동작하는지 살펴 보았으니, 이를 Plasma EVM에 옮기기 위한 과정을 살펴봅시다.
Plasma EVM에서 루트 체인과 자식 체인간의 자산의 이동이 가능한 것을 Requestable이라고 부릅니다. 따라서 CryptoKitty의
Requestable 해야하는 기능들은 다음과 같습니다.

#### ERC721

키티는 ERC721 표준을 따르는 토큰입니다. 따라서 ERC721 기능인 `tokenId`를 통한 전송는 반드시 requestable해야
합니다.

#### KittyCore / GeneScience

CryptoKitties에서 가장 중요한 기능은 번식 입니다. 특히 자신이 가지고있지 않은 키티를 번식하기 위해서 Siring Auction
기능은 반드시 플라즈마 체인에서 수행되어야 합니다. 또한 번식을 위해서는 **두 부모의 유전자 정보**가 필요하기 때문에 단순히 ERC721
토큰을 requestable하게 만드는 것이 아니라`KittyCore`또한 플라즈마 체인에서 온전히 동작해야 합니다.

#### Sale Auction / Siring Auction

옥션 중인 키티(Kitty on auction)의 소유권은 해당 옥션 컨트랙트가 가지고있습니다. 자산을 보호하기 위해서는 해당 키티들 또한
requestable 해야 됩니다. `KittyCore` 컨트랙트가 이미 requestable 이고 두 옥션 컨트랙트의 주소또한
`KittyCore` 컨트랙트에 지정되어있으므로 별도로 각 세일 컨트랙트는 requestable 할 필요는 없습니다.

*****

`KittyCore`에서 필요한 주된 request는 다음과 같이 정의될 수 있습니다.

#### 1. Request for Kitty Ownership

ERC721 토큰으로서 키티의 소유권을 다른 체인으로 보낼 수 있어야 합니다. 이 때 해당 키티의 유전자와 breeding 데이터를 함께
전달해야 합니다.

#### 2. Request for Kitty Gene

여러 세대에 걸쳐 번식된 키티를 다른 체인으로 request할 경우 부모 키티의 정보를 항상 조회할 수 있어야 합니다. 이 때 단순한 유전자
정보는 키티의 소유권과 다르게 누구나 접근 가능한 read-only 데이터이기 때문에 request 할 수 있을 것입니다.

#### 3. Request for Kitty on Auction

Asset security를 보장하기 위하여 특정 컨트랙트가 가진 키티 또한 반드시 requestable 해야 합니다. 특히
`KittyCore`에 직접 연결된 컨트랙트인 `SaleAuction`, `SiringAuction`의 경우 각 옥션의 판매자가 해당 키티에
대한 request를 생성할 권한을 가집니다.

*****

`KittyCore` 컨트랙트를 requestable 하도록 변경하기 위해서는 다음과같은 기능들이 필요합니다.

#### 1. Pseudo-random Kitty ID as uint256

자식 키티의 ID는 auto-incrementing sequence 입니다. 따라서 자식 체인과 부모 체인에서 동시에 생성된 키티는 동일한
ID를 가지는 충돌이 발생할 수 있기에 키티의 유전자 정보, 부모 키티의 ID 등을 이용해 unique ID를 부여할 필요가 있습니다.

#### 2. uint256 for matronId, sireId, siringWithId for Kitty struct

`KittyCore`의 `Kitty` 구조체는 부모 키티의 ID를 uint32 자료형으로 저장합니다. 생성 가능한 키티의 갯수가 32 비트를
초과하지 않는다는 가정과 auto-incrementing id를 이용할 경우 `Kitty` 구조체를 최적할 수 있기 때문입니다. 하지만
Pseudo-random Kitty ID as uint256을 적용할 경우 키티의 ID가 uint32 자료형으로 표현될 수 없기 때문에마찬가지로
`Kitty.matronId`, `Kitty.sireId`, `Kitty. siringWithId` 또한 uint256 자료형으로 변경이
필요합니다.

#### 3. Randomness for Child Kitty Gene

`GeneScience.mixGene(uint256 _genes1, uint256 _genes2, uint256 _targetBlock)`
함수는 특정 이더리움 블록(`_targetBlock`) 해시에서 랜덤성을 가져옵니다. `_targetBlock`는 matron의
`cooldownBlock-1` 으로 지정되는데, 해당 이더리움 블록 해시를 플라즈마에서도 사용할 수 있어야 합니다.

따라서 부모 블록 해시를 사용하는 기존 로직을 그대로 활용한다면, 별도의 이더리움 블록 해시를 자식 체인으로 복사하는 기능이 필요합니다. 부모
블록 해시는 여러 Requestable 컨트랙트에서 공통적으로 사용될 여지가 있기 때문에 Plasma EVM의 내부 프로토콜에서 구현되는 것이
바람직합니다.

`GeneScience.mixGene`함수는 부모 체인에서는 BLOCKHASH opcode를 사용하고 자식 체인에서는 다른 컨트랙트에서 부모
블록 해시를 읽어오도록 수정이 필요합니다. 따라서 `pure`하지 않고 특정 컨트랙트를 읽어야하는 `view` 함수가 되어야 합니다.

#### 4. withdrawBalance() with specific amount

`pregnantKitties * autoBirthFee`만큼의 이더를 항상 `KittyCore` 컨트랙트가 보유하고 있어야하기 때문에 경매
수수료로 생긴 이익금을 전액 출금한다면 다른 체인에서 임신한 키티가 request를 통해 이동하였을 때 `autoBirthFee`를 지불하지
못할 수 있습니다.

CFO가 `withdrawBalance()` 함수를 통해 가능한 모든 금액을 인출할 경우 따라서 CFO는 부모체인과 자식체인의 모든 임신한
키티의 수를 고려하여 인출할 금액을 지정해야 합니다.

*****

마지막으로 `KittyCore`의 모든 상태 변수들에대한 request는 다음과 같이 요약될 수 있습니다.

* `KittyAccessControll.paused`: only enter by anyone
* `KittyAccessControll.ceoAddress`: only enter by anyone
* `KittyAccessControll.cfoAddress`: only enter by anyone
* `KittyAccessControll.cooAddress`: only enter by anyone
* `KittyBreeding.autoBirthFee`: only enter by anyone

위 변수들은 루트체인에서 자식체인으로 enter 만 허용함으로서 권한을 일방향으로 강제할 수 있습니다.

* `KittyBase.kitties`: enter or exit **by anyone**
* `KittyBase.kittyIndexToOwner`: enter and exit **by kitty owner**

개별 키티의 데이터를 가지고있는 `kitties` 변수는 누구나 request할 수 있도록 허용하며, 해당 키티의 소유자만이 소유권에 대한
request를 만들 수 있어야 합니다.

* `KittyBase.kittyIndexToApproved`: non-requestable.
* `KittyBase.ownershipTokenCount`: non-requestable.
* `KittyBase.sireAllowedToAddress`: non-requestable

위 변수들은 transfer() 함수에서 소유권의 이전과 함께 삭제되는 값들입니다. 직접적인 request 대상이 되지 않습니다.

* `KittyBreeding.pregnantKitties`: Pregenent Kitty Ownership Request enter / exit
시 증감

임신한 키티의 소유권을 다른 체인으로 이동시킬 때 증감시킵니다.

* `KittyBase.saleAuction`: non-requestable. set by CEO
* `KittyBase.siringAuction`: non-requestable. set by CEO
* `KittyBreeding.geneScience`: non-requestable. set by CEO
* `KittyCore.newContractAddress`: non-requestable. set by CEO

외부 컨트랙트의 주소들은 오직 CEO만 설정 가능하기에 requestable 하지 않습니다.

* `KittyMinting.promoCreatedCount`: only enter by anyone
* `KittyMinting.gen0CreatedCount`: only enter by anyone

발행된 promo kitty와 0세대 키티들의 수는 자식 체인에서 추가 발행을 막기 위하여 누구나 enter 가능해야 합니다. 이러한 특별한
키티들은 오직 CEO만 발행할 수 있기 때문에 권한이 올바르게 유지된다면 추가 발행되는 일은 발생하지 않습니다.

* [Ethereum](https://medium.com/tag/ethereum?source=post)
* [Cryptokitties](https://medium.com/tag/cryptokitties?source=post)
* [Plasma Evm](https://medium.com/tag/plasma-evm?source=post)
* [Smart Contract](https://medium.com/tag/smart-contracts?source=post)

### [4000D](https://medium.com/@4000d)

carl.p

### [Onther-Tech](https://medium.com/onther-tech?source=footer_card)

Building an Ethereum Blockchain ECO system to Change the World


* content
{:toc}
{% include youtubePlayer.html id=page.youtubeId %}
{% include slidePlayer.html id=page.slideWebId %}
