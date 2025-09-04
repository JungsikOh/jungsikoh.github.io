---
layout: post
title: "Monte Carlo Simulation"
description: "Understanding Monte Carlo Methods"
date: 2025-06-14
tags: Graphics MonteCarlo Rendering Simulation
comments: true
use_math: true
---

# Monte Carlo(MC) Methods
MC 방법들이라고 불리는 이유는, 같은 원리를 다양한 문제에 적용할 수 있고, 서로 다른 알고리듬이 문제별로 존재하기 때문이다. 이 모든 알고리듬의 공통점은 무작위(확률적) 샘플링을 활용한다는 것이다. 대표적으로 사용되는 두 가지 영역이 존재한다.
- Simulation
    - 무작위 샘플을 이용해 과정을 돌려 결과를 예측
- Intergration
    - 고차원 적분에서는 전통적인 리만 합 방식의 계산량이 급증
    - MC 적분은 수렴 속도는 빠르지 않지만 비용 대비 합리적인 근사값 제공

하지만, 첫 시뮬레이션이 1시간 32분 결과를 도출하였고, 1억 번 반복 후 평균을 내니 역시 1시간 32분 결과를 도출해낼 수도 있다. 즉, 처음 한 번에 이미 '정답'을 뽑아 냈지만, **사전에 알 수 없다는 특징**이 존재한다.

naive한 Hit-or-Miss MC기법은 균등 난수를 사용해 Hit/Miss 카운트만으로 면적을 근사하는 기법이다. 샘플 수를 늘릴수록 정화도는 개선되나, 수렴속도(=표준편차가 감소하는 크기)는 $O(1/\sqrt{N})$으로 느리다.

# Monte Carlo Simulation
![image](https://www.scratchapixel.com/images/monte-carlo-methods/MCSimulation6.png?)

먼저, 광선이 물체 내부로 들어가면 물체를 구성하는 원자(atom)들과 충돌하게 될 것이다. 그렇게 되면, 광선이 원자랑 충돌 후 어느 방향으로 갈지에 대한 정보 $V$가 존재하게 되는데, 이 $V$와 부딪힌 원자의 위치 정보 $P_z$를 통해서, 우리는 표면 $s$까지 얼마나 남았는지를 알 수 있게 된다.

