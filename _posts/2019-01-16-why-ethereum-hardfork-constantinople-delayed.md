---
layout: post
title:  왜 이더리움 1월16일 콘스탄티노플 하드포크는 연기되었나?
date:   2019-1-16 00:00:00 +09:00
author: "신건우(Thomas)"
categories: ethereum core
tag: [ethereum, SmartContract, solidity, general, core]
youtubeId : cdTSxsajibo
slideWebId : 2PACX-1vRgrcknAw5n2_jHHgjDiQASHGGVjbQkxljgYs_NXYY1-UQzHX4FXJ0ZirzMulOYMKhdSfjjftPpL8_n
---

* content
{:toc}

## EIP 1283으로 인한 하드포크 연기

![](https://cdn-images-1.medium.com/max/800/1*A9ImG72ocext5SptLC9-yg.jpeg)

예정된 콘스탄티노플 하드 포크(*at block 7,080,000 on January 16, 2019*)가 특정 SSTORE 연산에 대해 적은
gas비를 소모하게 하는 스펙([EIP
1283](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1283.md)) 때문에
연기되었습니다. 이는 스마트 컨트랙트 내에서 `address.transfer(…)` 또는 `address.send(…)` 함수를 호출할 때
reentrancy 공격이 가능하다는 잠재적 위험성이 있습니다. 콘스탄티노플 하드포크 이전에 이 함수들은 [reentrancy에
안전](https://ethereum.stackexchange.com/a/38642)했지만 콘스탄티노플 하드포크 이후에는 더 이상
reentrancy 공격에 안전하지 않습니다. 이에 대해 기술적으로 알아보고자 합니다.

아래의 내용은 영상으로도 확인이 가능합니다.

* youtube:
[https://www.youtube.com/watch?v=cdTSxsajibo](https://www.youtube.com/watch?v=cdTSxsajibo)
* slide:
[https://docs.google.com/presentation/d/1eNf3qpkMxtWg6Tjck-mdRsBjq7JIVzdGBJJgUMv0B5w/edit#slide=id.p](https://docs.google.com/presentation/d/1eNf3qpkMxtWg6Tjck-mdRsBjq7JIVzdGBJJgUMv0B5w/edit#slide=id.p)

### EIP 1283

EIP 1283에 대해서 설명드리기 이전에 storage에 대해 간단히 설명드리겠습니다.

#### storage

**storage**는 key / value 맵핑 구조입니다. key는 storage slot을 말하며, value는 이 storage
slot에 들어가는 값입니다. 이 storage에 값을 저장시키기 위한 gas 소모량은 20,000입니다. 매우 비싸죠. 그러나 값을 새로 쓰는
경우에만 20,000 gas가 소모되고, 값을 업데이트할 때는 5,000 gas를 소모합니다. 그리고 만약 storage slot의 값을 0으로
set할 때는 15,000 gas를 refund 해줍니다. 왜냐하면 이는 storage를 비우는 행위이기 때문입니다.

#### original value / current value / new value

EIP 1283에서는 새로운 용어를 정의합니다.

* Storage slot’s **original value**: This is the value of the storage if a
reversion happens on the current transaction.
* Storage slot’s **current value**: This is the value of the storage before SSTORE
operation happens.
* Storage slot’s **new value**: This is the value of the storage after SSTORE
operation happens.

**original value**는 말 그대로 트랜잭션이 발생하기 직전에 storage slot이 가지고 있던 값입니다. 그리고
**current value**는 SSTORE opcode를 만났을 때 storage slot이 가지고 있는 값입니다. **new
value**는 SSTORE opcode를 실행한 이후 storage slot이 가지고 있는 값입니다.

여기서 original value와 current value가 같은거 아니야?라고 생각하실 수도 있지만 다릅니다. 트랜잭션이 실행되면서 특정
storage slot에 SSTORE가 여러번 실행될 수 있습니다.

예를 들어 하나의 트랜잭션 내에서 다음과 같은 opcode가 실행이 된다고 가정을 해보겠습니다. `SSTORE(1, 100), …,
SSTORE(1, 200), …, SSTORE(1, 300)`

두 번째 SSTORE opcode에서 current value는 100이고 new value는 200입니다. 하지만 opcode만을 보고서는
original value가 어떤 값인지 알 수 없습니다. 왜냐하면 트랜잭션이 실행되기 직전에 가지고 있던 storage slot의 값이
original value이기 때문입니다.

세 번째 SSTORE opcode에서 current value는 200이고 new value는 300입니다.

#### SSTORE gas 소모 변경 내역

* current value와 new value가 같으면 **200 gas**를 소모합니다.<br> e.g. <br> *original value:
0, current value: 100, new value: 100*<br> *original value: 100, current value:
100, new value: 100*
* current value와 new value가 다르고, original value와 current value가 같을 때 20,000 gas 또는
15,000 gas를 소모합니다(만약 new value가 0이면 15,000 gas를 refund합니다).<br> e.g. <br>
*original value: 0, current value: 0, new value: 20***–20,000 gas 소모**<br>
*original value: 10, current value: 10, new value: 20***–5,000 gas 소모**<br>
*original value: 10, current value: 10, new value: 0***–15,000 gas를 refund**
* current value와 new value가 다르고, original value와 current value가 다른 경우입니다.<br>
e.g.<br> *original value: 100, current value: 0, new value: 300* — **refund 해준
15,000 gas를 다시 차감**<br> *original value: 100, current value: 200, new value: 0 —
***15,000 gas를 refund**<br> *original value: 0, current value: 100, new value: 0
— ***19,800 gas를 refund.**<br> *original value: 100, current value: 200, new
value: 100 — ***4,800 gas를 refund**

경우의 수가 많아 단번에 이해하기는 힘들지만 차근차근 보시면 쉽게 파악이 가능합니다. 여기서 주목할 점은 original value와
current value가 다른 경우 특정 storage slot에 SSTORE가 이미 한 번 이상 발생했다는 것을 알 수 있습니다.

`original value: 100, current value: 0, new value: 300`에서 original value가 100이고
current value가 0이라는 의미는 이미 이전에 SSTORE(stroage slot, 0)를 실행했다는 의미입니다. 즉 storage
slot을 clear했기 때문에 15,000 gas를 refund해줬습니다. 하지만 다시 new value가 300인 SSTORE를 실행하기
때문에 이전 SSTORE 연산에서 15,000 gas를 refund했던 것을 다시 차감합니다.

#### new SSTORE scheme

위의 내용에 따라 SSTORE에 대한 scheme을 새로 정의합니다.

* **No-op**: EVM의 실행을 필요로 하지 않습니다.<br> > current value == new value
* **Fresh**: Storage slot의 변화가 없었거나 current value가 original value로 다시 reset된 상태를
말합니다.<br> > original value == current value && current value != new value
* **Dirty**: SSTORE 연산 이전에 한 번 이상 SSTORE 연산으로 인해 storage slot의 값이 바뀐 상태를 말합니다.<br>
> original value != current value && current value != new value

이를 아래의 표와 같이 나타낼 수 있습니다.

original value 값이 무엇이든 상관없이 current value와 new value가 같을 때 해당 storage slot은
No-op입니다. 그리고 original value와 current value가 같고 current value와 new value가 같을 때
해당 storage slot은 Fresh하다고 합니다. 반면 original value와 current value가 다르고 current
value와 new value가 다를 땐 이 storage slot을 Dirty하다고 합니다.

### Reentrancy Smart Contract

EIP 1283을 적용한 콘스탄티노플 버전에서 reentrancy 문제가 어떻게 발생할 수 있는지 컨트랙트 코드를 예시로 설명드리겠습니다.

하나는 PaymentSharer 컨트랙트고 하나는 reentrancy 공격을 하는 Attacker 컨트랙트입니다.

PaymentSharer 컨트랙트는 다음의 기능을 제공합니다.

* deposit(uint): ether를 deposit합니다.
* updateSplit(uint,uint): splits의 값을 update합니다.
* splitFunds(uint): deposit한 ether를 splits의 값(비율)에 따라 first 계정과 second 계정에게 분배합니다.

다음은 Attacker 컨트랙트입니다. Attacker 컨트랙트는 attack 함수를 호출해서 공격합니다. attack 함수는
PaymentSharer 컨트랙트의 주소를 가지고 온 뒤 PaymentSharer 컨트랙트의 updateSplit 함수와 splitFunds
함수를 호출합니다.

참고로 Attacker 컨트랙트의 fallback 함수에 있는 assembly 구문에 있는 코드는 updateSplit(0, 0) 호출합니다.
([cb18fb6](https://www.4byte.directory/signatures/?bytes4_signature=c3b18fb6):
function selector로 contract의 함수를 식별합니다.)

여기서 공격시나리오는 다음과 같습니다.

> 공격자는 deposit을 N만큼 하고, 2N만큼 가져간다.

1.  공격자는 attack 함수 내에서 updateSplit(0, 100) 함수를 호출해서 splits을 set합니다. **이렇게 하는 이유는 나중에
SSTORE의 gas비를 저렴하게 만들기 위해서입니다.** 이 때 first address는 Attacker contract 주소입니다.
second address는 EOA(Externally Owned Account)이던 CA(Contract Account)이던
상관없습니다.<br> > **splits’s storage slot** — *original value: 0, current value: 0,
new value: 100*
1.  이후 attack 함수 내에서 splitFunds(0) 함수를 호출해서 first address가 모든 fund를 받게 합니다. 이 때 a는
contract address(Attacker contract)이기 때문에 fallback 함수가 호출 됩니다.
1.  Attacker 컨트랙트의 fallback 함수에서 공격자는 splits을 다시 업데이트합니다(updateSplits(0, 0) 함수 호출).
reentrancy가 발생한 지점입니다.<br> > **splits’s storage slot** — *original value: 0,
current value: 100, new value: 0*
1.  이후 b.transfer 함수를 호출해서 second address에게 fund를 100% 지급하게 됩니다.

콘스탄티노플 버전 이전에는 다음과 같은 공격이 불가능합니다. 왜냐하면 3번 과정에서 Attacker 컨트랙트의 fallback 함수가
updateSplit 함수를 호출할 때 SSTORE가 발생하기 때문입니다. 콘스탄티노플 버전 이전에서 SSTORE는 최소 5,000 gas가
소모됩니다. transfer 함수로 호출된 fallback 함수 내에서는 2,300 gas까지 밖에 사용할 수 없기 때문에 revert가
발생합니다. 즉 transfer 함수로 인해 reentrancy가 방지되는 것이죠.

하지만 콘스탄티노플 버전에서는 3번 과정에서 gas를 200만큼만 사용하기 때문에 콘스탄티노플 이전 버전처럼 revert가 발생하지 않게
됩니다. 즉 transfer 함수는 위와 같은 경우엔 reentrancy 공격을 막아줄 수 없습니다.

아래의 그림을 통해 해당 컨트랙트의 로직을 좀더 쉽게 확인해보실 수 있습니다. (출처: status)

### 결론 및 대안

Reentrancy는 DAO 해킹 사건에 사용된 공격 패턴입니다. 그렇기 때문에 이러한 문제점을 미리 발견하지 못했더라면 제 2의 DAO 해킹
사건이 일어날 가능성도 있었겠죠. 콘스탄티노플 하드포크 이전에 문제를 발견하고 하드포크를 연기한 것은 어찌보면 당연한 것으로 보입니다.

아마 이더리움 재단 측에서는 EIP 1283를 제외하고 하드포크를 진행하거나 EIP 1283에 대한 스펙 변경이 있을 것으로 보입니다. 예를
들면 Dirty한 storage slot의 사용료를 200에서 2300이상으로 높일 수도 있죠. 다만 이런 경우도 다양한 결과에 대한 테스트가
필요할 것으로 보입니다. 긴 글 읽어주셔서 감사합니다.

### [Onther-Tech](https://medium.com/onther-tech?source=footer_card)

We do Ethereum blockchain focused research and develop. Onther means that we are
on Ethereum.

{% include youtubePlayer.html id=page.youtubeId %}
{% include slidePlayer.html id=page.slideWebId %}

# reference
- [(미디엄)왜 콘스탄티노플 하드포크는 연기되었나?](https://medium.com/onther-tech/%EC%99%9C-%EC%BD%98%EC%8A%A4%ED%83%84%ED%8B%B0%EB%85%B8%ED%94%8C-%ED%95%98%EB%93%9C%ED%8F%AC%ED%81%AC%EB%8A%94-%EC%97%B0%EA%B8%B0%EB%90%98%EC%97%88%EB%82%98-685e54d652a5)

- [https://blog.ethereum.org/2019/01/15/security-alert-ethereum-constantinople-postponement/](https://blog.ethereum.org/2019/01/15/security-alert-ethereum-constantinople-postponement/)
- [https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1283.md](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-1283.md)
- [https://medium.com/chainsecurity/constantinople-enables-new-reentrancy-attack-ace4088297d9](https://medium.com/chainsecurity/constantinople-enables-new-reentrancy-attack-ace4088297d9)
- [Research](https://medium.com/tag/research?source=post)
