---
layout: post
title: "유니티에서 헤어, 물 렌더링 구현"
description: "게임캡스톤디자인 프로젝트"
modified: 2025-11-11
tags: [unity, rendering]
categories: [project]
---

![alt text](/images/posts/game-capstone/reflection.png)

### 요약

유니티에서 Geometry Shader와 Kajiya-kay, Marschner 모델을 사용한 헤어 렌더링과 ray marching을 사용한 유체 렌더링에 Screen Space Reflection을 구현한다.

## 서론

게임캡스톤디자인에서 libWetHair라는 유체, 헤어 시뮬레이션을 유니티로 옮겨보는 프로젝트를 진행했다. 나는 거기서 물리를 담당하는 팀원이 전달한 데이터를 바탕으로 헤어와 물을 렌더링하는 작업을 맡았다.

## 헤어

헤어 그래픽에서 한 일은 색상 표현을 위한 셰이딩 모델을 찾는 것과, 자연스러운 헤어 표현을 위해 적절한 지오메트리를 표현하는 것이다. 따라서 진행 내용도 이 두 가지에 초점을 두어 진행하였다.

![alt text](/images/posts/game-capstone/hair-final.png)

## 지오메트리

지오메트리 측면에서 보면, 풍성한 헤어의 모습을 표현하기 위해서 물리 연산에 사용하는 헤어의 수를 늘리는 것은 연산량의 한계로 인해 어려운 선택지다. 따라서 연산을 늘리지 않는 선에서 헤어를 풍성하게 만들기 위한 방법이 필요했고, 헤어 사이를 보관해 가짜 헤어를 생성하는 방법으로 진행했다.

![alt text](/images/posts/game-capstone/hair-line.png)

헤어 물리에서 들어오는 정보는 일렬로 늘어진 파티클이다. 따라서 파티클을 면적을 가지는 삼각형으로 변환할 필요가 있었고, 모든 점의 위치정보를 하나의 버퍼에 담아 전달한 뒤, 지오메트리 셰이더에서 한 마디를 두 삼각형으로 만드는 방법을 사용했다.

![alt text](/images/posts/game-capstone/spline.png)

우선 파티클을 자연스러운 곡선으로 표현하기 위해 2차 베지어 곡선을 사용했다. 한 가닥의 헤어를 파티클 사이의 중점을 찍어서 짧은 세그먼트로 만들고, 각각에 대해 지정된 3개의 점을 이용해 곡선을 표현할 수 있다.

![alt text](/images/posts/game-capstone/tess.png)

3개의 층으로 이루어진 헤어와 테셀레이션으로 생성된 버텍스 시각화

## Kajiya-Kay

![alt text](/images/posts/game-capstone/kajiya-kay.png)

Kajiya-kay 모델은 머리카락의 스페큘러를 묘사하기 위해 고안된 모델로, 퐁 모델이 camera 방향과 light의 reflection을 사용하는 것과 대비되게, 헤어를 1차원 선으로 보고 방향을 나타내는 tangent를 계산에 사용한다. 머리카락의 표면에서 normal과 tangent는 수직이므로, 퐁 모델의 cos 연산은 sin으로 바뀐다.

## Marschner

![alt text](/images/posts/game-capstone/Marschner.png)

Marscher 모델은 머리카락을 반투명한 큐티클 원통형으로 보고 두 단면에서 빛을 고려한다. 머리카락의 스페큘러는 3개의 빛이 결합된 결과이며, 바로 반사되는 빛 T, 내부를 통과하는 빛 TT, 산란되어 반사되는 빛 TRT이다.

[참고한 자료: Strand-based hair rendering in Frostbite](https://advances.realtimerendering.com/s2019/hair_presentation_final.pdf)

실제 구현에서는 이런 공식을 만드는 방법을 찾기 어려워, Kajiya-kay를 변형해 TRT를 모방하는 2차 하이라이트를 추가하는 방식으로 진행했다.

![alt text](/images/posts/game-capstone/simple-marschner.png)

## 물

물리 부분에서 유체는 파티클의 형태로 계산되었다. 이를 실제 부피를 가지는 물로 보여주기 위해서 ray-marching과 smooth min 함수를 사용했다.

## ray marching, smin

3차원에서 구는 구 내부에 음수 값, 외부에 양수값, 경계에서 0을 가지는 함수로 표현할 수 있다. 만약 모든 구의 함수에 min을 적용하면 해당 구 영역을 합칠 수 있다. smooth min은 min을 부드럽게 이어주는 함수다. smooth min으로 형성된 영역을 ray marching을 이용해 어느 픽셀에 그려지는지 계산했다.

![alt text](/images/posts/game-capstone/ray-marching.png)

![alt text](/images/posts/game-capstone/min.png)

![alt text](/images/posts/game-capstone/smin.png)

![alt text](/images/posts/game-capstone/water-mid.png)

추가로 가장자리에서 반사가 더 되는 것을 표현하기 위해 Schlick's approximation을 이용해 배경 텍스처가 물에 비치게 만들었다.

## sreen space reflection

![alt text](/images/posts/game-capstone/ssr.png)

추가로 더 생동감 있는 물을 표현하기 위해 물의 반사를 흉내는 screen space reflection을 구현했다. 이 기술은 이미 렌더링 된 화면 내에 액체 표면에서 반사된 광선이 어느 지점을 향하는지 계산한다. ray-marching을 사용하는 것은 동일하지만, 타깃은 앞서 렌더링 된 깊이 텍스처가 되고, 광선과 텍스처의 깊이를 비교하면서 어느 표면에 부딪혔는지를 계산한다.

![alt text](/images/posts/game-capstone/water-final.png)

최종 물 렌더 과정