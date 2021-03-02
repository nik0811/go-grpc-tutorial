# go-grpc-tutorial
An introduction to gRPC in Go!

```
Prerequisites
Before you can complete this tutorial, you will have to have the following installed on your machine:

Protocol Buffers v3 installed - this can be done by running go get -u github.com/golang/protobuf/protoc-gen-go
You will have to ensure that $GOPATH/bin is on your environment path so that you can use the protoc tool later on in this tutorial.

gRPC Introduction
So, before we dive in, we first need to understand what gRPC is, how it works and so on.

Definition - gRPC is a modern, open source remote procedure call (RPC) framework that can run anywhere

Remote Procedure Calls are something that we use within distributed systems that allow us to communicate between applications. More specifically, it allows us to expose methods within our application that we want other applications to be able to invoke.

It’s similar to REST API communication in the sense that with it, you are effectively exposing functionality within your app to other apps using a HTTP connection as the communication medium.

Differences between gRPC and REST
Whilst REST and gRPC are somewhat similar, there are some fundamental differences in how they work that you should be aware of.

gRPC utilizes HTTP/2 whereas REST utilizes HTTP 1.1
gRPC utilizes the protocol buffer data format as opposed to the standard JSON data format that is typically used within REST APIs
With gRPC you can utilize HTTP/2 capabilities such as server-side streaming, client-side streaming or even bidirectional-streaming should you wish.
Challenges with gRPC
You should bear in mind that whilst gRPC does allow you to utilize these newer bits of technology, it is more challenging prototyping a gRPC service due to the fact that tools like the Postman HTTP client cannot be used in order to easily interact with your exposed gRPC service.

You do have options that make this possible, but it’s not something that’s readily available natively. There are options to use tools such as envoy to reverse proxy standard JSON requests and transcode them into the right data format but this is an additional dependency that can be tricky to set up for simple projects.

Building a gRPC Server in Go
Let’s start off by defining a really simple gRPC server in Go. Once we have a simple server up and running we can set about creating a gRPC client that will be able to interact with it.

We will start off by writing the logic within our main function to listen on a port for incoming TCP connections:

main.go
package main

import ( 
  "log"
  "net"
)

func main() {
  lis, err := net.Listen("tcp", ":9000")
  if err != nil {
    log.Fatalf("failed to listen: %v", err)
  }
}
Next, we’ll want to import the official gRPC package from golang.org so that we can create a new gRPC server, then register the endpoints we want to expose before serving this over our existing TCP connection that we have defined above:

main.go
package main

import (
	"log"
	"net"

	"google.golang.org/grpc"
)

func main() {

	lis, err := net.Listen("tcp", ":9000")
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	grpcServer := grpc.NewServer()

	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %s", err)
	}
}
This right here is the absolute minimum for a gRPC server written in go. However, right now it doesn’t exactly do much.

Adding Some Functionality
Let’s see how we can start exposing some functionality via our gRPC server so that gRCP clients can interact with our server in a meaningful way.

Let’s start by defining the chat.proto file which will act as our contract:

chat.proto
syntax = "proto3";
package chat;

message Message {
  string body = 1;
}

service ChatService {
  rpc SayHello(Message) returns (Message) {}
}
This .proto file exposes our ChatService which features a solitary SayHello function which can be called by any gRPC client written in any language.

These .proto definitions are typically shared across clients of all shapes and sizes so that they can generate their own code to talk to our gRPC server.

Let’s generate the Go specific gRPC code using the protoc tool:

$ protoc --go_out=plugins=grpc:chat chat.proto
You’ll see this will have generated a chat/chat.pb.go file which will contain generated code for us to easily call within our code. Let’s update our server.go to register our ChatService like so:

server.go
package main

import (
	"fmt"
	"log"
	"net"

	"github.com/tutorialedge/go-grpc-beginners-tutorial/chat"
	"google.golang.org/grpc"
)

func main() {

	fmt.Println("Go gRPC Beginners Tutorial!")

	lis, err := net.Listen("tcp", fmt.Sprintf(":%d", 9000))
	if err != nil {
		log.Fatalf("failed to listen: %v", err)
	}

	s := chat.Server{}

	grpcServer := grpc.NewServer()

	chat.RegisterChatServiceServer(grpcServer, &s)

	if err := grpcServer.Serve(lis); err != nil {
		log.Fatalf("failed to serve: %s", err)
	}
}

We are then going to have to define the SayHello method which will take in a Message, read the Body of the message and then return a Message of it’s own:

chat/chat.go
package chat

import (
	"log"

	"golang.org/x/net/context"
)

type Server struct {
}

func (s *Server) SayHello(ctx context.Context, in *Message) (*Message, error) {
	log.Printf("Receive message body from client: %s", in.Body)
	return &Message{Body: "Hello From the Server!"}, nil
}

If we wanted to define more advanced functionality for our gRPC server then we could do so by defining a new method built off our Server struct and then add the name of that function to our chat.proto file so that our application can expose this as something that other gRPC clients can hit.

With these final changes in place, let’s try running our server:

$ go run server.go
Go gRPC Beginners Tutorial!
Awesome! We now have a brand spanking, shiny new gRPC server up and running on localhost:9000 on our machine!

Building a gRPC Client in Go
Now that we have our server up and running, let’s take a look at how we can build up a simple client that will be able to interact with it.

client.go
package main

import (
	"log"

	"golang.org/x/net/context"
	"google.golang.org/grpc"

	"github.com/tutorialedge/go-grpc-beginners-tutorial/chat"
)

func main() {

	var conn *grpc.ClientConn
	conn, err := grpc.Dial(":9000", grpc.WithInsecure())
	if err != nil {
		log.Fatalf("did not connect: %s", err)
	}
	defer conn.Close()

	c := chat.NewChatServiceClient(conn)

	response, err := c.SayHello(context.Background(), &chat.Message{Body: "Hello From Client!"})
	if err != nil {
		log.Fatalf("Error when calling SayHello: %s", err)
	}
	log.Printf("Response from server: %s", response.Body)

}

When we go to run this, we should see that our client gets a very nice Hello message back from the server like so:

$ go run client.go
2020/04/30 20:10:09 Response from server: Hello From the Server!
Awesome, we have successfully created a very simple gRPC client that now talks to our new gRPC server!

```
