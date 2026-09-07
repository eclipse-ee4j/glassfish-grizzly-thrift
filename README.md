# Grizzly Thrift

The Grizzly Thrift is a Java server/client library that integrates the transport layer of [Apache Thrift (RPC framework)](https://thrift.apache.org/) with Grizzly.

## Key Functions

This library provides two main components.

### Server Library
It allows you to configure high-performance, high-capacity thrift servers. It replaces the transport layer in the basic thrift server configuration with the Grizzly NIO framework. Additionally, it easily supports thrift over HTTP by utilizing the Grizzly HTTP Server.

### Client Library
It provides a client library with built-in features for connection management and failover/failback in large-scale cluster thrift server environments. This also supports both thrift clients integrated with Grizzly and thrift clients over HTTP by utilizing the Grizzly HTTP Client Filter.
  - Connection Management:
    - Establishing and maintaining connections to one or more Thrift servers.
  - Round-robin Routing:
    - You can dynamically configure the configuration of servers through [Zookeeper](https://zookeeper.apache.org/).
  - Error and Failure Handling:
    - Managing timeouts and network errors, and sometimes marking servers as dead and redirecting requests to the remaining active servers (failover).
  - Protocol Support:
    - Uses Thrift's protocol layer as is. For convenience, TBinaryProtocol and TCompactProtocol are supported by default.

Both server and client libraries are built on top of [Grizzly NIO](https://github.com/eclipse-ee4j/glassfish-grizzly/).


## Getting Started

### Maven coordinates

```
<dependencies>
    <dependency>
        <groupId>org.glassfish.grizzly</groupId>
        <artifactId>grizzly-thrift</artifactId>
        <version>1.3.16</version>
    </dependency>
</dependencies>
```

### Prerequisites

We have different JDK requirements depending on the branch in use:

- JDK 21+ for master and 1.4.x.
- JDK 1.8+ for 1.3.x.

Apache Maven 3.3.9 or later in order to build and run the tests.

### Installing and running the tests

If building in your local environment:

```
mvn clean install
```

### Example of use

- Grizzly Thrift Server

```java
import org.glassfish.grizzly.filterchain.FilterChainBuilder;
import org.glassfish.grizzly.filterchain.TransportFilter;
import org.glassfish.grizzly.nio.transport.TCPNIOTransport;
import org.glassfish.grizzly.nio.transport.TCPNIOTransportBuilder;
import org.glassfish.grizzly.thrift.ThriftFrameFilter;
import org.glassfish.grizzly.thrift.ThriftServerFilter;

import org.glassfish.grizzly.thrift.CalculatorHandler;
import tutorial.Calculator;

import java.io.IOException;

public class ApplicationServer {

    public static void main(String[] args) {
        // final user-generated.thrift.Processor tprocessor = new user-generated.thrift.Processor(new user-generated.thrift.Handler);
        // ex) CalculatorHandler class is thrift's tutorial code.
        // shared.* and tutorial.*' classes are thrift's generated codes based on shared.thrift and tutorial.thrift files in thrift tutorial.
        final CalculatorHandler handler = new CalculatorHandler();
        final Calculator.Processor tprocessor = new Calculator.Processor(handler);

        final FilterChainBuilder filterChainBuilder = FilterChainBuilder.stateless();
        filterChainBuilder.add(new TransportFilter());
        filterChainBuilder.add(new ThriftFrameFilter());
        filterChainBuilder.add(new ThriftServerFilter(tprocessor));
        final TCPNIOTransport transport = TCPNIOTransportBuilder.newInstance().build();
        transport.setProcessor(filterChainBuilder.build());
        try {
            // sets your server port
            transport.bind(9090);
            transport.start();
            // ...
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        // ...

        // shuts down
        try {
            transport.shutdownNow();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

- Grizzly Thrift Server over HTTP

```java
import org.glassfish.grizzly.http.server.HttpServer;
import org.glassfish.grizzly.http.server.NetworkListener;
import org.glassfish.grizzly.thrift.http.ThriftHttpHandler;

import org.glassfish.grizzly.thrift.CalculatorHandler;
import tutorial.Calculator;

import java.io.IOException;

public class ApplicationServer {

    public static void main(String[] args) {
        // final user-generated.thrift.Processor tprocessor = new user-generated.thrift.Processor(new user-generated.thrift.Handler);
        // ex) CalculatorHandler class is thrift's tutorial code.
        // shared.* and tutorial.*' classes are thrift's generated codes based on shared.thrift and tutorial.thrift files in thrift tutorial.
        final CalculatorHandler handler = new CalculatorHandler();
        final Calculator.Processor tprocessor = new Calculator.Processor(handler);

        final HttpServer httpServer = new HttpServer();
        // sets your http server port
        final NetworkListener listener =
                new NetworkListener("grizzly-thrift-http", NetworkListener.DEFAULT_NETWORK_HOST, 9090);
        httpServer.addListener(listener);
        httpServer.getServerConfiguration().addHttpHandler(new ThriftHttpHandler(tprocessor), "/calculator");
        try {
            httpServer.start();
            // ...
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
        // ...

        // shuts down
        httpServer.shutdownNow();
    }
}
```

- Grizzly Thrift Client

```java
import org.glassfish.grizzly.thrift.client.GrizzlyThriftClient;
import org.glassfish.grizzly.thrift.client.GrizzlyThriftClientManager;
import org.glassfish.grizzly.thrift.client.ThriftClient;
import org.glassfish.grizzly.thrift.client.ThriftClientCallback;

import tutorial.Calculator;

import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.util.Set;

public class Application {

    public static void main(String[] args) {
        // creates a ThriftClientManager
        final GrizzlyThriftClientManager manager = new GrizzlyThriftClientManager.Builder().build();

        // creates a ThriftClientBuilder
        final GrizzlyThriftClient.Builder<Calculator.Client> builder = manager.createThriftClientBuilder("Calculator", new Calculator.Client.Factory());

        // sets initial servers
        final Set<SocketAddress> initServerSet = Set.of(new InetSocketAddress("thriftserver1.example.com", 9091),
                                                        new InetSocketAddress("thriftserver2.example.com", 9092));
        builder.servers(initServerSet);

        // creates the client
        final ThriftClient<Calculator.Client> calculatorThriftClient = builder.build();

        // thrift operations
        try {
            Integer result = calculatorThriftClient.execute(new ThriftClientCallback<Calculator.Client, Integer>() {
                @Override
                public Integer call(Calculator.Client client) throws Exception {
                    return client.add(1, 2);
                }
            });
            // the result is equal to 3
            // ...
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            // shuts down
            manager.shutdown();
        }
    }
}
```

- Grizzly Thrift Client over HTTP

```java
import org.glassfish.grizzly.thrift.client.GrizzlyThriftClient;
import org.glassfish.grizzly.thrift.client.GrizzlyThriftClientManager;
import org.glassfish.grizzly.thrift.client.ThriftClient;
import org.glassfish.grizzly.thrift.client.ThriftClientCallback;

import tutorial.Calculator;

import java.net.InetSocketAddress;
import java.net.SocketAddress;
import java.util.Set;

public class Application {

    public static void main(String[] args) {
        // creates a ThriftClientManager
        final GrizzlyThriftClientManager manager = new GrizzlyThriftClientManager.Builder().build();

        // creates a ThriftClientBuilder
        final GrizzlyThriftClient.Builder<Calculator.Client> builder = manager.createThriftClientBuilder("Calculator", new Calculator.Client.Factory());

        // sets initial servers
        final Set<SocketAddress> initServerSet = Set.of(new InetSocketAddress("thriftserver1.example.com", 9091),
                                                        new InetSocketAddress("thriftserver2.example.com", 9092));
        builder.servers(initServerSet);

        // sets the http uri path
        final String uriPath = "/calculator";
        builder.httpUriPath(uriPath);

        // creates the client
        final ThriftClient<Calculator.Client> calculatorThriftClient = builder.build();

        // thrift operations
        try {
            Integer result = calculatorThriftClient.execute(new ThriftClientCallback<Calculator.Client, Integer>() {
                @Override
                public Integer call(Calculator.Client client) throws Exception {
                    return client.add(1, 2);
                }
            });
            // the result is equal to 3
            // ...
        } catch (Exception e) {
            throw new RuntimeException(e);
        } finally {
            // shuts down
            manager.shutdown();
        }
    }
}
```

Additionally, you can check the example through the test case source below:

- https://github.com/eclipse-ee4j/glassfish-grizzly-thrift/blob/master/src/test/java/org/glassfish/grizzly/thrift/ThriftTutorialTest.java
- https://github.com/eclipse-ee4j/glassfish-grizzly-thrift/blob/master/src/test/java/org/glassfish/grizzly/thrift/client/GrizzlyThriftClientTest.java

### Performance Measurement

See the [benchmark results](https://github.com/eclipse-ee4j/glassfish-grizzly-thrift/wiki/Performance-Measurement).


## License

This project is licensed under the EPL-2.0 - see the [LICENSE.md](https://github.com/eclipse-ee4j/glassfish-grizzly-thrift/blob/master/LICENSE.md) file for details.
