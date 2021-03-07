---
title: "gRPC Server and Client examples with Typescript (static generated code)"
date: 2021-03-04T12:53:37+01:00
draft: false
tags: [grpc, backend, protobuf, typescript, nodejs]
---

## gRPC and Protobuf

**gRPC** is a framework for RPC (Remote Procedure Call) focused on performance and environments compatibility. Any information sent with this framework is serialised with **protocol buffers**. If you don't know what this means I suggest to have a look at the official websites for [gRPC](https://grpc.io/) and [protocol buffers](https://developers.google.com/protocol-buffers).

### Reflection vs static code

Thanks to **protocol buffers,** messages sent through gRPC services are defined into a schema (.proto file). This means we can generate the code to scaffold both server and client as we know the shape of the information flowing through the system and which are the procedures exposed by any service.

For interpreted languages, like JavaScript, there are two approaches for the generated code by protocol buffers: reflection and static code.

1. **reflection**: in this case we generate the necessary functions and classes runtime when your code gets executed. Once your service stops, this code doesn't exist anymore, it was only in memory. The advantage of this option is the speed, you just need the `.proto` file in the filesystem of your program and execute the right lines of code.
2. **static code**: in this case, we generate actual code into files. The generation tool is the protocol buffers' command `protoc` which can generate code for any of the main languages. Once the code it's generated you use start to use it as any other code or pack it and distribute it. The advantage of this approach is the decoupling from the `.proto` file which is not code, the fact we can distribute the code as a library and the control we have on the generated code as we can version it each time we run `protoc` and keep an eye on it.

### Static code

There are several reasons why I prefer the static code vs reflection but the biggest one here is generating TS types declaration along with the JS files.
This is possible to a `protoc` plugin called [grpc_tools_node_protoc_ts](https://github.com/agreatfool/grpc_tools_node_protoc_ts).

## Let's do it

### Proto definition

Let's say we have a proto at this path `./protos/helloworld.proto` which has the following content.

```proto
syntax = "proto3";

package helloworld;

// The greeting service definition.
service Greeter {
  // Sends a greeting
  rpc SayHello (HelloRequest) returns (HelloReply) {}
}

// The request message containing the user's name.
message HelloRequest {
  string name = 1;
}

// The response message containing the greetings
message HelloReply {
  string message = 1;
}
```

Source: [helloworld.proto](https://github.com/grpc/grpc/blob/master/examples/protos/helloworld.proto)

### Generating the code

As I just said this a job for `protoc` and the only input needed it's our `.proto` file.

```bash
# --unsafe-perm needed in docker

npm install -g --unsafe-perm grpc-tools@1.10.0
npm install grpc_tools_node_protoc_ts@5.0.0 --save-dev

mkdir my_generated_code

grpc_tools_node_protoc \
--plugin=protoc-gen-ts=./node_modules/.bin/protoc-gen-ts \
--ts_out=grpc_js:./my_generated_code \
--grpc_out=grpc_js:./my_generated_code \
-I ./protos \
./protos/helloworld.proto
```

If something it's not clear how to use these tools please check the relative documentations for [grpc-tools](https://github.com/grpc/grpc-node/tree/master/packages/grpc-tools) and its plugin [grpc_tools_node_protoc_ts](https://github.com/agreatfool/grpc_tools_node_protoc_ts).

### The generated code

Now, you'll have a bunch of `.js` and `.d.ts` files mirroring your protos structure (yes, you can generate code for more than one proto file).

- `helloworld_pb.js`
- `helloworld_pb.d.ts`
- `helloworld_grpc_pb.js`
- `helloworld_grpc_pb.d.ts`

The **_pb.js** modules contain all the classes to serialise and parse any **message** defined in the proto file. In this example, `HelloRequest` and `HelloReply`.

The **_grcp_pb.js** modules contain all the classes to create a server or a client instance to expose or interact with the **service** defined in the proto file. In this example `Greeter`.

This is the only code needed to create a server and a client along with two dependencies: [@grpc/grpc-js](https://www.npmjs.com/package/@grpc/grpc-js) and [google-protobuf](https://www.npmjs.com/package/google-protobuf).

This allows us to easily create an **npm** package and share it between projects, particularly valuable for large organisations.

## Implementation

### Server

```typescript
import * as grpc from "@grpc/grpc-js";

import { HelloRequest, HelloReply } from "./my_generated_code/helloworld_pb";
import { GreeterService } from "./my_generated_code/helloworld_grpc_pb";


const sayHello = (
  call: grpc.ServerUnaryCall<HelloRequest, HelloReply>,
  callback: grpc.sendUnaryData<HelloReply>
): void => {
  const reply = new HelloReply();
  reply.setMessage(`Hello ${call.request.getName()}`);
  callback(null, reply);
}

var server = new grpc.Server();
server.addService(GreeterService, { sayHello });

server.bindAsync('0.0.0.0:50051', grpc.ServerCredentials.createInsecure(), () => {
  server.start();
});
```

### Client

```typescript
import * as grpc from "@grpc/grpc-js";

import { HelloRequest, HelloReply } from "./my_generated_code/helloworld_pb";
import { GreeterClient } from "./my_generated_code/helloworld_grpc_pb";

const client = new services.GreeterClient(
  "localhost:50051",
  grpc.credentials.createInsecure()
);

const request = new HelloRequest();
request.setName("world");

client.sayHello(request, (error, response) => {
  if(!error) {
    console.info("Greeting:", response.getMessage());
  } else {
    console.error("Error:", error.message); 
  }
});
```

## Conclusion

This example covers a basic scenario for a server and a client in typescript. It won't be hard thanks to types to find how to change channels options and other configs. I hope it helped especially because there a lack of official resources on how to use **static generated code**, in particular with **typescript**.
