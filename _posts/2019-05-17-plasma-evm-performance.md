---
layout: post
title:   Plasma EVM 성능 테스트
date:   2019-5-17 00:00:00 +09:00
author: "신건우(Thomas)"
categories: ethereum scalability plasma plsmaEVM
tag: [ethereum, dapp, zkp]
youtubeId :
slideWebId : 
---
* content

# Plasma EVM 성능 테스트

![](https://cdn-images-1.medium.com/max/1600/1*DpK4Hee3UDE6zIn49LRjFw.png)

Plasma-EVM은 Tokamak 네트워크의 코어 구현체로 EVM을 내장하고 있어, 튜링 완전한 연산을 실행할 수 있는 새로운 형태의 플라즈마
구현체입니다. Plasma-EVM에 대한 기술 스펙은
[여기](https://github.com/Onther-Tech/papers/blob/master/plasma-evm/plasma-evm.pdf)서
자세히 확인할 수 있습니다.

Plasma-EVM은 단일 오퍼레이터가 운영하는 플라즈마 체인으로 체인의 성능을 최대한 끌어올릴 수 있습니다. 체인의 성능은 TPS, 즉 초당
얼마나 많은 트랜잭션을 처리하는가로 평가됩니다.

### Plasma-EVM

Plasma-EVM의 블록 생성 시간, 즉 블록 타임은 **14초**이고 블록 가스리밋은 **3억**입니다. 현재 Ethereum
mainnet과 비교했을 때 블록 타임은 비슷하지만 블록 가스리밋에서 큰 차이가 있습니다.

> 현재 Ethereum mainnet의 block gasLimit은 약
> [8백만](https://etherscan.io/chart/gaslimit)이고 평균 블록 타임(블록 생성 시간)은 평균
[14초](https://etherscan.io/chart/blocktime) 입니다.

TPS 측정에 사용되는 트랜잭션은 단순 이더 전송 트랜잭션, 즉 21,000 gas만을 소모하는 트랜잭션입니다. Plasma-EVM의 블록
가스리밋이 3억이기 때문에 이를 21,000으로 나누면 약 14285가 됩니다. Plasma-EVM의 블록 타임이 14초이기 때문에 이를 또
14초로 나누면 약 1000이 됩니다. 즉 이론적으로 1초당 1000개의 Transaction을 Plasma-EVM에서 처리할 수 있습니다.

**1000 TPS** ≒ 300,000,000 *block gasLimit* / 21,000 *gasUsed* / 14s *block
time*

#### 구성

우선 Plasma-EVM의 TPS를 측정하기 위해서는 아래 그림과 같이 네트워크가 구성되어야 합니다.

![](https://cdn-images-1.medium.com/max/1200/1*zLh-TbIoqaHOftXdV5OZag.png)

* **Ethereum(RootChain)**: Tokamak network는 플라즈마 체인이기 때문에 루트체인에 종속되어 있습니다.
Ethereum 네트워크가 바로 Tokamak network의 루트체인입니다.
* **RootChain contract**<br> : Tokamak 네트워크의 상태를 강제 또는 Plasma Block을 검증하는 역할을 수행하는
컨트랙트입니다. 이 컨트랙트가 있어야 올바른 플라즈마 체인 운영이 가능합니다.
* **Plasma(Tokamak)**<br> : Tokamak network는 단일 오퍼레이터가 운영하는 플라즈마 체인입니다.

### TPS 측정

Plasma EVM의 TPS는 여러 변수들에 의해 달라질 수 있습니다. 우선 Plasma-EVM은 루트체인에 종속적이기 때문에 **루트체인의
블록 타임**에 영향을 받습니다. 루트체인의 블록 타임이 빠르면 빠를 수록 Plasma-EVM의 TPS 또한 증가합니다. 또한
Plasma-EVM의 **블록 가스리밋**과 **블록 타임**은 Plasma EVM의 TPS에 직접적인 영향을 끼칩니다. 그렇기 때문에
Plasma-EVM의 블록 가스리밋과 블록 타임을 조절해서 TPS를 낮추거나 높일 수 있습니다.

지금부터는 Plasma-EVM 성능 테스트를 누구나 해볼 수 있도록 가이드를 제공하고자 합니다.

> **Guide to measure TPS on Plasma EVM**:
> [https://www.notion.so/onther/Guide-to-measure-TPS-on-Plasma-EVM-56b77c9920c54679a93181525ef0e02c](https://www.notion.so/onther/Guide-to-measure-TPS-on-Plasma-EVM-56b77c9920c54679a93181525ef0e02c)

우선 Plasma-EVM 성능 테스트를 하기 위해선 3개의 프로젝트를 clone해야 합니다.

* go-ethereum: `git clone https://github.com/Onther-Tech/go-ethereum.git`
* plasma-evm: `git clone https://github.com/Onther-Tech/plasma-evm.git`
* chainload: `git clone https://github.com/Onther-Tech/chainload.git`

#### go-ethereum

저희는 내부적으로 Plasma-EVM을 테스트하기 위해 루트체인으로 로컬 네트워크를 사용합니다. 아래와 같이 실행하면 RootChain을 실행할
수 있습니다.

1.  `git clone https://github.com/Onther-Tech/go-ethereum.git`
1.  `cd go-ethereum`
1.  `bash run.rootchain.sh`

`run.rootchain.sh` 스크립트는 다음의 역할을 수행합니다.

1.  **블록 타임을 14초로 설정해 14초마다 새로운 블록이 생성됩니다.** 블록 타임을 14초로 설정한 이유는 이더리움의 메인넷과 가장 비슷한
환경을 제공하기 위함입니다.
1.  **오퍼레이터 어카운트에 충분한 ETH를 할당해둡니다.** 그 이유는 오퍼레이터가 블록을 제출하기 위해서는 ETH를 충분히 가지고 있어야 하기
때문입니다.

#### plasma-evm

Plasma EVM을 구동하기 위해서는 아래의 명령어를 실행합니다.

1.  `git clone https://github.com/Onther-Tech/plasma-evm.git`
1.  `cd plasma-evm`
1.  `git checkout measure-tps`
1.  `bash run.pls.sh`

TPS를 측정을 위한 measure-tps 브랜치를 사용합니다. 이후 run.pls.sh 스크립트를 실행해 Plasma-EVM을 구동시킵니다.

run.pls.sh 스크립트는 다음의 역할을 수행합니다.

1.  **RootChain에 RootChain contract를 배포합니다.** 플라즈마 체인이 정상적으로 동작하기 위해서는 RootChain
contract가 필요합니다.
1.  **블록 타임을 14초로 설정합니다.** Plasma-EVM의 블록 타임은 이더리움 메인넷의 평균 블록 타임과 같은 14초입니다.
1.  **여러 EOA 어카운트에 충분한 PETH를 할당해둡니다 .**플라즈마 체인에서 트랜잭션을 보내기 위해서는 PETH; Plasma ETH가
필요합니다.

또한 `cache` 플래그 값과 `txpool.globalslots` 플래그 값을 높게 설정해야 합니다. 하나의 노드에 대량의 트랜잭션을 보내기
때문에, Plasma EVM에서 이를 받을 수 있도록 설정되어야 합니다.

1.  `--cache 4096`: Megabytes of memory allocated to internal caching (default:
1024)
1.  `--txpool.globalslots 163840`: Maximum number of executable transaction slots
for all accounts (default: 4096)

#### chainload

이제 Plasma EVM을 실행한 노드에 대량의 트랜잭션을 보내야 합니다. 이를 위해 저희는 GoChain에서 개발한
[chainload](https://github.com/gochain-io/chainload) 프로젝트를 발견하고 사용했습니다.
chainload를 이용해서 원하는 TPS만큼의 트랜잭션을 보낼 수 있습니다.

chainload는 GoChain 성능 평가를 위해 개발된 것으로 Plasma EVM에서 사용할 수 있도록 몇 가지 코드를 수정 해야 했습니다.
그렇기 때문에 Plasma EVM에서 TPS 측정을 할 땐 chainload 프로젝트를 포크해 수정한 버전을 사용합니다.

1.  `git clone https://github.com/Onther-Tech/chainload.git`
1.  `cd chainload`
1.  `gvm use 1.12 make`
1.  `make`

chainload는 GO의 버전이 1.12 이상일 때 동작합니다. 그렇기 때문에 gvm을 이용해서 1.12 버전을 사용하도록 만듭니다. 이후
make를 해주면 chainload를 사용할 수 있게 됩니다. 이후 아래의 명령어로 대량의 트랜잭션을 노드에 보내게 됩니다. chainload
명령어 플래그는 [여기](https://github.com/gochain-io/chainload#how-to-use)서 참고하시길 바랍니다.

`./chainload -id 16 -urls http://127.0.0.1:8549 -tps 1000 -senders 100 -v`

### 결과

measure-tps 브랜치에서 Plasma EVM은 `$HOME/.pls.dev` 디렉토리를 사용합니다. 이 디렉토리에서 geth attach
명령어를 이용해 콘솔로 연결할 수 있습니다.

    cd $HOME/.pls.dev
    geth attach geth.ipc

이후 콘솔창에서 다음의 스크립트를 실행시킵니다.

    var lastTime = 0
    for (i = 1; i <= web3.eth.blockNumber; i++ ) {
      console.log(i, web3.eth.getBlock(i).transactions.length,
      web3.eth.getBlock(i).timestamp - lastTime);   
      lastTime = web3.eth.getBlock(i).timestamp
    }

Plasma EVM에서 충분히 블록을 만든 뒤 해당 스크립트를 실행시키면 아래와 같은 결과 값을 확인할 수 있습니다.

    1 100 1555487212
    2 7321 35
    3 14274 14
    4 14276 14
    5 14276 38
    6 14277 14
    7 14277 40
    8 14276 14
    9 14276 35
    10 14276 14
    11 14277 50
    12 14275 14
    13 14275 40

    ...

가장 앞에 나온 값은 **블록 번호**이고 두 번째는 **블록 하나의 포함된 트랜잭션 개수**입니다. 그리고 마지막 숫자 값은 **블록
타임**입니다. 결과 값을 보면 평균적으로 블록 하나의 포함된 트랜잭션 개수가 14000여개 정도입니다. 블록 타임이 14초일 때 1초당
1000개의 트랜잭션을 처리하는 것을 확인할 수 있습니다.

#### 한계

그런데 블록 타임을 보면 블록 타임이 14초이거나 14초가 아닐 때가 있습니다. 블록 타임이 14초를 넘어가는 경우는 해당 블록이 제출되고
RootChain contract로부터 이벤트를 기다려야 하는 시간이 더해진 것입니다. 이러한 점을 고려했을 때 RootChain 네트워크의
상태에 따라 Plasma EVM의 TPS에 영향을 끼칠 수 있습니다.

이를 해결하기 위해 Plasma EVM에서 블록을 생성하는 것과 생성된 블록을 제출하는 것을 병렬로 처리하려는 시도가 이루어지고 있습니다. 또한
메인넷의 성능 또한 계속 발전되고 있기 때문에 더 많은 연구 개발이 이루어지면 이 문제는 충분히 개선될 것으로 기대합니다.

* [Plasma Evm](https://medium.com/tag/plasma-evm?source=post)

### [신건우(Thomas Shin)](https://medium.com/@thomashin)

### [Onther-Tech](https://medium.com/onther-tech?source=footer_card)

Building an Ethereum Blockchain ECO system to Change the World

{:toc}
{% include youtubePlayer.html id=page.youtubeId %}
{% include slidePlayer.html id=page.slideWebId %}
