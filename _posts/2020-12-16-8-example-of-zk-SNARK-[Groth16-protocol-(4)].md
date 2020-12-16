---
layout: post
title:   ZKP 기술보고서 - 8. zk-SNARK의 예 [Groth16 프로토콜 (4)]
date:   2020-12-16 00:00:01 +09:00
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

Groth16 프로토콜: QAP와 ECC를 활용한 zk-SNARK 프로토콜
------------------------------------------------------

하나의 증명상황을 가정하겠다. 증명하고자 하는 명제 $$S$$는 "증명자는 비공개 정보 $$u_1$$,$$u_2$$를 알며, 이를 사용하여 함수 $$f\left( u_1,u_2 \right)$$ 충실히 계산 했고, 그 결과로 출력 $$y_1$$, $$y_2$$를 얻었다"이다. 여기서 함수 $$f$$와 출력 값 $$y_1$$,$$y_2$$는 공개된 정보이다. 이 예시에서는 비공개 정보를 2개, 공개 정보를 2개로 설정하였지만, Groth16프로토콜에서는 일반적으로 비공개정보의 개수와 공개 정보의 개수를 제한하지 않는다.

증명자는 QAP를 사용하여 $$f\left( u_1,u_2 \right)$$를 분해하였다고 가정하겠다. QAP의 결과로 생성된 다항식들은 $$v_i\left( x \right)$$, $$w_i\left( x \right)$$, $$y_i\left( x \right)$$, $$t\left( x \right)$$이며 계수는 $$c_i$$이다. 여기서 $$i=1,\cdots ,m$$이고, $$c_1=u_1$$, $$c_2=u_2$$, $$c_m-1=y_1$$, $$c_m=y_2$$이다. 즉, $$c_1,c_2,\cdots ,c_m-2$$는 모두 비공개 정보이며, $$c_m-1$$과 $$c_m$$은 공개된 정보이다. 증명자는 복원다항식 $$p\left( x \right)$$를 계산하였고, 몫 다항식 $$h\left( x \right)$$를 계산하였다고 가정하겠다.

덧붙여 함수 $$f$$는 공개된 정보이기 때문에 검증자를 포함한 누구나 $$i=1,\cdots ,m$$에 대한 QAP 다항식들 $$v_i\left( x \right)$$, $$w_i\left( x \right)$$, $$y_i\left( x \right)$$, $$t\left( x \right)$$를 얻을 수 있다. 그러나 증명자 이외의 그 누구도 비공개 정보 $$u_1$$,$$u_2$$를 알지 못하는 한 QAP 계수들$$c_1,c_2,\cdots ,c_m-2$$를 알 수는 없고, 그러므로 $$h\left( x \right)$$를 알지 못한다.

QAP 결과로 생성된 다항식과 계수들을 공개정보 혹은 비공개정보의 기준으로 분류하면 표 5와 같다.

| 공개 정보   | QAP 계수들 중 $$c_m-1$$,$$c_m$$                      |
| ----------- | ------------------------------------------------------------ |
| $$\qquad$$ $$\quad$$ | QAP 다항식 $$v_i\left( x \right)$$,$$w_i\left( x \right)$$,$$y_i\left( x \right)$$ for $$i=1,\cdots ,m$$ |
| $$\qquad$$ $$\quad$$ | QAP 근 다항식 $$t\left( x \right)$$                            |
| 비공개 정보 | QAP 계수들 중 $$c_1,\cdots ,c_m-2$$                |
| $$\qquad$$ $$\quad$$ | QAP 복원다항식 $$p\left( x \right)$$                           |
| $$\qquad$$ $$\quad$$ | QAP 몫 다항식 $$h\left( x \right)$$                            |

표5 공개/비공개 특성에 따른 QAP 결과의 분류

Groth16 프로토콜의 증명-검증 과정이 프로토콜 4에 소개되어 있다. 프로토콜에서 사용되는 연산들에서 정수간의 사칙연산과 타원곡선 점 덧셈연산, 그리고 타원곡선 점간의 pairing을 잘 구분하여 읽을 필요가 있다. 예를 들어, 같은 $$+$$기호더라도 앞 뒤의 피연산자가 정수이면 정수덧셈이며, 피연산자가 타원곡선의 점이면 점 덧셈연산이고, 피연산자가 pairing 결과이면 정수 곱셈임을 인지하여야 한다.



|               프로토콜 4. Groth16 프로토콜                   |
|------------------------------------------------------------ |
| Setup: 제 3자가 실행한다. Common reference string (CRS)를 생성한다. |


 1. 무작위 비공개 정수 리스트 $$\tau :=\left( \alpha ,\beta ,\gamma ,\delta ,x \right)$$를 생성한다 ($$\tau $$는 증명자에게 알려져서는 안된다).<br/> 2. 타원곡선상의 두 점(generator) $$G$$와 $$H$$를 선택하여 공개한다.<br/> 3. 첫 번째 CRS인 $$\sigma _G$$를 생성한다:<br/>
$$\sigma _G:= \begin{Bmatrix} \left [ \alpha \right]_G, \left[ \beta \right]_G,\left[ \delta \right]_G, \left\{ {\left[ {x^i} \right]}_G \right\}_{i=0}^{n-1}, \\ \left\{ {\left[ \frac{\beta v_i\left( x \right)+\alpha w_i\left( x \right)+y_i\left( x \right)}{\gamma } \right]}_G \right\}_{i=m-1}^{m},\\ \left\{ {\left[ \frac{\beta v_i\left( x \right)+\alpha w_i\left( x \right)+y_i\left( x \right)}{\delta } \right]}_G \right\}_{i=1}^{m-2}, \\ \left\{ {\left[ \frac{x^i t(x)}{\delta } \right]}_G \right\}_{i=0}^{n-2}& \\ \end{Bmatrix}$$<br/><br/>
4. 두 번째 CRS인 $$\sigma _H$$를 생성한다:<br/>$$\sigma _H:=( \left [ \beta \right]_H, \left[ \gamma \right]_H, \left[ \delta \right]_H, \left\{ {\left[ {x^i} \right]}_H \right\}_{i=0}^{n-1} )$$.<br/><br/>5. $$\sigma _G$$와 $$\sigma _H$$를 공개한다.


| Prove: 증명자가 수행한다. 증명자만이 아는 비공개 정보와 CRS $$\sigma _G$$, $$\sigma _H$$를 조합하여 증거를 생성한다. |
|--------------------------------------------------------------------------------------------------------------- |
| 1. 무작위 비공개 정수 $$r$$,$$s$$를 생성한다 ($$r$$,$$s$$는 검증자에게 알려져서는 안된다).|
|<br/> 2. CRS $$\sigma _G$$를 사용하여 첫 번째 증거 $$\left[ A \right]_G$$를 생성한다:<br>$$\left[ A \right]_G:= \left[ \alpha \right]_G +\sum\nolimits_{i=1}^m c_i \sum\nolimits_{k=0}^{n-1} a_{i,k} {\left[ {x^k} \right]}_G + r{\left[ \delta \right]}_G$$,<br/>여기서 $$v_i\left( x \right)=\sum\nolimits_{k=0}^{n-1} a_{i,k} x^k$$.<br/><br/>|
|3. CRS $$\sigma_H$$를 사용하여 두 번째 증거 $$\left[ B \right]_H$$를 생성한다:<br/>$$\left[ B \right]_H:= \left[ \beta \right]_H + \sum\nolimits_{i=1}^m c_i \sum\nolimits_{k=0}^{n-1} b_{i,k} {\left[ {x^k} \right]}_H + s{\left[ \delta \right]}_H$$,<br/>여기서 $$w_i\left( x \right)=\sum\nolimits_{k=0}^{n-1}b_{i,k} x^k$$.<br/><br/>|
|4. CRS $$\sigma_G$$를 사용하여 세 번째 증거 $${\left[ C \right]}_G$$를 생성한다:<br/>$${\left[ C \right]}_G:=\sum\nolimits_{i=1}^{i=m-2} c_i{\left[ \frac{\beta v_i(x)+\alpha w_i (x) + y_i (x)}{ \delta } \right]}_G + \sum\nolimits_{k=0}^{n-2} d_k {\left[ \frac{x^k t(x)}{\delta } \right]}_G + s{\left[ A \right]}_G + r{\left[ B \right]}_G - rs{\left[ \delta \right]}_G$$,<br/>여기서 $$h( x ) = \sum\nolimits_{k=0}^{n-2} d_k x^k$$이고, $${\left[ B \right]}_G := {\left[ \beta \right]}_G + \sum\nolimits_{i=1}^{m} c_i \sum\nolimits_{k=0}^{n-1} b_{i,k} {\left[ x^k \right]}_G + s{\left[ \delta \right]}_G$$.<br/><br/>|
|5. 세 개의 증거 리스트 $$\pi := ( {\left[ A \right]}_G, {\left[ B \right]}_H, {\left[ C \right]}_G )$$를 공개한다. |


| Verify: 검증자가 수행한다. 검증자가 아는 공개정보와 증거 $$\pi $$, 그리고 CRS $$\sigma _G$$,$$\sigma_H$$를 조합하여 증거를 생성한다. |
|--------------------------------------------------------------------------------------------------------------- |
|1. 증거 $$\pi $$의 데이터들이 모두 타원곡선상의 점이 맞는지 확인한다.<br/>|
|2. 증거 $$\pi $$를 사용하여 $$LHS:={\left[ A \right]}_G\cdot {\left[ B \right]}_H$$를 계산한다.<br/>|
|3. 증거 $$\pi $$와 CRS를 사용하여 다음을 계산한다:<br/>$$RHS:={\left[ \alpha \right]}_G\cdot {\left[ \beta \right]}_H + \left( \sum\nolimits_{i=m-1}^{m} c_i {\left[ \frac{\beta v_i(x)+\alpha w_i (x) + y_i (x)}{ \gamma } \right]}_G \right)\cdot {\left[ \gamma \right]}_H + {\left[ C \right]}_G \cdot {\left[ \delta \right]}_H$$.<br/><br/>|
|4. $$LHS=RHS$$를 확인하여, 만약 등호가 성립하면 증거를 인정하고, 그렇지 않으면 증거를 부정한다. |



Groth16의 영지식 증명 성능 분석
-------------------------------

***Q. (완전성) 만약 증명자가 비공개정보를 알고있다면, 검증자는 $$LHS=RHS$$를 확인함으로써 이를 납득할 수 있는가?***

*A. 그렇다.*

연산 $${\left[ \cdot \right]}_G$$와 $${\left[ \cdot \right]}_{G\cdot H}$$의 정의를 사용하여 $$LHS$$와 $$RHS$$를 각각 정리하면 ***정리 1***의 조건 인 $$p\left( x \right)=t\left( x \right)h\left( x \right)$$를 유도 할 수 있다. 이는 곧 ***정리 3***[^1]에 의해 증명자는 써킷을 완벽히 복원 할 수 있고 따라서 모든 비공개 정보를 다 알고 있음을 말한다.

$$p\left( x \right)=t\left( x \right)h\left( x \right)$$를 유도하기 위해 ,먼저 $${\left[ A \right]}_G$$ 와 $${\left[ B \right]}_H$$를 정리하면 각각 다음과 같다:

<br/><br/>

$${\left[ A \right]}_G = {\left[ \alpha + \sum\nolimits_{i=1}^{m} c_i v_i (x) + r \delta \right]}_G  \qquad$$

$${\left[ B \right]}_H = {\left[ \beta + \sum\nolimits_{i=1}^{m} c_i w_i (x) + s \delta \right]}_H \qquad (17)$$

<br/><br/>

그리고, $${\left[ C \right]}_G$$를 정리하면 다음과 같다:

<br/><br/>

$$
{\left[ C \right]}_G = {\left[
\frac{\sum\nolimits_{i=1}^{i=m-2} c_i ( \beta
v_i ( x ) + \alpha w_i ( x ) + y_i ( x ) ) + h ( x ) t ( x )} { \delta } + sA + rB - rs \delta \right]}_G  \qquad (18)
$$

<br/><br/>

먼저 $$LHS$$를 정리하면,

<br/><br/>

$$
LHS = {\left[ A \right]}_G \cdot {\left[ B \right]}_H = {\left [ \alpha + \sum\nolimits_{i=1}^{m} c_i v_i(x) + r\delta \right]}_G \cdot {\left [ \beta + \sum\nolimits_{i=1}^{m} c_i w_i ( x ) + s \delta \right]}_H  
$$

<br/><br/>

$$
= { \left [ \alpha \beta + \sum\nolimits_{i=1}^{m} c_i v_i ( x ) \cdot  \sum\nolimits_{i=1}^{m} c_i w_i ( x ) + ( \alpha + r \delta ) \sum\nolimits_{i=1}^{m} c_i w_i ( x ) + ( \beta +s\delta ) \sum\nolimits_{i=1}^{m} c_i v_i ( x )+ \alpha s \delta + \beta r \delta + r s {\delta }^{2} \right]}_{G\cdot H}  \qquad (19)
$$

<br/><br/>

그리고 $$RHS$$를 정리하면,

<br/><br/>

$$
RHS = { \left [ \alpha \right]}_G \cdot { \left [ \beta \right]}_H + ( \sum\nolimits_{i=m-1}^{m} c_i { \left [ \frac { \beta v_i ( x ) + \alpha w_i ( x ) + y_i ( x ) }{ \gamma } \right]}_G ) \cdot { \left [ \gamma \right]}_H + { \left [ C \right]}_G \cdot { \left [ \delta \right]}_H
$$

<br/><br/>

$$
= { \left [ \alpha \beta + \sum\nolimits_{i=1}^{m} c_i ( \beta v_i (x) + \alpha w_i (x) + y_i (x) ) + h ( x ) t ( x ) + s \delta A + r \delta B - r s { \delta }^{2} \right]}_{G\cdot H} \qquad (20)$$

<br/><br/>

$$LHS$$와 $$RHS$$에 공통으로 존재하는 항들을 제거하면 다음과 같다:

<br/><br/>

$$
\frac {LHS}{RHS} = { \left[ \sum\nolimits_{i=1}^{m} c_i v_i ( x ) \cdot \sum\nolimits_{i=1}^{m} c_i w_i ( x ) - \sum\nolimits_{i=1}^{m} c_i y_i ( x ) - h ( x ) t ( x ) \right]}_{G \cdot H}
$$

<br/><br/>

$$
\qquad +{\left[ s \delta \sum\nolimits_{i=1}^{m} c_i v_i ( x ) + r \delta \sum\nolimits_{i=1}^{m} c_i w_i ( x ) \right]}_{G \cdot H} \qquad (21)
$$

<br/><br/>

$$
\qquad +{ \left [ \alpha s \delta + \beta r \delta + 2 r s {\delta
}^{2} - s \delta A - r \delta B \right]}_{G \cdot H}
$$

<br/><br/>

식 (6)에 의 $$A$$와 $$B$$를 대입하면,

$$
\frac{LHS}{RHS}={ \left [ \sum\nolimits_{i=1}^{m} c_i v_i ( x ) \cdot \sum\nolimits_{i=1}^{m} c_i w_i ( x ) - \sum\nolimits_{i=1}^{m} c_i y_i ( x ) - h ( x ) t ( x ) \right]}_{G \cdot H} \qquad (22)
$$

<br/><br/>

복원방정식 $$p\left( x \right)$$의 정의를 대입하면 식 (7)은 다음과같다:

<br/><br/>

$$
\frac {LHS}{RHS}={ \left [ p ( x ) - h ( x ) t ( x ) \right]}_{G \cdot H} \qquad (23)
$$

<br/><br/>

결과적으로, $$LHS=RHS\Leftrightarrow p\left( x \right)=h\left( x \right)t\left( x \right)$$이고, 정리 1에 의해 관계식 $$LHS=RHS$$와 명제 "증명자가 써킷을 완벽히 복원 할 수 있다"는 것이 등가이다.

***Q. (건실성) 만약 거짓 증명자가 비공개 정보를 모른다면, 제출한 거짓 증거가 $$LHS=RHS$$를 만족시킬 수 있는가?***

*A.* CRS의 내용이 암호화 되어있지 않으면 거짓 증명자가 검증자를 쉽게 기만 할 수 있다. 그러나 Groth16 프로토콜에서는 *CRS가 암호화 되어 있기 때문에 그렇게 하기가 어렵다.*

만약 Setup 과정에서 비공개 정수 리스트 $$\tau =\left( \alpha ,\beta ,\gamma ,\delta ,x \right)$$의 값들이 공개된다면 거짓 증명자가 검증자를 속이는 것은 매우 쉽다. $$x$$는 변수가 아닌 정수 상수이기 때문에 $$ v_i ( x )$$, $$w_i( x )$$, $$y_i( x )$$, $$t( x )$$,그리고 $$h ( x )$$도 더 이상 다항식이 아닌 어떤 정수이다. 다음과 같이 정의하자:
<br/><br/>

$$
V := \sum\nolimits_{i=1}^{m-2} c_i v_i ( x ), \quad
W := \sum\nolimits_{i=1}^{m-2} c_i w_i ( x ), \quad
Y := \sum\nolimits_{i=1}^{m-2} c_i y_i ( x ), \quad
H := h ( x )
$$

<br/><br/>

이제 거짓 증명자가 할 일은 식 (8)의 계산 결과가 1이 되도록 하여 정리 3[^1]의 조건을 만족시키는 적절한 정수 $$V$$, $$W$$, $$Y$$, $$H$$를 임의로 선택하는 것이다. 예를 들어, $$V,W,Y$$를 무작위로 선택 한 후, 다음의 조건을 만족하는 정수 $$H$$를 찾는다:
<br/><br/>

$$
( V + \sum\nolimits_{i=m-1}^{m} c_i v_i ( x ) ) ( W + \sum\nolimits_{i=m-1}^{m} c_i w_i ( x )) - ( Y + \sum\nolimits_{i=m-1}^{m} c_i y_i ( x )) = t ( x ) H \qquad (24)
$$  

<br/><br/>
마지막으로 거짓 증명자가 최종적으로 제출 할 거짓 증거는 다음과 같다:
<br/><br/>

$$
\pi ' =  \begin{pmatrix} { \left [ [A' \right]}_G := { \left [ \alpha \right]}_G + { \left [ V \right]}_G + { \left [ \sum\nolimits_{i=m-1}^{m} c_i v_i ( x ) \right]}_G + r { \left [ \delta \right]}_G, \\
{ \left [B' \right]}_H := { \left [ \beta \right]}_H + { \left [ W \right]}_H + { \left [ \sum\nolimits_{i=m-1}^{m} c_i w_i ( x ) \right]}_H + s{ \left [ \delta \right]}_H, \\
{ \left [ C' \right]}_G := { \left [ \frac { ( \beta V + \alpha W + Y )}{ \delta } \right]}_G + { \left [ \frac { t ( x ) H }{ \delta } \right]}_G + s { \left [
A' \right]}_G + r { \left [ B' \right]}_G - rs{ \left [ \delta \right]}_G \\
\end{pmatrix} \qquad (25)
$$  

<br/><br/>
거짓 증거를 사용하여 Verify과정의 $$LHS$$와 $$RHS$$를 계산 해 보면 $$LHS=RHS$$인 것을 알 수 있다.

Groth16 프로토콜에서는 Setup 과정에서 생성되는 비공개 정수 리스트 $$\tau =\left( \alpha ,\beta ,\gamma ,\delta ,x \right)$$가 암호화 후 공개된다. 그렇기 때문에 증명자는 $$\tau $$의 값들을 알지 못한다. 구체적으로, $$x$$를 모르기 때문에 과 같이 정리 1의 조건을 만족시키는 값 $$V,W,Y,H$$도 찾지 못한다. 이 뿐만 아니라, $$\alpha ,\beta ,\delta $$를 모르기 때문에 와 같은 거짓 증거를 만들어 낼 수도 없다.

***Q. (영지식성) 검증자는 증거 $$\pi $$로부터 비공개 정보 $$c_1,\cdots , c_m-2$$를 알아 낼 수 있는가?***

*A.* ECC에 의해 암호화된 증거를 복호하여 *비공개 정보를 추출하는 것은 매우 어렵다.* 비록 비공개 정보를 추출하지 못하더라도, 악의적인 검증자가 증거를 재사용 하려는 경우가 있을 수 있다. 이러한 *재사용 시도는 증명자가 임의로 생성하는 비공개 값 $$r,s$$에 의해 차단 된다.*

만약 $$r,s$$가 존재하지 않는다면, 혹은 $$r,s$$의 값이 노출된다면, 악의적인 검증자가 증거를 조작하여 재사용 할 수 있다. 여기서 악의적인 검증자가 Setup 과정에서 생성된 비공개 정수 리스트 $$\tau$$를 알고 있다고 가정하겠다. 동일한 증명자의 비공개 정보는 일정하다고 가정하겠다. 증명자는 프로토콜을 $$N$$회 사용하였으며, 따라서 검증자는 $$N$$개의 증거를 보유하고 있다. 여기서 $$N\>m-2$$이다. $$k=1,\cdots ,N$$번째 제출된 증거를 $$\pi_k = ( {\left[ A_k \right]}_G, { \left [ B_k \right]}_H, {\left[ C_k \right]}_G )$$라 하겠다. 증명자가 $$k$$번째 증거를 생성할 때 사용 한 파라미터를 $$\tau_k = ( \alpha_k, \beta_k, \gamma_k, \delta_k, x_k )$$,$$r_k$$,$$s_k$$라 하겠다. 다음의 알고리즘은 $$\tau_k, r_k, s_k$$의 값이 검증자에게 알려져 있을 때, 검증자가 증거를 재사용 할 수 있도록 하는 알고리즘이다:


| 알고리즘 5. 보안 파라미터인 $$\tau_k, r_k, s_k$$가 공개된 경우 Groth16 프로토콜에 사용 될 수 있는 증거 재사용 알고리즘 |
| ------------------------------------------------------------ |
| 1. 검증자는 $$k=1,\cdots ,N$$에 해당하는 각각의 증거에 대하여 다음을 계산한다:<br/><br/>$$D_k := {\left [A_k \right]}_G - {\left [ \alpha_k \right]}_G - { \left [ r_k \delta_k \right]}_G$$, $$\qquad$$ (26) <br/> $$E_k := { \left [ B_k \right]}_H - { \left [ \beta_k \right]}_H - { \left [ s_k \delta_k \right]}_H$$. $$\qquad$$ (27)<br/><br/>2. 계산 결과는 다음의 관계식을 갖는다: $$D_k = \sum\nolimits_{i=1}^{m-2} v_i ( x_k ) { \left [ c_i \right]}_G$$, $$E_k = \sum\nolimits_{i=1}^{m-2} w_i ( x_k ){ \left [ c_i \right]}_H$$.<br/>3. $${ \left [ c_i \right]}_G$$와 $${ \left [ c_i \right]}_H$$에 대하여, $$D_k$$와 $$E_k$$로부터 선형 연립방정식에 해를 구한다.<br/>4. $${ \left [ c_i \right]}_G$$와 $${ \left [ c_i \right]}_H$$를 사용하여 $${ \left [ C_N \right]}_G$$로부터 다음을 계산한다:<br/>

$$
{ \left [ \frac { h ( x_N ) t ( x_N )}{ \delta_N } \right]}_G = { \left [ C_N \right]}_G - \sum\nolimits_{i=1}^{i=m-2} \frac { ( \beta_N v_i ( x_N ) + \alpha_N w_i ( x_N ) + y_i ( x_N ) ) }{\delta_N} { \left [ c_i \right]}_G $$<br/>$$- s_N{ \left [A_N \right]}_G - r_N { \left [ B_N \right]}_G - {\left[ r_N s_N \delta_N \right]}_G \qquad (28)
$$

<br/>5. (10) 의 계산 결과를 사용하여 다음을 계산한다:
<br/>

$$
{ \left [ h ( x_N ) t ( x_N ) \right]}_G = \delta_N { \left [ \frac { h ( x_N ) t ( x_N )}{ \delta_N} \right]}_G \qquad (29)
$$

<br/><br/>
6.새로운 $$ \tau _{N+1} := ( \alpha ,\beta ,\delta ,\gamma ,x )$$에 관하여 다음의 재사용 증거를 생성한다:
<br/><br/>

$$
\pi ' = \begin{pmatrix} { \left [ A' \right]}_G := { \left [ \alpha +r \delta \right]}_G + \sum\nolimits_{i=1}^{m} v_i ( x_N ) { \left [ c_i \right]}_G, \\ { \left [ B ' \right]}_H = { \left [ \beta + s \delta \right]}_H + \sum\nolimits_{i=1}^{m} w_i ( x_N ) { \left [ c_i \right]}_H, \\ { \left [ C ' \right]}_G = \sum\nolimits_{i=1}^{i=m} \frac { ( \beta v_i ( x_N ) + \alpha w_i ( x_N ) + y_i ( x_N ) ) }{ \delta }{ \left [ c_i \right]}_G - \sum\nolimits_{i=m-1}^{i=m} \frac{ ( \beta v_i ( x ) + \alpha w_i ( x ) + y_i ( x ) )}{ \delta }{ \left [ c_i \right]}_G \\  +\frac { {\left [ h ( x_N ) t ( x_N ) \right]}_G}{ \delta } + { \left [ s A '+ r B ' - r s \delta \right]}_G \\
\end{pmatrix} \qquad (30)
$$

<br/><br/>
알고리즘 5에 의해 생성된 재사용 증거 $$\pi '$$의 완전성을 조사하기 위해 $$LHS$$와 $$RHS$$를 계산 해 보면, 각각 과 의 관계에서 $$x$$대신 $$x_N$$이 대입된 것과 같다. 따라서 $$x$$가 아닌 $$x_N$$에 대하여 의 등호를 만족하므로, 재사용 증거 $$\pi '$$는 완전하다.

결과적으로, 악의적인 검증자가 보안 파라미터인 $$\tau ,r,s$$를 알고 있을 때 알고리즘 5의 방법으로 증거를 재사용 할 수 있음을 보였다. 그러나 다행히도, Groth16은 Prove과정에서 증명자에게 임의의 정수 값 $$r$$과 $$s$$를 선택하게 하여 공개하지 않도록 한다. 설령 검증자가 $$\tau $$를 알고 있다 하여도 $$r$$,$$s$$를 모르는 한, 와 을 계산 할 수 없기 때문에 적어도 알고리즘 5와 같은 방법으로는 증거를 조작하여 재활용 할 수 없다.

***Q. (간결성) 증거의 길이는 간결한가?***

*A.* Groth16에서 증거 데이터는 타원곡선의 좌표들이다. 따라서 증거의 길이의 단위를 타원곡선 좌표의 개수로서 정의하겠다. 결과적으로 증거의 길이는 $$3n+m+5$$으로, 연산 프로그램의 크기에 *선형적으로 비례한다.* 따라서 증거가 간결하다고 말할 수 있다.

프로토콜의 보안문제, 즉 건실성과 영지식성을 보장하기 위하여 매 증거 생성시마다 CRS가 갱신되어야 한다. 그리고 검증자는 검증을 위해 CRS와 증거 $$\pi $$를 함께 확보하여야 한다. 즉, 증거 $$\pi $$ 뿐만 아니라 CRS도 매번 공유되어야 하는 데이터이기 때문에, CRS의 길이 또한 증거로 포함할 수 있다.

프로토콜 4를 확인하면 증거 $$\pi$$의 길이는 3으로 고정되어 있음을 알 수 있다. 반면 증거 CRS의 길이는 연산 프로그램의 크기에 따라 변한다. 어떤 연산 프로그램을 program analysis한 결과로서 써킷에 선의 개수가 $$m$$개, 곱셈 게이트의 개수가 $$n$$개가 존재한다면, CRS의 길이는 총 $$3n+m+5$$이다. 결과적으로, 증거 $$\pi$$와 CRS의 총 길이는 $$3n+m+8$$ 이며, 이는 연산프로그램의 크기인 $$m$$과 $$n$$에 선형적이다.

정리) 보안 파라미터 $$\tau $$와 $$r$$,$$s$$의 중요성
-----------------------------------------------------

요약하면, Groth16 프로토콜의 건실성을 보장하기위한 필요조건은 파라미터 리스트 $$\tau $$가 거짓증명자에게 공개되지 않는것이다. 위의 분석을 통해 만약 $$\tau $$가 거짓증명자에게 알려지면, 거짓증명자는 검증자를 기만 할 수 있음을 보였다. 그렇기 때문에 프로토콜에서는 $$\tau $$를 ECC를 사용하여 암호화 한 후, 그 결과를 CRS라는 이름으로 공개하고, 원본 $$\tau $$는 즉각 소멸시킨다. 이렇게 사용 후 소멸시켜야 하는 파라미터들을 "보안폐기물(toxic waste)"이라 부른다.

한편 Groth16 프로토콜은 영지식성 보장을 위해 $$\tau $$와 $$r$$,$$s$$라는 이중보안장치를 사용한다. 만약 $$\tau $$와 $$r$$,$$s$$ 모두 악의적인 검증자에게 공개된다면, 악의적인 검증자가 알고리즘 5를 이용해 기 제출된 증거들을 수집하여 재활용 할 수 있음을 보였다. 만약 $$\tau $$나 $$\left( r,s \right)$$중 하나라도 노출되지 않는다면, 알고리즘 5를 사용 할 수 없다. $$\left( r,s \right)$$또한 $$\tau $$와 같이 보안폐기물이다.

다음은 $$\tau$$ 혹은 $$(r,s)$$가 노출되었을 때 발생할 수 있는 보안 문제를 정리한 표이다:

| 유출된 파라미터  |  건실성   | 영지식성  |
| :--------------: | :-------: | :-------: |
|      $$\tau$$      | 침해 가능 |    \-     |
|     $$(r,s)$$      |    \-     |    \-     |
| $$\tau$$와 $$(r,s)$$ | 침해 가능 | 침해 가능 |

표 6 보안 파라미터 $$\tau $$ 혹은 $$\left( r,s \right)$$가 노출되었을 때 건실성과 영지식성의 침해 가능 여부

보안 파라미터인 $$\tau $$를 생성하는 주체는 신뢰받는 제 3자(trusted third-party)인 것이 가장 이상적이지만, 제 3자를 정말로 신뢰할 수 있는가의 실용적인 문제가 고려 될 수 있다. 구체적으로 제 3자에 대한 신뢰를 법과 정책, 혹은 컴퓨터 프로토콜로써 완벽히 보장 할 수 있는가와 같은 문제이다. 여기서 "완벽한 신뢰"라는 표현은 $$\tau $$가 증명자나 검증자에게 알려질 가능성이 전혀 없음을 의미한다. 실용적인 환경에서는 제 3자에게 완벽환 신뢰를 기대하기 어렵다. 한 가지 예로 보안인증 시스템과 같이 검증자가 서버이고 증명자가 클라이언트인 경우가 있을 수 있다. 이 경우 검증자가 제 3자를 선별 및 구성한다. 검증자가 악의적인 의도가 있다면 제 3자와 모의하여 증명자를 기만할 수 있다.

보안 파라미터 $$r,s$$의 역할은 제 3자를 신뢰할 수 없는 경우에도 증명자의 비밀정보를 보호해주는 것이다. 파라미터 $$r,s$$는 증명자가 생성하고 사용 후 소멸시키기 때문에 검증자 혹은 제 3자가 그 값을 알아내는 것이 불가능하다. 결과적으로, 파라미터 $$r,s$$의 존재 덕분에 제 3자에 대한 의존이 필요 없어지게 된다.

*참고) Multi-party computation of CRS: "Powers of Tau ceremony"*

분석 결과와 같이, Groth16 프로토콜의 건실성과 영지식성은 파라미터 리스트 $$\tau$$(tau)에 의존하고 있다. Tau는 증명자에게는 절대 공개되어서는 안되고, 검증자에게도 공개되지 않으면 더 좋다. 이 때문에 Groth16 프로토콜에서는 제3자가 tau를 생성 한 후 암호화하여 공개하고 원본 tau는 즉시 소멸한다.

제 3자에 대한 의존을 완화하기 위해, 파라미터 tau의 생성을 분산화 하려는 노력이 시도된 적이 있다. ZCash가 2017년에 개최한 \"Powers of Tau ceremony\"가 그러한 노력이다[^2]. Powers of Tau는 전 세계의 참가자들이 공동으로 tau를 생성할 수 있도록 하는 행사이다. 누구나 기여 할 수 있지만, 누구도 최종 생성된 tau의 값은 알지 못한다. 다만 tau의 암호화 결과가 공개 될 뿐이다[^3]. 2019년에는 Plonk 프로토콜을 개발한 회사 Aztec protocol에서도 Powers of Tau와 유사한 "multiparty computing ceremony"를 개최하였다[^4].

아직 ZCash의 Powers of Tau가 정말로 안전한가에 관한 논란이 있을 수 있다. ZCash와 같이 ceremony의 형태로 다수가 참여하여 tau를 생성하면, tau의 값을 수시로 변경하기가 어렵다. 만약 tau가 증거를 생성하는데 한번 사용 된 직후 바뀌지 않는다면, 악의적인 검증자는 추가적인 조작 없이도 증거를 재사용 할 수 있다. 그럼에도 불구하고 분산화 계산을 도입하는 것은 훌륭한 접근이기도 하다. 더 많은 사람들이 참여할수록 tau를 악의적으로 조작하는 것이 어려워지기 때문이다. 실증연구를 통해 참여자 수와 tau의 변경 주기간의 trade-off 관계가 사용 환경에 따라 적절히 최적화 된다면, 보다 안전한 증명 프로토콜이 될 수 있을 것이다.

[^1]: 정리 3. Divisibility check: 복원다항식 $$p\left( x \right)$$가 써킷을 완벽히 복원할 수 있기 위한 필요충분조건은 $$p\left( x\right)$$를 $$t\left( x \right)$$로 나눴을 때 몫 다항식 $$h\left( x\right)$$가 존재하는 것이다.
[^2]: Zcash 세레모니 개최 포스트: https://www.zfnd.org/blog/powers-of-tau
[^3]: Zcash 세레모니 종료 포스트: https://www.zfnd.org/blog/conclusion-of-powers-of-tau
[^4]: Aztec 세레모니 개최 포스트: https://medium.com/aztec-protocol/aztec-announcing-our-ignition-ceremony-757850264cfe
