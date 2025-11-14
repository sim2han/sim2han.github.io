---
layout: post
title: "gRPC 실습"
description: "풀스택서비스네트워킹 실습"
tags: [c++]
categories: [project]
---

<figure>
	<img src="/images/posts/gRPC/bidirectional-streaming-demo.jpg" alt="">
	<figcaption>gRPC 양방향 데이터 스트리밍 데모</figcaption>
</figure>

[[github]](https://github.com/sim2han/fullstacknetwork-project)

### 요약

C++와 gRPC를 통해 서버, 클라이언트 간 통신, 스트리밍하는 실습을 진행했다.

### gRPC

RPC는 Remote Procedure Call의 약자로 다른 머신의 함수를 원격으로 호출하고 결과를 받는 등의 작업을 수행한다. gRPC는 거기서 더 나아가 데이터를 주고 받고, 스트리밍하는 기능도 수행한다.

gRPC를 사용하기 위해서는 전송할 데이터를 Protobuf로 직렬화해야한다. Protobuf는 google에서 만든, 언어와 무관하게 데이터를 직렬화해 주고 받을 수 있게 만든 도구이다.그렇기에  Protobuf 자체는 언어와 독립적인 자체적인 형식을 지닌다.

````
syntax = "proto3";

package bidirectional;

service Bidirectional {
    // A Bidirectional streaming RPC.
    //
    // Acccepts a stream of Message sent while a route is being traversed,
    rpc GetServerResponse(stream Message) returns (stream Message) {}
}

message Message {
    string message = 1;
}
````

이런 형식의 .proto 파일을 작성 후, Protobuf 도구를 사용하면 자동으로 언어에 맞는 코드를 출력한다. C++의 경우 이렇게 나온 코드를 프로젝트에 포함해 빌드하면 된다.

````
// 클라이언트 코드
#include <grpcpp/grpcpp.h>
#include <grpcpp/client_context.h>

#include <iostream>
#include <memory>
#include <string>
#include <array>

#include "src/bidirectional.pb.h"
#include "src/bidirectional.grpc.pb.h"

using grpc::Channel;
using grpc::ClientContext;
using grpc::Status;
using grpc::ClientReaderWriter;
using namespace bidirectional;

Message make_message(const char* str) {
  Message msg;
  msg.set_message(str);
  return msg;
}

int main() {
  std::unique_ptr<Bidirectional::Stub> stub(Bidirectional::NewStub(
    grpc::CreateChannel("localhost:50051", grpc::InsecureChannelCredentials())));
  ClientContext context;
  std::unique_ptr<ClientReaderWriter<Message, Message>> stream(
    stub->GetServerResponse(&context));
  
  std::array<Message, 5> messages = {
    make_message("message #1"),
    make_message("message #2"),
    make_message("message #3"),
    make_message("message #4"),
    make_message("message #5"),
  };

  for (auto& message : messages) {
    std::cout << absl::StrFormat("[client to server] %s", message.message()) << std::endl;
    stream->Write(message);
  }
  stream->WritesDone();
  Message response;
  while (stream->Read(&response)) {
    std::cout << absl::StrFormat("[server to client] %s", response.message()) << std::endl;
  }
  stream->Finish();
}
````

위 코드는 클라이언트가 서버와 상호 스트리밍하는 예제로 먼저 stub와 context를 만들고 서버와 stream을 생성한다. 그리고 Message를 생성하고 서버로 Write한뒤 서버에서의 값을 Read한다. 여기서 사용된 Message 클래스는 위의 proto 파일에서 자동으로 생성된 클래스다. 이렇듯 gRPC를 사용하면 굉장히 간단한 방식으로 서로 다른 언어 간 데이터를 주고 받을 수 있다.