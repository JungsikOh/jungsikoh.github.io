---
layout: post
title: "Halo Effect"
description: "using Ray Sphere Inspection"
date: 2024-10-07
tags: Graphics, Volume Lighting
comments: true
use_math: true
---

Halo effect는 일종의 Volume Rendering과 유사하다. 빛 주변의 입자들이 광원과 충돌하면서 광원의 에너지가 0이 될 때까지 충돌→산란→충돌을 반복하는 것이다. 하지만, Real-time Rendering에서는 이것을 실시간으로 계산하기에는 비용이 높다.

$$
P(r) = 1- \frac{r^2}{R^2} \tag{1}
$$

여기서, R은 빛이 영향을 미치는 radius이고, r은 빛의 광원에서부터 입자까지의 거리이다. 이러한 계산들을 매번 반복하기는 힘들다. 그렇기 때문에 Post-processing 단계에서 해당 화면에 그려지는 Halo의 에너지 총량을 계산해서 그 값을 보여주는 방식을 이용한다.

$$
\int\limits_{t_1}^{t_2}P(r)dt \tag{2}
$$

화면에서 Ray를 발사해 Halo를 통과할 때 시작지점을 t1, 끝나는 지점을 t2라고 정의한다. 이 두 지점에서 광원이 누적되는 합을 구하는 것이다. 여기서 중요한 것은 r을 어떻게 정의하는가이다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/41c8241d-4825-4254-a2df-bc5b03ef7998/6b9f9d9a-4550-413b-bca4-e81b0109988c/image.png)

$$
\vec{r} = \vec{P} + t\vec{v} \tag{3}
$$

(3)공식과 같이 r을 정의할 수 있다. P는 화면에서 광원까지의 벡터이고 t는 매개변수로써, 구와 화면광선의 교차점에 대한 해 값이다. t란 광선의 진행상황이 얼마나 진행되었는가에 대한 값이라고 생각해도 된다. 

광선의 방정식은 R(t) = P + t*d로 정의되기 때문이다. 그렇기 때문에 v에다가 t를 곱하는 것도 광선의 진행도가 어디까지 갔는냐를 알기위한 공식이라고 보면된다.

우리는 이제, P함수에 대한 공식도 알고 r에 대한 정의도 알고 있다. 

$$
\int\limits_{t_1}^{t_2}P(r)dt=\int\limits_{t_1}^{t_2}(1-(\vec{P}+t\vec{v})(\vec{P}+t\vec{v}))/R^2\ dt \tag{4}
$$

$$
=(1-\frac{\vec{P}\cdot\vec{P}}{R^2})(t_2-t_1)-\frac{\vec{P}\cdot\vec{v}}{R^2}(t_2^2-t_1^2)-\frac{\vec{v}\cdot\vec{v}}{3R^2}(t_2^3-t_1^3) \tag{5}
$$

해당 적분을 풀면 (5)와 같은 식을 전개할 수 있다. 최종적으로 화면의 해당 값과 빛의 색깔 값을 이용하면, 아래와 같은 화면을 얻을 수 있다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/41c8241d-4825-4254-a2df-bc5b03ef7998/312575dc-3c37-43f6-9793-c0646d4929a0/image.png)

이대로만 계산해서 화면에 보여준다면, 생기는 문제점은 두 가지이다.

1. 카메라가 뒤를 봐도 Halo가 발생함. 
2. Post-processing 단계에서 적용되므로 Halo가 벽을 통과하여 작동함.

1번 문제부터 들여다보자. 해당 문제가 발생하는 이유는 $pdotv$나 $t_1$, $t_2$가 부호가 바뀌어도 적분식의 결과 값이 그대로 적용되버린다. 그렇기 때문에, $t_2 >0$이라는 조건을 추가해서 계산을 진행하지 않는다면 문제는 간단하게 해결된다.

2번 문제의 경우, 아래 사진과 같이 벽이 완전히 빛을 가리고 있지만, 우리는 적분식을 통해서 t2와 t1만을 가지고 계산하기 때문에 아래와 같은 주황색 부분을 계산해서 화면에 그리게 된다. 

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/41c8241d-4825-4254-a2df-bc5b03ef7998/7e78a6b7-91c7-4f6b-8c90-e065c0682b38/image.png)

위 사진과 같은 문제를 방지하기 위해서, 기존의 Light의 그림자를 그리기 위해서 그려두었던 Shadow Mapping Texture를 활용하는 방안을 생각하였다. 

다만, 해당 방법의 문제점은 벽이 두 겹으로 되어있을 때, 첫번 째 벽에 대해서는 정상작동하지만, 두번째 벽에 대해서는 Halo Effect로 인해서 해당 벽의 특정 부분이 밝아진다는 것이다.

![image.png](https://prod-files-secure.s3.us-west-2.amazonaws.com/41c8241d-4825-4254-a2df-bc5b03ef7998/23afd87a-c191-4795-a965-df155961f325/image.png)