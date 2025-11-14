---
layout: post
title: "pygame 플랫포머 제작"
description: "게임프로그래밍입문 프로젝트"
modified: 2025-11-11
tags: [python]
categories: [project]
---

![screenshot](/images/posts/IWBTPY/screenshot.png)

[[github page]](https://github.com/sim2han/IWBTPY)

### 요약

pygame을 이용해 간단한 충돌, 중력 반전 요소가 있는 플랫포머를 제작했다.

## 목표

수업에서 배운 내용을 바탕으로 게임을 제작하는 프로젝트를 진행했다. 

이때, 벡터, 충돌 내용을 활용하기 적절한 게임을 구상하던 중, 중력이 변하는 플랫포머를 떠올렸고, I wanna be the guy의 팬게임을 제작해보기로 했다.

## 게임 디자인

1.	캐릭터는 Z로 점프하고, 이단 점프가 가능하다. Z를 누른 시간에 따라 높이가 바뀐다.
2.	캐릭터는 X로 총을 쏠 수 있고, 세이브 블록을 쏘면 현재 위치가 저장된다.
3.	캐릭터는 가시에 닿으면 사망한다.
4.	R을 누르면 스폰 위치에서 캐릭터가 재시작한다.
5.	중력 구역에 들어가면, 해당 구역에 설정된 방향으로 중력이 바뀐다.
6.	목적지에 도착하면 다음 레벨로 바뀐다.

## 게임 구조

main() -> 게임 초기화 -> 루프 시작 -> 입력 처리 -> 캐릭터 이동 -> 충돌 처리 -> 그래픽 게임을 구성하는 요소는 객체로 구성된다.

Character-캐릭터, Spike-가시, Block-벽, GravityBox-중력구역, SaveBlock-세이브, EndBox-목적지

레벨을 실제 구성하는 인스턴스는 레벨이 로드될 때 전역변수인 blocks, spikes등의 배열에 저장된 다. 캐릭터, 스폰 위치, 목적지 같은 하나만 있어도 되는 물체도 전역변수로 존재한다.

main이 시작되면 pygame을 초기화하고 게임에 사용될 레벨 데이터와 사운드를 읽는다. 읽은 레 벨은 levels에 추가되고 levels[0] 시작 레벨을 로드한다. 루프에서 사용될 변수도 설정한다.

루프가 시작되면 먼저 키 이벤트를 처리한다. 여기에는 점프, 재시작에 대한 키 이벤트가 있다. 그 뒤 플레이어를 움직인다. 플레이어의 좌우 운동은 고정된 속도만큼 이동시키고, 점프는 속도를 설정해 움직인다. 카메라 조작과 좌우 이동 애니메이션도 여기서 처리한다.

캐릭터가 움직인 뒤 물리를 처리한다. 목적지, 가시, 벽 순서로 플레이어가 충돌했는지 확인하고 그에 따른 처리를 한다. 충돌 검사가 끝나면 플레이어 속도를 조정한다. 플레이어가 공중에 있다 면 위에서 설정한 애니메이션을 공중 애니메이션으로 덮어 씌운다.

처리가 끝나면 그래픽을 그린다. 가시와 벽은 tmx 레벨 데이터를 사용해 그리고, 총알, 중력 구 역, 세이브 블럭, 목적지, 캐릭터 순서로 그린다. 대부분의 객체는 자신을 그리는 def draw(self, screen)을 가지고 있어서 호출하면 된다. 마지막으로 사망 횟수를 그리고 화면을 업데이트한다.

### 충돌 처리

![collision](/images/posts/IWBTPY/collision.png)

````
for spike in spikes:
    for line in spike.lines():
        ch_pos = Vector(character.pos.x, character.pos.y) v1_to_ch = ch_pos - line.v1
        v1_to_v2 = line.v2 - line.v1

        line_len = line.len()
        dot = v1_to_ch.dot(v1_to_v2) proj_len = dot / line_len

        if dot < 0 or proj_len > line_len:
            continue

        dist = sqrt(v1_to_ch.power_magnitude() - proj_len ** 2)
        if dist < character_radius:
            character.die()
            break
````

플레이어는 공으로 표현되고, 가시와 벽은 그것을 둘러 싸는 선의 집합이다. 만약 플레이어와 사 각형 블록을 판정하고 싶다면, 블록을 이루는 4개의 선을 각각 계산할 수 있다.

위는 spike의 경우로 spike.lines()는 자신을 이루는 삼각형의 선을 반환한다. 선과 공의 충돌 판정 을 진행한다. 간단히 v1-v2를 잇는 선 L과 구 B의 충돌은, v1-B를 v1-v2에 프로젝션 한 뒤, 투영된 곳이 선 내부에 있는가와 선까지 거리가 반지름 보다 작은가를 확인한다. 만약 그렇다면 충돌이 된 것이다.

### 레벨

![level](/images/posts/IWBTPY/level.png)

Tiled를 사용해 대부분의 레벨 디자인이 가능하게 만들었다. Tiled에서 레이어에 레벨을 그리면, 코드 수정 없이 레벨을 변경할 수 있다.

````
def __init__(self, name, spawn:Vector, end_point:Vector):
    self.map = load_pygame(name)
    self.map_background = self.map.get_layer_by_name("Background")
    self.map_static = self.map.get_layer_by_name("Static")
    self.map_spike_up = self.map.get_layer_by_name("Spike_up")
    # ……

def load(self):
    blocks.clear()
    spikes.clear()
    for x, y, pic in self.map_static.tiles():
        blocks.append(Block(x, -y))
    for x, y, pic in self.map_spike_up.tiles():
        spikes.append(Spike(x, -y, 0))
    # ……

levels.append(Level("Tiled/level0.tmx", Vector(2, -17), Vector(3, -9)))
````
레벨을 생성하면 tmx파일을 읽고 각 레이어를 분리해 저장한다. 레벨을 로드하면 tmx파일 내부에 각각의 레이어를 읽고 실제 게임에 사용될 오브젝트에 대응해 넣는다.

### 애니메이션

````
class Animation:
    def __init__(self, path, delay=1, size=32):
        self.delay = delay
        self.delay_count = 0
        self.idx = 0
        self.sprites = []
        sheet = pygame.image.load(path)
        for i in range(0, sheet.get_width() // size):
            self.sprites.append(sheet.subsurface(pygame.Rect(size * i, 0, size, size)))

    def add_sprite(self, path):
        sprite = pygame.image.load(path) self.sprites.append(sprite)

    def first(self):
        self.idx = 0
        return self.sprites[self.idx]

    def next(self): self.delay_count += 1
        if self.delay_count == self.delay:
            self.delay_count = 0
            self.idx += 1
            if self.idx == len(self.sprites):
                self.idx = 0
        return self.sprites[self.idx]

    def index(self, i):
        return self.sprites[i]
````

![animation](/images/posts/IWBTPY/animation.png)

class Animation은 이미지를 size*size 크기의 sprite로 쪼개서 저장한다. next()를 호출하면 객체는 다음 이미지에 해당하는 것을 반환한다. 하지만 프레임마다 애니메이션이 바뀌면 너무 빠르니, delay를 두어 해당 숫자만큼 next가 호출되었을 때 다음 이미지가 나오게 한다.

## 사용환경

python 3.11.4

pygame 2.5.2

pytmx 3.32
