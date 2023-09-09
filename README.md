# Greeter 

Greeter service is a "hello world" service in the grpc world. The client and server source code can be found at: https://github.com/grpc/grpc-go

The assembled server in docker: https://github.com/bavix/greeter-server.
Let's launch:
```bash
docker run -p 4770:4770 bavix/greeter-server -port=4770
```

The assembled client in docker: https://github.com/bavix/greeter-client.
Let's launch:
```bash
docker run --network=host bavix/greeter-client -addr=127.0.0.1:4770 -name=gripmock
```

You should see the result of the client's work:
```bash
2023/09/09 06:00:11 Greeting: Hello gripmock
```

Terminate the service, let's run the same in docker-compose:
```bash
docker compose up
```

You should see stdout something like this:
```bash
 ✔ Container greeter-gripmock-server-1   Created                                                                                                                             0.0s 
 ✔ Container greeter-gripmock-greeter-1  Created                                                                                                                             0.0s 
Attaching to greeter-gripmock-greeter-1, greeter-gripmock-server-1
greeter-gripmock-server-1   | 2023/09/09 06:02:10 server listening at [::]:4770
greeter-gripmock-greeter-1  | 2023/09/09 06:02:10 Greeting: Hello gripmock
greeter-gripmock-server-1   | 2023/09/09 06:02:10 Received: gripmock
greeter-gripmock-greeter-1 exited with code 0
```

The server started successfully, and the client started and sent a request to the server. Afterwards the client completed its work with code 0.

Docker compose has the ability to interrupt the operation of containers if some service has stopped working. Typically, this mechanism is used in integration tests.
```bash
docker compose up --abort-on-container-exit --exit-code-from greeter
```

Now the result is like this:
```bash
 ✔ Container greeter-gripmock-server-1   Created                                                                                                                             0.0s 
 ✔ Container greeter-gripmock-greeter-1  Created                                                                                                                             0.0s 
Attaching to greeter-gripmock-greeter-1, greeter-gripmock-server-1
greeter-gripmock-server-1   | 2023/09/09 06:05:53 server listening at [::]:4770
greeter-gripmock-server-1   | 2023/09/09 06:05:53 Received: gripmock
greeter-gripmock-greeter-1  | 2023/09/09 06:05:53 Greeting: Hello gripmock
greeter-gripmock-greeter-1 exited with code 0
Aborting on container exit...
[+] Stopping 2/2
 ✔ Container greeter-gripmock-greeter-1  Stopped                                                                                                                             0.0s 
 ✔ Container greeter-gripmock-server-1   Stopped     
 ```

Using the gripmock service, we simulate the operation of a greeter server.

Create a stub `stubs/greeter.yaml`:
```yaml
- service: Greeter
  method: SayHello
  input:
    equals:
      name: gripmock
  output:
    data:
      message: Hello, GripMock
```

Download the proto-file of the greeter service:
```bash
curl https://raw.githubusercontent.com/grpc/grpc-go/master/examples/helloworld/helloworld/helloworld.proto \
  --create-dirs -o api/helloworld.proto
```

Let's run gripmock:
```bash
docker run \
  -p 4770:4770 \
  -p 4771:4771 \
  -v ./api:/proto:ro \
  -v ./stubs:/stubs:ro \
  bavix/gripmock -stub=/stubs /proto/helloworld.proto
```

Let's launch the client:
```bash
docker run --network=host bavix/greeter-client -addr=127.0.0.1:4770 -name=gripmock
```

You should see the result of the client's work:
```bash
2023/09/09 06:16:10 Greeting: Hello, GripMock
```

In the docker-compose file it will look like this:
```yaml
version: '3.8'

services:
  greeter:
    image: bavix/greeter-client:1
    depends_on:
      gripmock:
        condition: service_healthy
    command:
      - -addr=gripmock:4770
      - -name=gripmock
    networks:
      - bx-greeter

  gripmock:
    image: bavix/gripmock:2
    ports:
      - 4770:4770
      - 4771:4771
    healthcheck:
      test: "curl --connect-timeout 1 --silent --show-error --fail http://gripmock:4771/api/health/readiness"
      timeout: 1s
      interval: 1s
      start_period: 1s
      retries: 10
    command:
      - -stub=/stubs
      - /proto/helloworld.proto
    volumes:
      - ./stubs:/stubs:ro
      - ./api:/proto:ro
    networks:
      - bx-greeter

networks:
  bx-greeter:
```

Running an integration test:
```bash
docker compose -f docker-compose.test.yaml up --abort-on-container-exit --exit-code-from greeter
```

Result of running the integration test:
```bash
 ✔ Container greeter-gripmock-gripmock-1  Created                                                                                                                            0.0s 
 ✔ Container greeter-gripmock-greeter-1   Recreated                                                                                                                          0.0s 
Attaching to greeter-gripmock-greeter-1, greeter-gripmock-gripmock-1
greeter-gripmock-gripmock-1  | Starting GripMock
greeter-gripmock-gripmock-1  | Serving stub admin on http://:4771
greeter-gripmock-gripmock-1  | grpc server pid: 40
greeter-gripmock-gripmock-1  | Serving gRPC on tcp://:4770
greeter-gripmock-gripmock-1  | 172.20.0.2 - - [09/Sep/2023:07:45:05 +0000] "GET /api/health/readiness HTTP/1.1" 200 57
greeter-gripmock-gripmock-1  | 127.0.0.1 - - [09/Sep/2023:07:45:06 +0000] "POST /api/stubs/search HTTP/1.1" 200 50
greeter-gripmock-greeter-1   | 2023/09/09 07:45:06 Greeting: Hello, GripMock
greeter-gripmock-greeter-1 exited with code 0
Aborting on container exit...
[+] Stopping 2/2
 ✔ Container greeter-gripmock-greeter-1   Stopped                                                                                                                            0.0s 
 ✔ Container greeter-gripmock-gripmock-1  Stopped    
 ```

In the pipeline, you can operate with the code returned from docker compose and, if a non-zero value is received, mark the test as failed.

---
Supported by

[![Supported by JetBrains](https://cdn.rawgit.com/bavix/development-through/46475b4b/jetbrains.svg)](https://www.jetbrains.com/)
