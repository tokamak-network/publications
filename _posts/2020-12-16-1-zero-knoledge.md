---
layout: post
title:    ZKP 기술보고서 - 1. 영지식 증명이란
date:   2020-12-16 00:00:00 +09:00
author: "장재혁"
categories: zkp
tag: [zkp]
youtubeId :
slideWebId :
---


> 작성자: 장재혁, [GIST 블록체인 인터넷 경제 연구센터 (센터장 이흥노)](https://infonet.gist.ac.kr/?page_id=6711)
>
> This work was created through a joint research with Onther Co., LTD., and supported by a grant-in-aid of Institute of Information & Communications Technology Planning & Evaluation (IITP), Republic of Korea.
>
> 이 글은 정보통신기획평가원(IITP)의 지원을 받아 (주)온더와의 공동연구를 통해 만들어진 결과물이다.

영지식 증명(zero-knowledge proof)은 증명 프로토콜로써 증명자가 검증자에게 어떤 명제(statement)가 참임을 납득시키는 프로토콜이다. 이때, 영지식 증명 프로토콜은 검증자에게 그 명제가 참 혹은 거짓이라는 사실 외의 다른 정보는 일절 전달하지 않는 영지식(zero-knowledge) 특성을 가져야 한다.

예를 들어 출입구가 단 하나뿐인 원형 동굴을 상상해보자. 출입구 반대편의 한 지점에는 비밀번호를 필요로 하는 잠금 장치가 있는 문이 있다. 즉, 입구에서 출발하여 원형 동굴을 한 바퀴 일주 한 후 출구로 나오기 위해서는 잠금 장치의 비밀번호를 알아야 한다. 이 때, "나는 잠금 장치의 비밀번호를 알고 있다"라는 명제를 타인에게 증명해야하는 상황을 가정하자. 영지식 증명은 잠금 장치의 비밀번호를 알려주지 않으면서 위의 명제가 참임을 납득시키는 프로토콜이다. 이러한 문제를 알리바바 동굴 (Alibaba's cave) 문제라 한다 (그림 1).



![What is zkSNARKs: Funny Moon Math](/images/article_1/media/image1.png)
{: width="100%" height="100%"}
그림 1 알리바바 동굴 문제

영지식성을 배제하면, 위의 예에서 검증자가 명제가 참 혹은 거짓임을 확인 할 수 있는 가장 쉬운 방법은 증명자로부터 비밀번호를 전달받은 후 잠금 장치를 열어보는 것이다. 반면 영지식 증명인 경우, 검증자가 비밀번호를 알아서는 안되기 때문에 증명이 어려워진다. 영지식 증명은 중요한 정보의 공개가 꺼려지는 상황에서 그 정보와 관련된 어떤 인증을 해야 할 때 사용 될 수 있다. 예를 들어 우리나라에서 자주 사용되는 공인인증서 검사 프로그램에 사용 될 수 있다. 영지식 증명이 적용된다면, 내 인증서의 비밀번호를 입력하지 않고도 내가 나임을 인증 할 수 있다. 공인인증 프로그램은 사용자가 설정한 비밀번호에 보안이 크게 의존되어 있기 때문에, 이러한 응용 예는 실용적일 수 있다.

정리하면, 공개된 정보와 비공개 정보로 구성된 어떤 명제를 $$S$$라 할 때, 어떤 증명 프로토콜이 영지식 증명 프로토콜이 되기 위한 필요충분조건 세 가지는

1\. 완전성(completeness): 만약 $$S$$가 참이면, 진실한 증명자가
검증자에게 $$S$$가 참임을 납득시킬 수 있어야 한다.

2\. 건실성(soundness): 만약 $$S$$가 참이 아니면, 거짓 증명자가
검증자에게 $$S$$가 참임을 납득시킬 수 없어야 한다.

3\. 영지식성(zero-knowledge): 검증자는 $$S$$가 참 혹은 거짓이라는 사실
외에, 어떤 비공개 정보도 알 수 없어야 한다. 즉, 증명자가 $$S$$의
참/거짓에 결정적 관련이 있는 정보들 중 일부를 선택적으로 공개하지 않을
수 있어야 한다.

영지식증명 프로토콜은 증명과정과 검증과정의 두 과정으로 구분 될 수 있다. 증명과정은 증명자가 명제 $$S$$가 참임을 스스로 검증 한 후, 그 증거를 생성하는 과정이다. 검증과정은 검증자가 증명자로부터 증거를 받아 증거에 문제가 없는지를 확인하는 과정이다. 만약 증거에 문제가 없다면, 증명자가 명제 $$S$$의 자가검증을 잘 수행했다는 사실이 수학적으로 보장되어 있어야 한다.

영지식 증명의 활용 예: 온라인 투표 시스템
=========================================

온라인 투표 시스템은 공동체 구성원들이 직접적인 대면 없이 의사를 전달하고 결정할 수 있도록 해주는 시스템이다. 이는 대표적인 대면 (오프라인) 투표시스템인 선거와 대조적이다. 선거와 같은 기존 대면 투표시스템은 완벽하지는 않지만 표 1의 특성들이 잘 지켜질 수 있도록 체계화 되어있다. 그러나 대면 투표시스템은 시간과 공간의 제약을 크게 받으며, 이로인해 발생하는 인적, 물적 자원 소요 또한 상당하다. 이에 반해 온라인 투표시스템은 경제적이며 언제, 어디서든 의사표현을 할 수 있다는 장점이 있지만, 표 1의 특성을 지키기 어렵다는 단점이 있었다. 일반적인 투표 시스템이 지녀야 할 투표의 특성은 아래 표 1과 같이 나열 할 수 있다.

| 정확성 | 모든 정당한 유효투표는 투표결과에 정확히 집계됨   |
|--------|---------------------------------------------------|
| 확인성 | 투표결과 위조방지를 위한 투표결과 검증수단이 필요 |
| 완전성 | 부정 투표자에 의한 방해 차단, 부정투표는 미 집계  |
| 단일성 | 투표권이 없는 유권자의 투표참여 불가              |
| 합법성 | 정당한 투표자는 오직 1회만 참여 가능              |
| 기밀성 | 투표자와 투표결과의 비밀관계 보장                 |
| 공정성 | 투표 중의 집계결과가 남은 투표에 영향을 주지 않음 |

표 1 투표 시스템의 특성

온라인 투표 시스템에서는 모든 과정에서의 산출물이 데이터로써 저장된다. 만약 데이터가 중앙 서버에 저장된다면, 데이터의 위변조가 가능하기 때문에 기술의 도움을 받을 필요가 있다. 이때 도움이 될 수 있는 기술 중 하나가 블록체인이다. 구체적으로, 투표의 특성 중 정확성과 확인성, 완전성, 합법성은 블록체인 기술로 해결 될 수 있다. 블록체인의 의의가 "위변조가 불가능하고 누구나 검증가능한 분산 서버"를 만드는 것이기 때문이다.

한편 온라인 투표의 단일성과 합법성은 digital identity (DID)[^1]와 public key infrastructure (PKI)[^2] 기술의 도움을 받아 만족시킬수 있다. 일반적인 대면방식의 선거에서는 선거위원이 투표자의 신원을 확인한 후 투표권 (도장이 찍힌 투표용지)을 나눠준다. 만약 온라인 투표라면, 신원확인에 필요한 신분증을 공인인증서와 같은 DID로 대체 할 수 있다. 문제는 온라인 투표권인데, 만약 투표권이 항상 변하지 않는 어떤 디지털 데이터라면, 복제되어 악용될 수 있다. 그러므로 투표권은 투표자의 DID에 따라 변하는 데이터여야 한다. 이렇게 DID를 기반으로 투표권을 생성할 수 있는 기술의 예가 PKI이다. 중앙 선거관리 위원회가 PKI를 통해 사용자의 DID를 넘겨받은 후 고유의 투표권 데이터를 다시 사용자에게 배포 할 수 있다. 이후 투표 결과를 검표할때는 투표권 데이터가 중앙 선거관리 위원회에 의해 배포된 투표권이 맞는지 확인할 수 있다. 즉, DID와 투표권 데이터간의 1대1 대응이 되도록 하는것이다. 만약 DID와 사용자 개인간의 완벽한 1대1 대응이 보장된다면, 단일성과 합법성이 만족 될 수 있다. 그러나 검표과정에서 어떤 DID가 어떤 의사결정을 하였는지가 중앙 선거관리 위원회에 공개되기 때문에, 기밀성을 만족시키지는 못한다.

영지식 증명은 온라인 투표 시스템이 기밀성을 만족하도록 해주는 도구가 될 수 있다. 예를 들어어떤 투표자가 있고, 이 투표자의 DID를 $$x$$, 그리고 $$x$$에 의해 생성된 투표권 데이터를 $$s\left( x \right)$$라 하자. 그리고 투표권 데이터를 확인 (verification)하는 함수를 $$f$$라 하자. 즉, $$s$$가 $$x$$로부터 생성된게 맞다면 $$f\left( x,s \right)=1$$이며, 그렇지 않다면 $$f\left( x,s \right)=0$$이다. 이제 투표자가 증명해야 할 명제 $$S$$는 "$$f\left( x,s \right)=1$$"이며, 여기서 공개되지 않아야 할 비공개정보는 $$x$$이다. 영지식 증명 프로토콜에 의해 투표자는 명제 $$S$$를 스스로 자가검증한 후 그 증거파일 $$\pi $$를 생성하여 투표결과와 함께 투표함에 기록한다. 이후 검표과정에서는 $$\pi $$에 문제가 없는지를 확인한다. 만약 증거파일 $$\pi $$에 문제가 없다면, 영지식 증명의 정의에 의해 해당 투표는 정당한 투표권자에 의한 투표임을 수학적으로 보장받게 되며 (단일성 만족), DID를 유출하지 않았기 때문에 투표자와 투표결과간의 비밀관계도 보장받게 된다 (기밀성 만족).

![](/images/article_1/media/image2.png)
<!-- {width="6.268055555555556in" height="4.441882108486439in"} -->
그림 2 한양대학교와 국민대학교 산학협력단이 출원한 영지식 증명이 적용된 블록체인기반 온라인 투표시스템

그림 2는 한양대학교와 국민대학교가 2018년 공개출원한 특허 "비밀 선거가 보장된 블록 체인 기반의 전자 투표를 수행하는 단말 장치 및 서버와, 전자 투표 방법"의 대표도해이다. 그림 2의 온라인 투표시스템에서, 투표자(유권자)의 DID는 $$pk\_ID$$이다. 선거관리 위원회가 투표자의 DID를 확인 한 후 투표자에게 투표권 데이터인 $$\sigma \_pk\_ID$$와 $$vkey\_M$$을 준다. 투표권 데이터는 투표자의 DID에 따라 변한다. 투표자가 투표권을 보유한 자인지를 확인하는 함수는 $$member\left(  \right)$$와 $$verify\left(  \right)$$이다. 함수 $$member$$는 투표자의 DID가 투표대상이 맞는지 (유권자 리스트에 포함되어 있는지)를 확인하는 함수이고, 함수 $$verify$$는 투표권 데이터가 투표자의 DID로부터 올바르게 생성된 데이터인지를 확인하는 함수이다.

만약 영지식증명이 적용되지 않은 투표시스템이라면, 두 함수 $$member$$와 $$verify$$는 검표과정에서 검표자 (선거관리 위원회)가 수행해야 할 함수이다. 반면 그림 2과 같이 영지식증명이 적용된 투표시스템에서는 두 함수를 투표자가 스스로 수행하여 자가검증한다. 그 후 자가검증을 잘 수행하였다는 증거 데이터 $$\pi $$를 자신의 투표 결과와 함께 블록체인에 기록한다. 투표가 끝난 후, 검표자는 증거 데이터 $$\pi $$에 오류가 없는지 확인함으로써 정당한 투표권자에 의한 투표임을 확인한다.

그림 2과 같은 투표 시스템은 온라인 투표의 실현 가능성을 제고하였으나, 아직 완벽하다고 할 수는 없다. 그 이유는 투표 시스템이 선거관리 위원회라는 신뢰받는 제 3자(trusted third-party)에 의존하고 있기 때문이다. 믿음을 기반으로 동작하는 선거관리 위원회가 타락하면 부정투표가 가능할 수 있다. 예를 들어 선거관리 위원회는 투표자들의 DID들과 각각에 대응되는 투표권 데이터들을 모두 보관하고 있을 수 있다. 이는 곧 선거관리 위원회가 타인의 투표권을 사용 할 수 있다는 의미이다. 만약 선거관리 위원회가 컴퓨터 프로그램이고 서버라면, 해커에 의해 모든 기록들이 탈취 될 가능성 또한 배제할 수 없다. 뿐만 아니라, 선거관리 위원회가 임의로 특정 투표를 누락시킬 수 도 있다.

비록 아직 완벽하지는 않지만, 그림 2과 같이 영지식 증명이 적용된 투표 시스템의 제안은 실용 가능성을 보여주었다. 실제로 영지식 증명을 적용하여 기밀성을 확보한 또다른 투표 시스템이 스위스의 업체 Luxoft와 스위스 루체른 대학의 공동연구로 2018년에 개발되었으며, 스위스 Zug시의 주관 하에 72명의 시민을 대상으로 시범투표가 시행되었다. 자세한 내용은 <https://www.stadtzug.ch/newsarchiv/615796> 에서 확인 할 수 있다.

[^1]: DID는 사용자가 자신이 신뢰받을 수 있는 개인 혹은 기관, 장치라는 정보가 기록된 디지털 데이터이다.
[^2]: PKI는 public-key encryption 기술을 활용해 증서를 발급하는 기반 시설(infrastructure)이다. PKI를 이용하여 DID를 네트워크상에서 안전하게 전송 할 수 있다.