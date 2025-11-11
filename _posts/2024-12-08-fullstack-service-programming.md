---
layout: post
title: "rust, flutter로 오목 앱, 서버 개발"
description: "풀스택서비스프로그래밍 프로젝트"
tags: [class project, rust, dart, flutter]
categories: [project]
---

![main](/images/posts/five-in-row/main.png)

[[github server]](https://github.com/sim2han/five-in-row-project)

[[github client]](https://github.com/sim2han/five-in-row-client)

### 요약

Figma로 앱을 디자인하고, rust, tokio, dart, flutter를 이용해 멀티 플레이가 가능한 오목 게임 서버와 클라이언트를 제작했다.

이 과정에서 서버와 클라이언트의 통신을 위해 RESTful API, WebSocket, 비동기 프로그래밍를 사용해 볼 수 있었다.

## 개요

수업 프로젝트로 flutter를 이용한 앱, 자유 언어 서버로 서비스를  만드는 과제가 나왔다.

평소 재밌게 사용하던 rust를 사용해 간단한 멀티플레이 게임을 제작하기로 했다.

## 기능

![app](/images/posts/five-in-row/app.png)

1. 아이디와 패스워드로 로그인을 한다.
2. 플레이어는 다른 플레이어와 오목 게임을 할 수 있다.
3. 계정에 따라 이전 게임 다시보기를 제공한다.

## 제작

### figma

![figma](/images/posts/five-in-row/figma.png)

figma를 이용해 메인 페이지, 로그인, 게임, 다시보기 각각에 대한 레이아웃을 디자인했다.

### flutter 클라이언트

![client](/images/posts/five-in-row/client.png)

Engine 싱글톤이 전체 앱의 정보를 가지고, 서버 통신이나 화면 갱신이 필요할 때, ValueNotifier로 화면을 구성한다.

### 서버

![server](/images/posts/five-in-row/server.png)

서버는 내부에서 크게 네 종류의 비동기 루프를 돌며 클라이언트와 메시지를 송수신한다. 계정, 게임 등의 데이터는 Mutex로 나누어 사용하고, 비동기 루프 간 통신은 tokio의 mpsc, mpmc 채널을 사용한다.

### Restful

![restful](/images/posts/five-in-row/restful.png)

로그인, 조회 등 실시간성이 필요 없는 작업은 restful api를 활용한다. Dart와 Rust간에 호환이 되도록 class, struct를 작성하고 json으로 정보를 주고 받는다.

### WebSocket

![websocket](/images/posts/five-in-row/websocket.png)

Websocket은 게임 등록, 시작, 자신과 상대의 플레이를 전달하기 위해 사용한다. 이때 메시지를 구분하기 위해 내부에 구분하는 string을 사용했다.

### 비동기 프로그래밍

개발에서 가장 많은 시간을 사용한 부분이 websocket 통신이다. 게임 중에는 실시간 양방향 통신이 가능해야 했기 때문에 비동기 처리를 해야했다. 많은 시행착오 후 websocket을 캡슐화해서 객체로 만들고, mpmc 채널을 창구로 사용하기로 했다.

![socket](/images/posts/five-in-row/socket.png)

Rust에서 WebSocket은 다음처럼 Socket 구조체에 담기고, 사용자는 필요할 때, Sender와 Receiver를 복사해 가져간다.

![loop1](/images/posts/five-in-row/loop1.png)

따라서 Socket 구조체는 Client에서 들어오는 메시지를 감지하는 루프, 게임에서 들어오는 메시지를 감지하는 루프 2개의 비동기 루프를 사용한다. 게임 로직도 client - socket에서 들어오는 메시지를 처리하기 위한 루프를 돈다.

![loop2](/images/posts/five-in-row/loop2.png)

client가 2개이니, 게임 로직도 2개의 루프를 사용한다. 이때, 게임의 동작을 한 곳에서 실행시키기 위해, 두 루프에서 다시 메시지를 한 루프로 보내는 작업을 진행했다.

### 사용한 라이브러리

![libs](/images/posts/five-in-row/libs.png)

나중에 알게 되었지만, Hyper가 조금 저수준 라이브러리라고 들었다..
