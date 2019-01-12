---
layout: post
title:  내 토큰을 마음대로 빼갈 수 있는 컨트렉트를 만들 수 있을까?
date:   2018-06-01 00:00:00 +09:00
author: "정순형(Kevin)"
categories: ethereum SmartContract solidity
tag: [ethereum, SmartContract, solidity]
youtubeId :
slideWebId :
---
* content
{:toc}
{% include youtubePlayer.html id=page.youtubeId %}
{% include slidePlayer.html id=page.slideWebId %}

![](https://cdn-images-1.medium.com/max/800/0*tS0TNAFoAgdkGP5O.png)

안녕하세요. 철학자입니다.

오늘 커뮤니티에 “**이오스 에어드롭 참가 과정에서 내가 보유한 토큰을 모두 잃을 수 있다”** 는 글이 올라왔습니다.

불안하신지 물어보시는 분들이 많아서 이에 대한 분석이 필요한 것 같아서 글을 남깁니다.

> 결론부터 말씀드리면, 현재 구현(ERC20)에서 토큰을 빼가는 것은 **불가능** 합니다.

### 이오스 컨트랙트의 transfer 구현

[이오스 토큰
컨트랙트](https://etherscan.io/address/0x86fa049857e0209aa7d9e616f7eb3b3b78ecfdb0#code)의
transfer는 다음으로 구현되어 있습니다.(일반적인 다른 토큰 구현도 거의 같습니다.)

```solidity
  function transfer(address dst, uint wad) returns (bool){
    assert(_balances[msg.sender] >= wad);  
    _balances[msg.sender] = sub(_balances[msg.sender], wad);
    _balances[dst] = add(_balances[dst]wad;
    Transfer(msg.sender, dst, wad);
    return true;  
  }
```

msg.sender를 통해서 잔액 확인을 하고 msg.sender의 잔액을 옮깁니다.

결국 에어드롭 컨트랙트를 호출하는 과정에서 **i)msg.sender가 바뀌느냐, 안바뀌느냐** 그 과정에서 **ii)잔액 이라는 상태 변수를
바꿀 수 있는 권한이 있느냐** 의 문제로 귀결됩니다.

### 에어드롭 컨트랙트가 호출되는 과정

에어드롭 컨트랙이 호출되는 과정은

어카운트 A가 에어드롭 컨트렉 B를 콜하고, 컨트렉 B는 이오스 컨트렉C를 콜하는 구조이죠.

A(EA) → B(에어드롭) → C(이오스)

에어드롭 컨트렉은 다양하게 구현될 수 있는데, 콜(call)과 델리게잇콜(delegateCall) ,스태틱 콜(staticCall) 세가지
방법으로 구현될 수 있습니다.<br> 세가지 경우의 수를 모두 살펴보죠.

### (1) : 에어드롭 컨트랙트(B)가 이오스 컨트렉트C 를 콜(call)하는 경우

이 경우에 콜 과정에서 msg.sender가 바뀝니다.

![](https://cdn-images-1.medium.com/max/800/1*M6QP78kLVDz9PeRU3rXwsA.png)
*<center>B가 C를 Call할 경우</center>*

따라서 B는 이오스 컨트랙트 C에 있는 A의 잔액을 바꿀 수 없습니다.

### (2) :에어드롭 컨트랙트(B)가 이오스 컨트렉트C를 델리게잇콜(delegateCall)하는 경우

이 경우에 콜 과정에서 msg.sender는 바뀌지 않습니다.

![](https://cdn-images-1.medium.com/max/800/1*9X5HfjrPVNUo3QC445_GYQ.png)
*<center>B가 A를 delegateCall할 경우</center>*

살짝 무서울 수도 있지만, 다행이도 **delegateCall은 caller의 상태만 바꿀 수 있고, callee의 상태를 바꿀 수
없습니다.(**`CALLER` behaves exactly in the callee's environment as it does in the
caller's
environment.**)[**[https://github.com/ethereum/EIPs/issues/23](https://github.com/ethereum/EIPs/issues/23)**]**

즉 msg.sender는 유지했지만, B는 C의 제어권을 잃게 되는 것이죠.<br> 따라서 B는 C에 기록된 A의 잔액을 바꿀 수 없습니다.

### (2) :에어드롭 컨트랙트(B)가 이오스 컨트렉트C를 스테틱 콜(staticCall)하는 경우

스태틱콜(static call)의 경우는 Caller가 Calee를 호출하는 과정에서 msg.sender를 유지함과 동시에 제어권도 넘기지
않습니다.[[https://github.com/ethereum/EIPs/blob/master/EIPS/eip-214.md](https://github.com/ethereum/EIPs/blob/master/EIPS/eip-214.md)]

![](https://cdn-images-1.medium.com/max/800/1*7pa31XhIlQvDnEvAczr72A.png)
*<center>B가 C를 staticCall할 경우</center>*

msg.sender도 바뀌지 않는데다가 제어권까지 넘기지 않으니 B는 절대로 C에 기록된 A의 잔액을 바꿀 수 없습니다.

### 결론

단순 0이더를 보내는 트랜잭션을 에어드롭 컨트렉트에 보낸다고 해도 **제3자 컨트랙트인 에어드롭 컨트랙트가 토큰을 맘대로 빼갈 수 있는 방법은
없습니다.**

*****

이오스 뿐만 아니라 이와 비슷한 형태의 다른 ERC20또한 마찬가지이며, 현재(2018–6–4) 이오스의 토큰 전송은 동결(freeze)되어있는
상태이므로 위의 경우와 상관없이 토큰을 빼가는 것은 불가능합니다.


# Related Links
- [내 토큰을 마음대로 빼갈 수 있는 컨트렉트를 만들 수 있을까?](https://medium.com/onther-tech/%EB%82%B4-%ED%86%A0%ED%81%B0%EC%9D%84-%EB%A7%98%EB%8C%80%EB%A1%9C-%EB%B9%BC%EA%B0%88-%EC%88%98-%EC%9E%88%EB%8A%94-%EC%BB%A8%ED%8A%B8%EB%A0%89%ED%8A%B8%EB%A5%BC-%EB%A7%8C%EB%93%A4-%EC%88%98-%EC%9E%88%EC%9D%84%EA%B9%8C-3f607d3c5ceb)
