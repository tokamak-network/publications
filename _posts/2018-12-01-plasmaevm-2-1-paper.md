---
layout: post
title:  플라즈마 EVM 2.1 - 탈중앙화된 상태를 강제하는 튜링 완전한 플라즈마
date:   2018-12-19 00:00:00 +09:00
author: "박주형(Carl)"
categories: ethereum scalability plasma plasmaEVM
tag: [ethereum, scalability, plasma, plasmaEVM]
youtubeId :
slideWebId :
use_math : true
---

* content
{:toc}
{% include youtubePlayer.html id=page.youtubeId %}
{% include slidePlayer.html id=page.slideWebId %}

### [Change Logs](https://hackmd.io/s/BJ8v8YggV)
플라즈마 EVM이 2.1 버전으로 업데이트 되었습니다. 변경 내역은 위의 change logs를 확인하세요.

## Abstract
이더리움의 EVM(Ethereum Virtual Machine)은 블록체인에서 튜링 완전한 연산을 지원함으로써 스마트 컨트랙트(smart contract)라는 이름으로 잘 알려진 일반 프로그램(general program)을 동작가능하게 하였다. 플라즈마 EVM은 플라즈마 체인에서 EVM을 실행할 수 있는 새로운 형태의 플라즈마이며, go-ethereum, py-evm, parity와 같은 현재 이더리움 클라이언트에 기반을 두고있다. 이 페이퍼에서는 자식체인의 검증된 상태만이 루트체인에 제출되는 것을 보장하기위해 두 체인 사이의 account storage에서 enter, exit하는 방법을 제공하여 상태가 강제되는(state-enforceable) 플라즈마 구조를 제안한다. 이는 두 체인이 완전히 동일한 구조를 갖기 때문에 가능하다. 플라즈마 EVM은 기존 이더리움 체인에 구축된 탈중앙화어플리케이션(Decentralized Application; Dapp)을 그대로 플라즈마 체인으로 옮겨와 댑(Dapp)의 탈중앙성, 성능, 안정성, 사용성 모두를 개선시킬 수 있다.


# Glossary
## General
- **루트체인(Root chain)**: 이더리움 블록체인
- **자식체인(Child chain)**: 플라즈마 블록체인
- **오퍼레이터(Operator)**: 자식체인을 운영하는 주체
- **널 어드레스(NULL_ADDRESS)**: 주소가 `0x00` or `0xFF...FF(2^160 - 1)`이고 논스(Nonce)와 서명값이 `v = r = s = 0`인 계정, NA로 표기한다.
- **트랜잭터(Transactor)**: 트랜잭션을 생성하는 어카운트(Account), `tx.origin`.
- **루트체인 컨트랙트(RootChain contract)**: ETH나 ERC20등에 대한 enter / exit 작업을 관리하는 Root chain의 플라즈마 컨트랙트, $R$로 표기한다.
- **요청가능 컨트랙트(Requestable contract)**: Root chain과 Child chain 양쪽에서 exit/enter request를 처리할 수 있는 컨트랙트이다. 2개의 완전히 동일한 컨트랙트가 root chain과 child chain에 모두 배포되어야 하고, $R$을 통해 두 컨트랙트의 주소가 매핑(Mapping)된다.
- **진입요청(enterRequest)**: Root chain에서 child chain으로 무언가를 이동시키기 위해 필요한 request를 의미한다. eg) Root chain에서 보유하고 있는 자산을 Child chain으로 예치하는 행위, Account storage 변수를 child chain으로 이동시키는 행위 등.
- **퇴장요청(exitRequest)**: Child chain에 예치되었던 자산이나 Account storage를 다시 Root chain으로 이동시키기 위해 필요한 Request를 의미한다. Root chain에 대한 exitRequest는 Child chain의 Account storage의 상태를 즉시 갱신해야 하며 Child chain에서의 갱신이 거절되었다면(트랜잭션이 revert 된 경우), Child chain을 갱신할 때 실행되었던 연산 결과를 증거로 삼아 `exitChallenge`를 할 수 있다.
- **요청블록(requestBlock)**: Root chain에 의해 상태 변환(state transition)에 대한 적용이 강제되는 블록
- **비요청블록(nonRequestBlock)**: Child chain에서 어카운트 간에 발생하는 일반적인 트랜잭션만을 포함하는 블록



<br>

## Challenges


- **요청 챌린지(requestChallenge)**: `requestBlock`이 정상적인 request를 포함하지 않거나 유효하지 않은 request를 포함하는 경우에 대한 챌린지
- **널 어드레스 챌린지(nullAddressChallenge)**: `nonRequestBlock`이 NULL_ADDRESS가 전송한 트랜잭션을 포함하는 경우에 대한 챌린지
- **연산 챌린지(computationChallenge)**: `requestBlock`이나 `nonRequestBlock`에서 상태 변환에 대한 연산이 올바르게 이루어지지 않는 경우에 대한 챌린지
- **연산 챌린지 기간(computationChallenge period)**: 연산 챌린지 기간. $CP_{computation}$으로 표기한다.
- **퇴장 챌린지(exitChallenge)**: 유효하지 않은 `exitRequest`가 자식체인에서 받아들여지지 않는 경우에 대한 챌린지 (단, request는 `requestBlock`에 포함되어야 한다)
- **퇴장 챌린지 기간(exitChallenge)**: 퇴장 챌린지 기간. $CP_{request}$으로 표기한다.
- **확정(Finalized)** : 모든 블록은 제출된 시점을 기준으로 연산 챌린지 기간 이후 결정적으로(deterministically) 확정된 상태가 된다. 또한 모든 퇴장요청은 퇴장 챌린지 기간 이후 결정적으로 확정된 상태가 된다.


<br>


## Solution to Data availability problem
* **유저에 의한 포크(User-Activated fork; UAF)** : 사용자에 의해 발생된 포크(Fork)이다. Data availability problem에 대한 탈출 수단으로 사용된다.
* **유저 요청 블록(User Request Block;$URB$)** : 사용자에 의해 제출되는 요청 블록(Request Block). 기존의 요청 블록과 달리 본인 혹은 다른 사용자의 탈출 요청(Exit Request for URB) 트랜잭션만 포함된 블록이다.
* **오퍼레이터 요청 블록(Operator Request Block;$ORB$)** : 오퍼레이터에 의해 제출된 요청 블록. 플라즈마 EVM의 요청 블록과 같다.
* **오퍼레이터를 통한 탈출 요청(Exit Request for ORB;$ERO$)** : 오퍼레이터 요청 블록($ORB$)을 통한 탈출 요청 트랜잭션
* **유저에 의한 탈출 요청(Exit Request for URB;$ERU$)** : 유저 요청 블록($URB$)을 통한 탈출 요청 트랜잭션
* **재정렬(Rebase)** : 가장 최근의 확정된 블록을 기준으로 유저 요청 블록($URB$)이 제출될 경우 확정(finalized)되지 않은 모든 플라즈마 블록들은 해당 유저 요청 블록($URB$)의 뒤에 위치하게 되며 유저 요청 블록($URB$)과 상충되는 트랜잭션은 되돌려지게(revert)된다. 이를 재정렬(Rebase)이라 한다.
* **유저 요청 블록 제출비용($C_{URB}$)** : 만들어진 유저 요청 블록($URB$)을 루트체인에 제출(Commit)하는데 제출자(Commiter)에게 부과되는 비용
* **유저 탈출 요청 제출비용($C_{ERU}$)** : 유저에 의한 탈출 요청($ERU$)이 블록에 포함되어 제출(Commit)될때 각 유저가 부담하는 비용
* **유저 탈출 요청 수($N_{ERU}$)** : 유저 요청 블록($URB$)에 포함된 유저 탈출 요청 트랜잭션($ERU$)의 수
* **총 유저 요청 블록 비용(Total Cost for User Request Block; $C_{URB}^{total}$)** : 유저 요청 블록을 루트체인에 제출(Commit)할 때 발생하는 총 비용. $C_{URB}^{total}$ = $C_{URB}$ + $N_{ERU}$ * $C_{ERU}$로 계산된다.

<br>

---

# Child chain
플라즈마 EVM의 클라이언트는 기존 이더리움 클라이언트에 기반했기 때문에 대부분의 기본 메커니즘은 이더리움과 동일하다. 자식체인의 블록, 트랜잭션 그리고 Receipt등의 자료구조는 이더리움과 같다. 다만 기존의 이더리움과 다르게 플라즈마 프로토콜을 적용하기 위해 몇가지 변경점이 존재한다.

1. 자식체인을 사용하려는 루트체인의 사용자 A의 자산(ETH/ERC20)이 루트체인 컨트랙트에 예치되면, A의 enterRequest는 `enterRequests` 대기열에 등록되고 오퍼레이터(혹은 NULL_ADDRESS)에 의해 다음 자식체인의 블록에 A의 자산을 *발행* 하는 트랜잭션이 생성된다. 루트체인에서 자식체인으로의 상태 변환은 이러한 방식으로 강제된다.

2. 사용자 A가 자식체인에서 exit하기를 원한다면, 해당 증거(Proof)를 포함하여 exitRequest를 제출해야 한다. 해당 exitRequest는 `exitRequests` 대기열에 등록되고 오퍼레이터(혹은 NULL_ADDRESS)는 반드시 A의 자산을 *소각* 하는 트랜잭션을 다음 자식체인의 블록에 생성해야 한다.

3. 루트체인의 exitRequest가 챌린지 없이 확정되면, 자식체인에는 더 이상 어떤 변화도 발생하지 않는다. 그러나 *루트체인* 의 컨트랙트는 **반드시** 루트체인에 해당 변경 사항을 적용해야 한다.

4. 만약 다음 블록이 Request와 관련된 트랜잭션을 올바르게 포함하지 않는다면, 이는 챌린지에 대한 근거가 될 수 있다. 그리고 챌린지가 받아들여지면 루트체인은 자식체인의 상태를 Request 트랜잭션이 발생하기 이전 블록으로 되돌린다.

오퍼레이터는 `requestBlock`과 `nonRequestBlock`, 두 가지 종류의 블록을 루트체인에 제출해야 한다. 루트체인 컨트랙트는 이 두 종류의 블록을 유기적으로 처리하기 위해 상황에 맞게 `WaitingNonRequestBlock`, `WaitingRequestBlock`, `AcceptingRequest`인 상태로 변한다. 위 세 가지 상태와 Request들이 어떤 방식으로 처리되는지 알아보도록 하자.

![](https://i.imgur.com/sxq9LrZ.png)
*<center>그림 1 : State transition of Root chain contract</center>*

1. 오퍼레이터는 반드시 새로운 블록을 제출하기 전에 `prepareToSubmit` 함수를 호출해야 한다. 그러면 루트체인 컨트랙트는 `WaitingNonRequestBlock`인 상태가 된다. 그 다음 오퍼레이터는 `nonRequestBlock`과 `requestBlock`을 모두 제출한다. 새로운 request들은 `WaitingNonRequestBlock`인 상태가`AcceptingRequest` 로 바뀔 때까지 펜딩(Pending)된다.

2. 먼저, `nonRequestBlock`은 반드시 enter/exitRequest와 관련 없는 모든 트랜잭션만을 포함하여야 한다. 자식체인에서 생성된 트랜잭션이 없다면, `transactionRoot`, `postTransactionalStatesRoot`는 `emptyTrie`를 가리킨다. 만약 `nonRequestBlock`가 NULL_ADDRESS로 부터 전송되는 트랜잭션이 포함하거나 (nullAddressChallenge), 상태 변환이 잘못된 방법으로 이루어 졌을 경우(computationChallenge), 비잔틴(Byzantine) 상황으로부터 자식체인을 복구하기 위해 챌린지 될 수 있다.

- 오퍼레이터가 `nonRequestBlock`을 제출한 경우, root chain은 `WaitingRequestBlock` 상태가 된다.

3. `requestBlock`은 오직 Request와 관련된 트랜잭션들만 포함하여야 한다. 따라서 `requestBlock`은 enter&exitRequest의 적용을 위해, NULL_ADDRESS을 트랜잭터로 하는 해당 Request와 관련된 컨트랙트 어카운트에 대한 상태 강제(state-enforcing) 트랜잭션만을 포함해야 한다.

- `requestBlock`이 Request와 관련없는 트랜잭션을 포함하거나 포함되어야 할 Request를 제외하면, 챌린지가 발생하게 되고 해당 블록은 챌린지 되기 전 상태로 Revert 된다. 이러한 종류의 챌린지를 `requestChallenge`라고 한다.
- 오퍼레이터가 `requestBlock`을 제출하면, 루트체인은 `AcceptingRequest` 상태가 된다.


루트체인은 `requestBlock`을 통해 자식체인의 상태 변화를 강제한다. 오퍼레이터가 이러한 상태 변화를 제대로 적용하지 않아 이에 대한 챌린지를 받는다면 현재 블록에서 이전 블록으로 Revert 된다.(*requestChallenge*)

![](https://i.imgur.com/IYZYV5v.png)
*<center>그림 2 : Request and challenge diagram</center>*


### Epoch
Epoch는 N($N>=2$)개의 블록들을 갖고 있고, N-1개의 `nonRequestBlock`과 1개의 `requestBlock`으로 이루어져 있다. *그림 2* 는 N이 2인 경우에 대한 설명이다. 그러나 epoch 길이를 확장하여 `requestBlock`과 `nonRequestBlock`의 비율을 변경할 수도 있다.

<br>

## Block submission
Operator는 각 블록마다 `stateRoot`, `transactionsRoot`, `postTransactionalStatesRoot`(또는 `intermediateStatesRoot`)이 세 종류의 머클 루트를 제출한다. 상태 변환에 대한 연산이 제대로 실행되지 않았다면 `computationChallenge`를 통해 올바른 상태를 가지고 있는 블록으로 복구 된다. 이 방식은 TrueBit의 verification game과 유사한 방식이며, pre-stateRoot 및 post-stateRoot를 사용하여 '트랜잭션 별 상태 변환 함수'를 확인하는 방식으로 해결한다.


## Finality
플라즈마 EVM은 챌린지 기간을 어떠한 유효한 챌린지 없이 넘긴 블록에 대해 플라즈마 XT처럼 블록의 체크포인트를 지정할 수 있다. 이처럼 블록에 Finality를 부여하여 루트체인 컨트랙트를 효율적으로 구현할 수 있다.


# Account Storage Enter & Exit
플라즈마 EVM에서는 어떠한 유형의 Account storage 변수도 루트체인과 자식체인을 자유롭게 이동 할 수 있다. Storage 변수를 enter/exitRequest에 사용 할 수 있는 이유는 storage는 trie 구조로 되어있고, storage 내의 모든 값에 해당 키를 사용하여 접근 할 수 있기 때문이다. Enter 와 Exit은 계정의 **단일 storage 변수** 를 한 체인에서 이에 상응하는 컨트랙트의 다른 체인으로 이동하는 것을 의미한다.

## Notation

NULL_ADDRESS: $NA$
Operator: $O$
User of plasma: $U$
Challenger: $Ch$
RootChain contract: $a_R$

Root chain *Requestable contract* 의 주소: $a_r$
Child chain *Requestable contract* 의 주소: $a_c$

* (각 체인에는 단 하나의 컨트랙트 주소만 있다.)

Root chain의 컨트랙트 어카운트 storage 변수: $σ[a_r]s[k]$, 이를 간단히 하면 $A^{r}_{s}[k]$ 와 같이 나타낼 수 있다.

child chain의 컨트랙트 어카운트 storage 변수: $σ[a_c]s[k]$, 이를 간단히 하면 $A^{c}_{s}[k]$ 와 같이 나타낼 수 있다.
(단, $k$ 는 $trieKey$이고 $A_s$는 $σ[a]s$ 일 때)

Trie의 *임의의 storage 변수* 는 $A_s[k]$ 이고 루트체인과 자식체인의 요청가능 컨트랙트 어카운트를 $a_r$ 과 $a_c$ 라 하자.
이들의 storage 변수는 각기 $$A^{r}_{s}[k]$$ 와 $A^{c}_{s}[k]$ 이다. Storage 변수의 enter/exitRequest는 storage 변수가 $$A^{r}_{s}[k]$$ 와 $$A^{c}_{s}[k]$$ 사이를 이동하는 것으로 정의한다. Enter는 $A^{r}_{s}[k]$ 을 바탕으로 $$A^{c}_{s}[k]$$ 가 변경되는 과정이고 Exit은 $$A^{c}_{s}[k]$$ 를 바탕으로 $$A^{r}_{s}[k]$$ 가 변경되는 과정이다. 단, 상태를 변경하는 구체적인 방법은 구현 방식에 따라 달라질 수 있다. Storage의 변경을 적용하기 위해서는 $a_r$과 $a_c$ 가 동일한 바이트코드를 가져야 하며, 이는 두 컨트랙트가 같은 storage 레이아웃을 갖는다는 것을 의미한다.(이는 두 컨트랙트의 codeHash를 비교하는 것으로 검증 가능하다.)

그러나 요청가능 컨트랙트라 하더라도 해당 컨트랙트 안의 모든 변수가 요청가능(Requestable)일 필요는 없다. Request와 관련된 기능이 필요 없는 변수도 있고, 특정 변수에 대한 Request 권한을 사용자별로 구분해야 하는 경우도 고려해야 하기 때문이다. 예를 들어, 누구나 타인의 토큰 밸런스에 대해 Request 할 수 있어서는 안되기 때문이다. 따라서 우리는 미리 컨트랙트 함수에 이러한 문제를 해결할 수 있는 로직을 배치하고자 한다. 이는 변수가 선언 된 순서와 trieKey를 이용하여 특정 변수에 대한 Request 권한을 확인 할 수 있기에 가능하다. 자세한 내용은 아래의 의사 코드에 대해 다룬 섹션에서 살펴 볼 것이다.

우선, `Enter`와 `exit`의 구체적인 과정에 대해 살펴보도록 하겠다.



## Enter
![](https://i.imgur.com/pqGxsfT.png)
*<center>그림 3 : Enter diagram</center>*

<br>

1. 사용자 $U$는 $a_r$, `trieKey` 및 `trieValue`를 인자로하여 $a_R$의 `enter()` 함수를 호출하여 $a_r$에 대응하는 $a_c$에 $A^{r}_{s}[k]$를 enter 한다.
2. `enter()` 함수에서, $a_r$의 `applyRequestInRootchain()` 함수가 호출된다.
3. 2번 과정이 Revert 된다면, enterRequest를 처리하는 과정이 중단된다. Revert 되지 않는다면 $R$에 `enterRequest` 이벤트가 생성된다.
4. $O$는 다음`requestBlock`에 $NA$을 트랜잭터로 하는 `enterRequest` 처리 트랜잭션을 포함시켜야 한다. $O$가 `enterRequest`를 블록에 제대로 포함시키지 않는다면 $Ch$는 이에 대해 `requestChallenge`를 할 수 있다.
5. 4번 과정의 트랜잭션에서 $NA$는 $a_c$의`enterRequest`에 따라`applyRequestInChildchain()` 함수를 실행한다. 이로 인해 $A^{c}_{s}[k]$가 $A^{c}_{s}[k]'$로 성공적으로 변환된다. 이러한 상태 변환 과정이 올바르게 이루어지지 않으면 $Ch$는`computedChallenge`를 할 수 있다.

## Exit
![](https://i.imgur.com/vhMVtaA.png)
*<center>그림 4 : Exit diagram</center>*

<br>

1. 사용자 $U$는 $a_r$로 $A^{c}_{s}[k]$를 exit 하기 위해, $a_r$,`trieKey` 및`trieValue`를 인자로 하여 $a_R$에서`startExit()`을 호출한다.
2. $U$의 `exitRequest`는 $a_R$에서 생성되며 $O$에 의해 다음`requestBlock`에 포함된다. $O$가 `exitRequest`를 포함하지 않는다면 ,$Ch$는 이에 대해 `requestChallenge`를 할 수 있다.
3. 다음 `requestBlock`에는 $NA$가`exitRequest` 절차에 따라 $a_c$에서`applyRequestInChildchain()`함수를 실행하는 트랜잭션이 포함되어야 한다. *3번 과정* 이 올바르게 실행되지 않으면 $Ch$는`computationChallenge`를 할 수 있다.
4. *3번 과정* 이 제대로 실행되었음에도 트랜잭션이 Revert 된다면, $Ch$는 트랜잭션의 Output을 증거로 하여 `exitChallenge`를 통해 `exitRequest`를 챌린지할 수 있다.
5. *4번 과정* 에서 챌린지가 발생하지 않는다면, `exitRequest`가 `finalize` 된다. 그러나 만약`exitChallenge`가 타당하다는 것이 증명되면 `exitRequest`는 취소된다.
6. $U$는 $a_R$에서`finalizeExit()`함수를 호출하여 `exitRequest`를 `finalize` 할 수 있다. 이어 $a_R$은 $a_r$의 `applyRequestInRootchain()` 함수를 호출한다. `applyRequestInRootchain()` 함수가 실행 결과, $$A^{r}_{s}[k]$$ 에서 $$A ^{r}_{s}[k]'$$ 로의 변환이 성공적으로 이루어진다. 컨트랙트 개발자는 $a_r$ 의`applyRequestInRootchain ()` 함수가 throw되면 안된다는 것을 **반드시** 확인 해야한다. 이러한 변환은 반드시 이루어져야 하며, Exit의 유효성은`exitChallenge`에 대한 증명으로 사용될 수 있는 $a_c$의`applyRequestInChildChain()`함수에서 확인 할 수 있다.

<br>

## RootChain and Requestable Contract

루트체인의 루트체인 컨트랙트는 `RequestableRootChain` 인터페이스를 가지고 있다. Requestable 컨트랙트는`enterRequest`가 storage 변수를 $a_r$에서 $a_c$로 enter하게 할 수 있다. 사용자는`rootchain.enter`로 직접 enter request를 할 수 있다.

```solidity
interface RequestableRootChain {
  function getExitFinalized(uint256 requestId) public returns(bool);

  function startExit(
    address contractAddress,
    bytes32 trieKey,
    bytes32 trieValue
  ) returns (bool);

  // User can directly make an enter request with trieKey and trieValue.
  function enter(
    address contractAddress,
    bytes32 trieKey,
    bytes32 trieValue
  ) returns (bool);

  // Requestable contract can initialize an enter request with proper
  // trieKey and trieValue. This makes user not to take care of what is exact
  // trieKey for `balances[requestor]`.
  function makeEnterRequest(
    address requestor,
    bytes32 trieKey,
    bytes32 trieValue
  ) returns (bool);

  // finalize exits
  function finalizeExit() public returns (bool);
}

interface Requestable {
  // check the request is applied or not in root chain.
  function setRequestApplied(uint256 requestId) internal returns (bool);
  function getRequestApplied(uint256 requestId) internal returns (bool);

  // Apply storage changes in root chain for exit and enter reuqests.
  // For enter request, This is called in the middle of `rootchain.enter`
  // or `rootchain.makeEnterRequest` to initialize an enter request.
  // If this returns true, the enter request is queued into `enterRequests`.
  // For exit request, This is called in  `rootchain.finalizeExit`.
  function applyRequestInRootChain(
    bool isExit,
    uint256 requestId,
    address requestor,
    bytes32 trieKey,
    bytes32 trieValue
  ) external returns (bool success);

  function applyRequestInChildChain(
    bool isExit,
    uint256 requestId,
    address requestor,
    bytes32 trieKey,
    bytes32 trieValue
  ) external returns (bool success);
}
```

간단한 Requestable 토큰 컨트랙트의 예시는 다음과 같다.

```solidity
contract RequestableToken is Requestable {
  // 3 requestable variables.
  address owner;     // `owner` is stored at bytes32(0).
  uint totalSupply;  // `totalSupply` is stored at bytes32(1).

  // `balances[addr]` is stored at keccak256(bytes32(addr), bytes32(2)).
  mapping(address => uint) balances;

  RequestableRootChain rootchain;
  address NULL_ADDRESS = address(0); // or 0xFF..FF

  // this is only called by the RootChain contract in root chain
  // when i) enterRequest is initialized or
  //     ii) exitRequest is finalized
  function applyRequestInRootChain(
    bool isExit,
    uint256 requestId,
    address requestor,
    bytes32 trieKey,
    bytes32 trieValue
  ) external returns (bool success) {
    require(msg.sender == address(rootchain));
    require(!getRequestApplied(requestId)); // check double applying

    if (isExit) {
      // exit must be finalized.
      require(rootchain.getExitFinalized(requestId));

      if(bytes32(0) == trieKey) {
        // only owner (in child chain) can exit `owner` variable.
        // but it is checked in applyRequestInChildChain and exitChallenge.

        // set requestor as owner in root chain.
        owner = requestor;
      } else if(bytes32(1) == trieKey) {
        // no one can exit `totalSupply` variable.
        // but do nothing to return true.
      } else if (keccak256(requestor, 2) == trieKey) {
        // this checks trie key equals to `balances[requestor]`.
        // only token holder can exit one's token.
        // exiting means moving tokens from child chain to root chain.
        balances[requestor] += uint(trieValue);
      } else {
        // cannot exit other variables.
        // but do nothing to return true.
      }
    } else {
      // apply enter
      if(bytes32(0) == trieKey) {
        // only owner (in root chain) can enter `owner` variable.
        require(owner == requestor);
        // do nothing in root chain
      } else if(bytes32(1) == trieKey) {
        // no one can enter `totalSupply` variable.
        revert();
      } else if (keccak256(bytes32(requestor), bytes32(2)) == trieKey) {
        // this checks trie key equals to `balances[requestor]`.
        // only token holder can enter one's token.
        // entering means moving tokens from root chain to child chain.
        require(balances[requestor] >= uint(trieValue));
        balances[requestor] -= uint(trieValue);
      } else {
        // cannot apply request on other variables.
        revert();
      }
    }

    setRequestApplied(requestId);
    return true;
  }

  // this is only called by NULL_ADDRESS in child chain
  // when i) exitRequest is initialized by startExit() or
  //     ii) enterRequest is initialized
  function applyRequestInChildChain(
    bool isExit,
    uint256 requestId,
    address requestor,
    bytes32 trieKey,
    bytes32 trieValue
  ) external returns (bool success) {
    require(msg.sender == NULL_ADDRESS);

    if (isExit) {
      if(bytes32(0) == trieKey) {
        // only owner (in child chain) can exit `owner` variable.
        require(requestor == owner);

        // do nothing when exit `owner` in child chain
      } else if(bytes32(1) == trieKey) {
        // no one can exit `totalSupply` variable.
        revert();
      } else if (keccak256(bytes32(requestor), bytes32(2)) == trieKey) {
        // this checks trie key equals to `balances[tokenHolder]`.
        // only token holder can exit one's token.
        // exiting means moving tokens from child chain to root chain.

        // revert provides a proof for `exitChallenge`.
        require(balances[requestor] >= uint(trieValue));

        balances[requestor] -= uint(trieValue);
      } else { // cannot exit other variables.
        revert();
      }
    } else { // apply enter
      if(bytes32(0) == trieKey) {
        // only owner (in root chain) can make enterRequest of `owner` variable.
        // but it is checked in applyRequestInRootChain.

        owner = requestor;
      } else if(bytes32(1) == trieKey) {
        // no one can enter `totalSupply` variable.
      } else if (keccak256(bytes32(requestor), 2) == trieKey) {
        // this checks trie key equals to `balances[tokenHolder]`.
        // only token holder can enter one's token.
        // entering means moving tokens from root chain to child chain.
        balances[requestor] += uint(trieValue);
      } else {
        // cannot apply request on other variables.
      }
    }

    return true;
  }

  // make an enter request for `owner` variable.
  function enterOwner() external returns (bool) {
    require(msg.sender == owner);

    require(rootchain.makeEnterRequest(msg.sender, bytes32(0), owner));

    return true;
  }

  // make an enter request for `balances[msg.sender]`.
  // If user know trieKey, one can call `rootchain.enter` directly.
  function enterBalance(uint amount) exernal returns (bool) {
    // it is also checked in applyRequestInRootChain
    require(balances[msg.sender] >= amount);

    bytes32 trieKey = keccak256(bytes32(msg.sender), bytes32(2));

    // this subtracts `balances[msg.sender]` by `amount`
    require(rootchain.makeEnterRequest(msg.sender, trieKey, amount);

    return true;
  }

  // User can get the trie key of one's balance and make an enter request directly.
  function getBalanceTrieKey(address who) public pure returns (bytes32) {
    return keccak256(bytes32(who), bytes32(2));
  }
}
```

<br>

# Challenges

## Request Challenge
`requestChallenge`를 위하여 RootChain 컨트랙트는 Request의 유효성 검증을 위한 트랜잭션 데이터를 구성할 수 있다. 트랜잭션 의 `from'`은 NULL_ADDRESS, `to`는 자식체인의 요청가능 컨트랙트의 주소, `value`는 0, `data`는 request 유형과 파라미터에 의해 정의되고 `nonce`는 각 Request마다 순차적으로 증가한다. 실제 트랜잭션 데이터가 루트체인 컨트랙트에 의해 구성되는 데이터와 일치하지 않는다면, `requestBlock`이 이전 `nonRequestBlock`으로 되돌릴 것이다. Request에 대한 연산은 `computationChallenge`에 의해 검증 된다.

<br>

## NullAdress Challenge
`nonRequestBlock`에 트랜잭터가 NULL_ADDRESS인 트랜잭션이 있는지 확인한다.

<br>

## Exit Challenge
`requestBlock`에 포함된 `exitRequest`가 *Revert* 되었다면, 해당 `exitRequest` 는 **반드시** 루트체인 컨트랙트에서 삭제되어야 한다. Invalid exitRequest에 대한 `exitChallenge` 절차는 revert 된 트랜잭션을 증거로 삼아 처리된다. 그러나 `exitRequest`에 대해 챌린지하기 전에, 해당 Request를 포함하고 있는 블록이 일단 확정 되어야 한다. 그 이유는 유효한 블록에서 Revert되지 않는 트랜잭션이 유효하지 않은 블록에서는 Revert될 수 있기 때문이다. 따라서 `exitChallenge`는 `requestBlock`이 확정된 후에 시작 될 수 있다.

<br>

## Computation Challenge
연산 챌린지(computationChallenge)는 오퍼레이터가 트랜잭션을 올바르게 실행했는지를 검증한다. 즉 상태가 미리 정의된 방식으로 변경 되었는지를 확인하는 것이다(예: 스마트 컨트랙트 바이트 코드). 오퍼레이터가 잘못된 `stateRoot`를 제출하면, 이는 상태 변환이 올바르게 이루어지지 않았다고 판단할 수 있는 근거가 된다. 그러면 `txData`,`preStateRoot` 및`postStateRoot`와 함께 TrueBit의 Verification game과 유사한 방법을 사용하여 챌린지 될 수 있다.


$$postState = STF_{tx}(preState, TX_i)$$

$$postState = postTransactionalStates[i]$$


$$
preState=\begin{cases}
    postTransactionalStates[i-1] & \text{if}\ i>0 \\
    previousStateRoot & \text{otherwise}\end{cases}$$

RootChain 컨트랙트에서 $STF_{tx}$를 시뮬레이션하고 output을 이미 제출 된 output과 비교하여 연산이 올바르게 이루어졌는지에 대한 여부를 확인할 수 있다.



## Verification Game
TrueBit은 outsource된 연산을 검증하기 위한 방법으로 *verification game* 을 제안했다. 그러나 TrueBit이 제안한 게임의 마지막 단계는 이더리움에서 연산을 한 번 수행하고 실제 output과 예상 output을 비교하는 방법을 사용한다. 하지만 우리는 Ohalo와 PARSEC에서 구현해 왔던 EVM 내부에서 EVM을 실행하는 스마트 컨트랙트인 solevm을 사용하여 연산 결과를 검증하고자 한다.


![](https://i.imgur.com/BVFY09y.png)
*<center>그림 5 : verification game</center>*

<br>

그러나 [수수료 위임 체인](https://hackmd.io/s/SkxNKAXU7)을 사용하려면 EVM을 통한 트랜잭션 실행을 포괄해야 한다.

## Challenge period
플라즈마 EVM에서는 두 종류의 챌린지 기간이 존재한다. 첫째는 블록에 대한 챌린지 기간인 $CP_{computation}$ 이고, 둘째는 exitRequest에 대한 연산 챌린지 기간인 $CP_{request}$이다. 앞서 언급했던 것처럼 제출된 블록과 제출된 exitRequest에는 각각 할당된 챌린지 기간을 어떠한 유효한 챌린지 없이 넘기는 경우에 한해서 확정된다. 각 챌린지 기간의 길이는 다음과 같다.

$$ CP_{computation} = 1 day $$

$$ CP_{request} = 1 day $$

블록을 대상으로 하는 챌린지 기간이 $CP_{computation}$ 인 이유는 $CP_{computation}$이 블록에 대한 모든 종류의 챌린지를 포괄할 수 있는 기간이기 때문이다. Computation Challenge를 제외한 챌린지가 단 한번의 트랜잭션을 전송하는 것으로 그 결과가 결정된다. 하지만 Computation Challenge는 자식체인의 블록 Gas Limit 1억을 기준으로 최악의 경우 최소 90번 이상의 트랜잭션을 필요로 한다. 따라서 $CP_{computation}$는 블록에 대한 다른 모든 종류의 챌린지를 실행하기에 충분한 시간인 것이다.

$CP_{computation}$이 1일인 이유는 90번의 트랜잭션이 전송되기 위해서는 적어도 2시간 ~ 2시간 30분 정도의 시간이 필요하고, 이는 네트워크 상황에 따라 변동이 있을 수 있기 때문이다. 하지만 향후 영지식 증명의 도입등을 통해 현재의 검증방식을 Verification game에서 보다 더 효율적인 방식으로 변경함으로써 챌린지 기간은 단축될 수 있을 것이다.

![](https://i.imgur.com/r1fK7Zv.png)
*<center>그림 6 : challenge period</center>*

<br>

# Solution to Data availability problem

Plasma EVM은 데이터 가용성 문제(Data availability problem, 이하 DA문제)에 대한 솔루션으로 User-Activated Fork(UAF)와 Dynamic cost mechanism을 제시한다. 이는 다음 문서에 바탕을 두고 있다. [A proposal for Data availability Solution of Plasma EVM](https://hackmd.io/kXWTlmyhSP2vVSIzsenU8w?view)

Plasma EVM에서 사용자들은 오퍼레이터의 Data withholding attack으로 DA문제가 발생할 경우 누구나 User Request Block(URB)를 제출하여 UAF를 생성함으로써 항상 유효한 상태만을 강제할 수 있다. 또한 DA문제와 관계없이 생성되는 무분별한 포크로 인한 혼란을 방지하기 위하여 포크에 대해 동적으로 비용을 부과하는 Cost mechanism을 도입하였다.


<br>

## User-Activated Fork
상기한대로 자식체인에서 DA문제를 인지한 사용자들은 URB를 제출하여 자식체인에서 안전하게 exit할 수 있다. 이 때 URB는 가장 마지막에 확정된 블록을 부모 블록으로 하여야 한다. 또한 URB는 오직 루트체인 컨트랙트에 제출된 ERU만을 포함하여야 한다. URB가 제출되면 오퍼레이터는 그 즉시Rebase 과정을 실행하여 제출된 URB를 조상블록으로 하는 NRB를 제출해야 한다.


![](https://i.imgur.com/OzCwqMX.png)
*<center>그림 7 : UAF diagram 1</center>*

현재 가장 마지막으로 확정된 블록은 Block#1이고, 루트체인 컨트랙트에는 NRB#5까지 제출되어 있는 상태이다. 여기서 NRB#3가 오퍼레이터에 의해 withheld 상태가 되어 DA문제가 발생했다고 가정해보자. 이 때 사용자들은 NRB#3가 확정되기 전까지 URB#2를 제출하여 포크를 발생시킴으로써 자식체인에서 안전하게 exit할 수 있다. 제출된 URB의 부모블록은 *그림 7* 과 같이 반드시 가장 마지막으로 확정된 블록인 Block#1이어야 한다. 이 과정에서 URB를 제출하는 사용자는 기존에 제출된 ORB들에 포함된 요청(request)들을 반영하여야 한다. 단, 기존의 ORB에 포함되지 않은 새롭게 제출된 ERO들은 URB에 포함되지 않아야 한다.

<br>

![](https://i.imgur.com/5mgithJ.png)
*<center>그림 8 : UAF diagram 2</center>*

URB가 제출되어 포크가 발생했을 경우 오퍼레이터는 체인을 계속 이어나가기 위해서는 반드시 재정렬(Rebase) 과정을 거쳐야 한다. *그림 8* 과 같이 오퍼레이터는 기존에 제출된 NRB#3와 NRB#5와 동일한 $transactionRoot$를 갖는 NRB#3' 와 NRB#4'를 제출해야 한다. 즉, 이전의 NRB에 포함되었던 거래들을 그대로 반영해야 한다. 또한 NRB#3' 와 NRB#4'은 URB#2 를 근거로 해야 한다.


### Mass URB
URB를 제출하기 위해서는 먼저 루트체인 컨트랙트에서 `prepareToSubmitURB`함수를 호출해야 한다. 하지만 DA문제가 발생할 경우 사용자들이 URB를 제출하기 위해 동시다발적으로 `prepareToSubmitURB`함수를 호출하는 경우가 발생할 수 있다. 이 경우 가장 먼저 `prepareToSubmitURB` 를 호출하는 트랜잭션을 전송한 사용자에게 URB를 제출할 권한이 주어진다.

제출될 URB는 루트체인 컨트랙트에 제출된 ERU를 포함하는 블록이기 때문에, 제출 전에 포함되어야 할 트랜잭션은 이미 결정되어 있다. 따라서 처리되는 exit request는 어느 URB를 선정하나 동일하기 때문에 URB에 대한 제출권한을 위와 같이 결정하게 되더라도 문제가 되지 않는다.

### Extension of chain head
URB가 제출된다고 해서 항상 포크가 발생하는 것은 아니다. 이전에 제출된 URB가 확정되기 전에 URB가 연이어 제출되었을 경우, URB의 제출은 포크가 아니라 chain head를 연장하는 행위로 간주된다. 따라서 이어지는 URB는 반드시 이전 URB의 자식블록이어야 한다. 물론 이 경우에도 오퍼레이터는 새롭게 연장된 chain head를 기준으로 재정렬 과정을 수행해야 한다.

물론 이전에 제출된 URB가 확정된 후에 URB가 연이어 제출되었을 경우, 이후에 제출된 URB는 chain head를 연장하는 행위가 아니라 포크를 발생시키는 행위로 간주된다. 따라서 제출되는 URB는 가장 마지막으로 확정된 블록을 부모블록으로 하여야 하며, 해당 URB의 제출이후 오퍼레이터는 재정렬 과정을 수행해야 한다.

<br>


![](https://i.imgur.com/KxfLGBd.png)
*<center>그림 9 : Extension of chain head</center>*

chain head를 연장하는 경우를 *그림 9* 를 통해 더 자세히 살펴보도록 하자. URB#2가 제출된 이후에 URB#3가 연달아 제출되었다. 이 경우 URB#3의 상태는 반드시 URB#2를 기반으로 연산되어야 한다. 따라서 URB가 새롭게 제출되었지만 이는 Block#1을 기준으로 하여 fork를 생성하는 것이 아니라, URB#2가 포함된 블록체인의 head를 연장하는 행위가 되는 것이다. 또한 오퍼레이터는 새롭게 연장된 URB#3를 기준으로 기존의 블록들을 모두 재정렬해야 한다.

<br>

### Invalid URB
![](https://i.imgur.com/ZsVmhKI.png)
*<center>그림 10 : Invalid URB</center>*

*그림 10* 은 제출된 URB의 stateRoot가 유효하지 않아 챌린지를 받게 되었을 경우에 대한 과정을 나타낸다. 현재 URB#2를 기반으로 또 다른 사용자가 제출한 URB#3, 오퍼레이터가 재정렬한 NRB#3'와 NRB#4'가 제출되어 있는 상황에서 URB#2에 대한 챌린지가 성공하였다. 이 경우 URB#2를 포함하여 URB#2를 조상블록으로 하는 모든 자손 블록들은 더 이상 유효하지 않게 된다. 단, 챌린지에 대한 처벌의 경우 URB#2의 제출자와 유효하지 않은 URB를 조상블록으로 하는 다른 URB제출자가 그 대상이 된다. 오퍼레이터는 유효하지 않은 URB를 기준으로 재정렬하더라도 처벌받지 않는다.

또한 URB에 대한 computation challenge를 제출하는 경우 챌린저는 반드시 대상이 되는 URB와 그 자손 URB들의 새로운 stateRoot, transactionRoot, receiptRoot를 함께 제출하여야 한다. 또한 재정렬 과정과 동일하게 새로운transactionRoot는 챌린지 대상이 되는 URB의 transactionRoot와 같아야 한다. 다시 말해 챌린저는 URB#2에 대해 챌린지를 하는 동시에 URB#2'와 URB#3'를 제출하게 되는 것이다. 역시 챌린지 이후 오퍼레이터는 URB#3'에 이어서 기존 NRB들을 다시 재정렬해야 한다.

### Data availability of URB
URB는 Data withholding attack의 대상이 될 수 없다. 그 이유는 다음과 같다. URB의 부모블록은 항상 확정된 블록이고 URB에 포함되는 트랜잭션들은 루트체인 컨트랙트에 공개되어 있기 때문에 누구나 URB의 올바른 stateRoot에 대해 알 수 있다.

<br>


## Cost mechanism against Infinite fork attack

### Infinite fork attack
User-activated fork 방식은 DA문제가 생길 경우 누구나 마지막으로 확정된 블록을 기준으로 URB를 제출하여 안전하게 exit이 가능하게 하였다. 하지만 여기에는 치명적인 약점이 존재하는데, 바로 재정렬 과정을 이용한 Infinite fork attack이 그것이다. 악의적인 사용자는 DA문제의 여부와 관계 없이 지속적으로 URB를 제출하여 블록의 Finalization을 무한히 막을 수 있다. 이는 URB가 제출되는 순간 기존의 확정되지 않은 NRB들은 재정렬 되어야 하고, 이는 해당 블록들에 새로운 챌린지 기간을 두는 것을 필요로 하기 때문이다.

<br>

### Dynamic cost mechanism

우리는 이러한 공격을 막기 위해 User-activated fork에 대해 동적 비용 메커니즘을 두었다. 그 설계 목표는 다음과 같다.

1. **URB의 제출이 공격의도일 확률에 가깝다면 많은 비용이 부과되어야 하고, DA문제상황에 대한 탈출의 용도일 확률에 가깝다면 낮은 비용이 부과되어야 한다.**
2. **재정렬을 발생시키는 URB의 제출 횟수는 가능한 최소화 되어야 한다.**

DA문제가 해결하기 어려운 이유는 자식 체인에서 실제로 DA문제가 발생했는지의 여부를 판단하는 것이 매우 어렵기 때문이다. 따라서 Plasma EVM에서는 첫번째 원칙처럼 DA문제의 발생여부에 대해 확률의 측면에서 접근하고자 한다.

또한 설령 다수의 사용자들이 URB를 제출하는 상황, 즉 DA문제가 발생했다고 판단하는 사용자들이 많아지더라도 실제로는 DA 문제가 없었을 가능성도 여전히 존재한다. 따라서 두번째 원칙에서 명시한 것처럼 항상 자식체인의 원활한 작동을 최대한 보장하기 위해 모든 상황에서 URB의 제출은 최소화 되어야 한다.

<br>

### DA probability
첫번째 원칙은 그 자체로서는 매우 타당한 규칙이지만, 결국 '확률'을 알아야만 적용이 가능하다는 한계가 있다. 이는 달리 말하면 확률을 얼마나 정확하게 계산할 수 있는지가 이 비용 모델 설계의 핵심이라고 할 수 있다. 그 확률의 산정에 가장 적합한 재료가 무엇일까? 우리는 앞서 DA문제에 대한 판단을 사용자 개인에게 일임한다고 하였다. 독립적인 개인이 DA문제에 대해 스스로 판단을 한다면, DA문제의 발생확률과 가장 상관관계가 높은 것은 바로 사용자 개인의 판단일 것이다.

즉, DA문제가 있다고 판단하는 사용자들의 수가 많아질수록 DA가 발생했을 가능성 또한 높아지는 것이다. 반대로 DA문제가 있다고 판단하는 사용자들의 수가 적다면 그만큼 자식체인에 문제가 없었을 가능성이 크다. 따라서 우리는 DA 문제에 대한 사용자들의 판단, 즉 ERU의 수를 통해 DA문제 발생확률을 산정하고 그에 맞게 URB와 ERU에 대한 비용을 조정하는 모델을 설계하고자 한다.

물론 ERU를 제출한 어카운트들이 모두 독립적인 개인일 것이라는 가정은 매우 순진하다. 그렇기 때문에 ERU를 제출하는 것에도 비용이 부과되어야 하는 것이다. 공격자가 많은 어카운트를 동원해 URB를 제출하기에 유리한 조건을 형성하기 위해서는 개별 ERU에 대한 비용도 모두 부담해야 할 것이기 때문에 이러한 공격방법은 그 효과를 잃을 것이다.

<br>

### Cost function

위에서 제시한 원칙을 최대한 만족하기 위해서는 어떠한 형태의 비용함수를 도출해야 하는지에 대해 논의해보자. 첫번째 원칙을 만족시키기 위해서는 URB에 포함되는 ERU의 수가 많아질수록 URB의 제출비용이 감소하고, ERU에 대한 비용 또한 감소해야 한다. 또한 두번째 원칙을 만족시키기 위해서 ERU의 수가 증가함에 따라 감소하는 비용의 폭이 점점 증가하는 형태가 바람직 할 것이다.

**$C_{URB}$ : URB를 제출하는데 소모되는 비용
$D_{URB}$ : URB를 제출하기 위해 필요한 보증금
$C_{ERU}$ : ERU를 통해 exit하는데 소모되는 비용
$N_{ERU}$ : URB에 포함된 ERU의 수**

우리는 다음과 같이 앞서 제시한 조건을 만족하는 비용함수를 간단하게 도출할 수 있다.


![](https://i.imgur.com/qxbifiz.png)
*<center>그림 11 : Cost function of submitting URB</center>*

URB를 제출하는데 소모되는 비용은 위와 같다. URB에 포함된 ERU의 수, 즉 $N_{ERU}$가 증가함에 따라 곡선의 기울기는 급격하게 감소한다. 여기서 $C_{URB}$가 일정 수준이하로 더 이상 감소하지 않는 이유는 앞서 짧게 언급했듯이 공격자가 다수의 어카운트를 생성하여 일종의 Sybil attack을 하는 경우도 고려해야 하기 때문이다.  

여기서 $N_{ERU}$이 증가함에 따라 $C_{URB}$ 선형적으로 감소하지 않고, 지수적으로 감소하는 이유는 두번째 원칙을 최대한 만족시키기 위함이다. 비용곡선이 직선의 형태라면 $N_{ERU}$가 증가함에 따라 감소하는 비용의 변화량이 일정하지만, 곡선의 형태라면 감소하는 비용의 변화량이 증가하는 형태가 된다. 따라서 URB를 제출하는 입장에서는 가능하면 하나라도 더 ERU를 URB에 포함시키는 것이 유리해진다. 이는 결과적으로 소수의 URB만으로 많은 ERU를 효율적으로 처리하는 것을 권장하게 되는 것이다.

![](https://i.imgur.com/8iuq26m.png)
*<center>그림 12 : Deposit amount for submitting URB</center>*

또한 URB를 제출하는 이는 위의 제출비용과 더불어 URB의 유효성을 보증하기 위해 $D_{URB}$을 예치하여야 한다. 이는 URB가 확정될 경우 제출자에게 모두 반환되며, 만약 해당 URB에 대한 챌린지가 성공했을 경우 $D_{URB}$는 모두 몰수된다.

단, URB에 대한 챌린지를 제출하는 챌린저는 오직 $D_{URB}$만 예치함으로써 새로운 URB를 제출할 수 있다.

![](https://i.imgur.com/k07vmZq.png)
*<center>그림 14 : Cost function of submitting ERU</center>*

ERU를 통해 Exit하는데 소모되는 비용은 *그림 14* 와 같다. $C_{ERU}$또한 $N_{ERU}$가 증가할수록 감소하는 형태를 가진다. 여기서도 $C_{URB}$의 비용곡선과 마찬가지로 Sybil attack 을 막기 위해 일정 수준 이하로 비용이 감소하는 것을 방지하고 있다. 사용자는 ERU를 제출하는 시기에 보증금의 형태로 $N_{ERU}$=0 일 때의 $C_{ERU}$를 예치하게 된다. 이 보증금에서 URB가 제출될 때의 $N_{ERU}$를 기준으로 공제된 수수료 $C_{ERU}$ 를 제외한 나머지를 ERU가 확정된 뒤에 돌려받게 된다.

<br>

### Total cost
앞서 비용함수를 설명할 때 잠깐 언급했던 Sybil attack에 대한 내용을 기억할 것이다. 공격자가 다수의 어카운트를 생성해서 ERU를 생성할 경우 $C_{URB}$, $C_{ERU}$가 너무 낮은 수준이 되면 공격비용이 지나치게 낮아질 위험이 있다. 이를 막기 위해 두가지 비용 모두 일정 수준 이하로 감소하는 것을 방지하였는데, 사실 이는 이러한 공격루트를 완벽하게 막는 형태는 아니다.

편의상 $N_{ERU}$가 모두 하나의 공격자에 의해 생성되었다고 가정할 경우, 결국 공격자가 지불해야 할 총 비용 $$C_{URB}^{total}$$ 은 $$C_{URB} + C_{ERU} * N_{ERU}$$ 가 될 것이다. 그렇다면 위에서 제시한 비용함수를 근거로 총 비용함수의 형태를 *그림 15* 와 같이 간략하게 나타낼 수 있다.

![](https://i.imgur.com/sYp2Ctf.png)
*<center>그림 15 : Total cost function</center>*



위의 총 비용함수에서 $N_{ERU}$ 가 증가함에 따라 $C_{URB}^{total}$ 또한 감소하는 형태를 갖는다. 이는 두가지 비용함수 $C_{URB}$ 와 $C_{ERU}$ 의 기울기가 모두 감소하는 형태이기 때문에 총 비용함수의 또한 기울기가 감소하는 형태가 되기 때문이다. 결국 공격자가 공격을 할 경우에는 $N_{ERU}$ 를 $C_{URB}^{total}$ 가 가장 낮아지는 지점까지 생성하게 될 것이다. 만약 그러한 경우에도 $C_{URB}^{total}$ 가 충분히 높은 상태라면 큰 문제가 되지 않을 수 있다.

하지만 $N_{ERU}$가 충분히 많은 상황에서도 $C_{URB}^{total}$ 을 높게 유지하는 것은 필연적으로 유저들의 exit에 대한 비용부담 또한 증가시키게 되는 문제가 있다. 이와 같은 문제를 해결하기 위해서 현재 연구가 더 필요한 상황이다.


<br>


## Issues

### Resolved issues

#### 1. 특정 사용자에게만 block withholding attack을 하는 경우
UAF 모델은 사용자들이 포크에 대한 비용을 분담하는 형태라고 볼 수 있다. 따라서 오퍼레이터가 특정 사용자만을 대상으로 Data withholding attack을 하는 경우 해당 사용자는 홀로 많은 비용을 부담하여 URB를 통해 exit을 해야 하는 상황이 발생할 수 있다. 하지만 이는 풀노드를 작동시키고 있는 다른 사용자가 P2P네트워크를 통해 해당 사용자에게 블록을 전달해주는 것으로 쉽게 해결이 가능하다. 정직한 사용자라면 본인이 언제든지 오퍼레이터의 공격대상이 될 수 있다. 때문에 다른 사용자들과 P2P연결을 유지하면서 블록을 공유할 유인이 충분하다고 볼 수 있다. 따라서 위와 같은 공격형태는 그 효과를 쉽게 잃을 가능성이 매우 높다.

<br>

#### 2. 특정 사용자에게만 영향이 있는 일부 상태(State) 데이터만 Withholding attack을 하는 경우
만약 특정 사용자에게만 영향을 미치는 일부분의 상태 데이터(eg. 사용자 A의 크립토키티 소유권)만 유효하지 않게 변경되고 동시에 Withheld 되는 경우, 해당 사용자를 제외한 다른 사용자는 UAF에 참여할 유인이 없어진다. 이처럼 오퍼레이터는 특정 사용자만을 대상으로 공격을 함으로써 해당 사용자가 URB 제출에 많은 비용을 부담할 수 밖에 없게끔 하여 공격의 효과를 극대화 할 수 있다. 만약 이와 같은 공격이 가능하다면 플라즈마 EVM은 DA문제에 매우 취약해질 것이다. 하지만 이와 같은 공격은 플라즈마 EVM에서 근본적으로 불가능하다.

그 이유는 특정 상태의 일부분만을 Withholding 하는 것이 불가능하기 때문이다. 오퍼레이터가 Withholding할 수 있는 대상은 곧 오퍼레이터가 다른 풀노드에게 블록을 전파할 때 전달해주어야 할 데이터를 의미한다. 즉, 전달할 필요가 없는 데이터는 Withholding의 대상이 될 수 없는 것이다. 이더리움에서는 블록을 전파할 때 크게 헤더와 트랜잭션들을 전달한다. 헤더는 stateRoot와 같은 Root값들이 포함되어 있고, 트랜잭션은 말 그대로 트랜잭션이다.

이렇게 블록을 전파받은 노드들은 해당 블록에 대해서 여러가지 검증 과정을 거치게 된다. 그 중 핵심은 그 중 핵심은 전달받은 트랜잭션들을 하나하나 실행하면서 도출된 stateRoot와 전달받은 헤더의 stateRoot와 일치하는지 확인하는 작업이다. 또한 이 과정에서 자연스럽게 여러 어카운트의 상태가 변경된다.

중요한것은 여기서 상태 데이터베이스 자체를 전파하는게 아니라는 것이다. 상태의 일부분만 withholding 하는 것이 불가능한 이유는, 해당 정보가 애초에 전파의 대상이 아니기 때문이다. 만약 이렇게 오퍼레이터로부터 전파받은 블록이 데이터의 일부분만 Withheld된 상태라면 해당 블록을 검증하는 과정에서 누구나 DA문제를 인지할 수 있을 것이다. 하지만 여기서 사용자들이 확인할 수 있는 정보는 오직 전달되어야 할 블록 데이터의 일부분이 누락되었고, 루트체인에 유효하지 않은 stateRoot가 제출되었다는 것 뿐이다.

그 이유는 어떤 머클 트리의 루트가 변경되었을 때 해당 트리에 어떤 데이터가 변경되었는지 알아내는 것이 불가능한 것처럼, stateRoot만 변경된 것을 보고 해당 트리에 어떤 특정 데이터가 변경되었는지 알아내는것이 불가능하기 때문이다. 다시 말해 블록의 일부라도 Withholding이 된 순간, 본인의 이더가 공격받았는지 혹은 다른 사용자의 크립토키티가 공격받았는지 확인할 수 있는 길이 없다는 것이다. 알 수 있는 사실은 오로지 오퍼레이터가 유효하지 않은 상태변환을 하였고 그에 대한 데이터를 모두 제대로 전파하지 않았다는 사실 뿐이다. 따라서 부분적으로 데이터를 Withholding하는 경우에 대해서 사용자들은 항상 모두 UAF를 통해 exit할 유인이 생기기 때문에 특정 사용자에게만 영향을 끼치는 Data withholding attack을 할 수 없다.

<br>

### Unsolved issues

#### 1.과거의 트랜잭션을 취소하려는 목적으로 URB를 제출하는 경우
User activated fork 모델이 갖는 단점은 포크로 인해 유효한 상태도 확정이 되기 전에는 얼마든지 변경이 가능하다는 것이다. 이것이 왜 문제가 되는지 다음 예시를 통해 간단하게 살펴보도록 하자.

탈중앙거래소(Decentralized Exchange; DEX)가 운영되고 있는 플라즈마 체인이 있다고 해보자. DEX에는 Token A가 활발하게 거래되고 있다. 다수의 사용자들은 한시간 전 A를 1이더에 다량 구매하는 트랜잭션 T를 전송하였다. 하지만 암호화폐 시장의 붕괴로 A의 가격이 0.1이더로 폭락하였다. 또한 현재 모든 T들이 포함된 블록은 확정되지 않은 상태이다. 이 경우 T를 전송했던 사용자들은 UAF에 대한 비용이 T가 확정되었을 경우 입을 손실보다 낮다면 항상 UAF를 통해 자신의 이더를 exit하여 새롭게 재정렬된 블록에서 T의 실행결과를 변경시킬 유인이 있을 것이다. 요컨대 UAF에 대한 비용보다 과거의 트랜잭션들을 취소했을 때의 이득이 더 크다면 DA문제가 없더라도 항상 사용자들은 UAF를 생성할 유인이 있음을 의미한다.

# References
- [Joseph Poon and Vitalik Buterin, Plasma: Scalable Autonomous Smart Contracts](https://plasma.io/)
- [Minimal Viable Plasma](https://ethresear.ch/t/minimal-viable-plasma/426)
- [Plasma Cash](https://ethresear.ch/t/plasma-cash-plasma-with-much-less-per-user-data-checking/1298)
- [Plasma XT](https://ethresear.ch/t/plasma-xt-plasma-cash-with-much-less-per-user-data-checking/)
- [PARSEC Labs, PLASMA - FROM MVP TO GENERAL COMPUTATION](https://parseclabs.org/files/plasma-computation.pdf)
- [Why Smart Contracts are NOT feasible on Plasma](https://ethresear.ch/t/why-smart-contracts-are-not-feasible-on-plasma/2598)
- [TrueBit, A scalable verification solution for blockchains ](https://people.cs.uchicago.edu/~teutsch/papers/truebit.pdf)

# Related Links
* [플라즈마 EVM 2.1 : 탈중앙화된 상태를 강제하는 튜링 완전한 플라즈마](https://hackmd.io/s/B1r45tQC7?fbclid=IwAR3LzguUu0NzLVAEgxYZtF3PND9lYi294slWCjVgxujmEYF_7pAVf7AdY1c)
