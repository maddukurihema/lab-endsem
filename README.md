# Round-Robin Load Balancing at an API Gateway (Spring Boot MVC)

## Objective
Distribute incoming requests across three Review Service instances using a
Round-Robin algorithm. If an instance cannot be reached, the gateway skips it
temporarily and tries another instance.

## Technology
- Java 17
- Spring Boot 3.4.5
- Spring Boot MVC (`spring-boot-starter-web`)
- Maven
- REST APIs

## Project structure

```text
round-robin-api-gateway/
├── pom.xml
├── review-service/
│   ├── pom.xml
│   └── src/main/java/com/example/review/
│       ├── ReviewServiceApplication.java
│       └── ReviewController.java
└── api-gateway/
    ├── pom.xml
    └── src/main/java/com/example/gateway/
        ├── ApiGatewayApplication.java
        ├── GatewayController.java
        ├── RoundRobinLoadBalancer.java
        ├── ServiceInstance.java
        └── AllInstancesUnavailableException.java
```

## Step 1: Build the project

Open a terminal in the root project directory:

```bash
mvn clean package
```

## Step 2: Start three Review Service instances

Open three separate terminals in the project root and run:

Terminal 1:
```bash
mvn -pl review-service spring-boot:run -Dspring-boot.run.arguments=--server.port=8081
```

Terminal 2:
```bash
mvn -pl review-service spring-boot:run -Dspring-boot.run.arguments=--server.port=8082
```

Terminal 3:
```bash
mvn -pl review-service spring-boot:run -Dspring-boot.run.arguments=--server.port=8083
```

Each process runs the same Review Service application on a different port.

## Step 3: Start the API Gateway

Open another terminal:

```bash
mvn -pl api-gateway spring-boot:run
```

The gateway starts on port `8080`.

## Step 4: Test Round-Robin distribution

Send repeated GET requests to:

```text
http://localhost:8080/api/reviews
```

Using curl:

```bash
curl http://localhost:8080/api/reviews
curl http://localhost:8080/api/reviews
curl http://localhost:8080/api/reviews
curl http://localhost:8080/api/reviews
curl http://localhost:8080/api/reviews
```

Expected instance sequence while all three are running:

```text
8081
8082
8083
8081
8082
```

The response includes `instancePort`, which identifies the instance that handled
the request. The sequence may start at a different port if requests were already
sent.

## Step 5: Test instance failure

1. Stop the Review Service running on port `8082` (press Ctrl+C in its terminal).
2. Continue sending requests to `http://localhost:8080/api/reviews`.
3. The gateway detects the connection failure, marks that instance unavailable
   for 10 seconds, and tries another instance.
4. Requests continue to be served by ports `8081` and `8083`.
5. Restart the stopped instance. After its cooldown, the gateway can select it
   again and it rejoins rotation.

If all three instances are unreachable, the gateway returns HTTP `503 Service
Unavailable`.

## How the algorithm works

1. An `AtomicInteger` stores the next rotation position safely across threads.
2. For each request, the gateway selects the next service instance.
3. After the last instance, selection wraps to the first instance.
4. If a connection failure occurs, that instance is put in a 10-second cooldown.
5. The gateway tries another available instance.
6. If all instances fail, it returns HTTP 503.

Example:

| Request | Selected instance |
|---|---|
| 1 | 8081 |
| 2 | 8082 |
| 3 | 8083 |
| 4 | 8081 |
| 5 | 8082 |

## Important notes
- This is a teaching/demo implementation of Round-Robin load balancing using
  Spring Boot MVC. It uses a configured list of service URLs rather than a
  service registry such as Eureka.
- The failure handling detects connection-level failures. Application-level
  HTTP errors such as 500 responses are returned by the selected service and
  are not automatically treated as a dead instance.
- For production, add health checks, timeouts, metrics, centralized
  configuration, and a service registry or platform-native load balancer.
