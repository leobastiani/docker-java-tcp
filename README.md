# Docker container is not able to connect to my java application listening on host but it is able to connect to others applications.

Here is my setup which works well with others applications like `netcat`, `npx http-server` or `python -m SimpleHTTPServer`.

Tested on Java 8 and 11. The following screenshots have been taken from this version.

```
java --version

openjdk 11.0.10 2021-01-19
OpenJDK Runtime Environment AdoptOpenJDK (build 11.0.10+9)
OpenJDK 64-Bit Server VM AdoptOpenJDK (build 11.0.10+9, mixed mode)
```

## Steps

```sh
java ./Server.java
```

```sh
docker-compose up -d
ssh root@localhost -p 2222 # password is root
apt update && apt install -y netcat
nc -v docker-host 8080 # or nc -v host.docker.internal 8080
```

## docker-compose.yml

```yml
version: "3.8"
services:
  docker-host:
    image: qoomon/docker-host
    cap_add: ["NET_ADMIN", "NET_RAW"]
    restart: on-failure
    environment:
      - PORTS=8080

  ssh:
    image: rastasheep/ubuntu-sshd
    ports:
      - 2222:22
```

## Server.java

```
import java.io.BufferedReader;
import java.io.IOException;
import java.io.InputStreamReader;
import java.io.PrintStream;
import java.net.InetAddress;
import java.net.ServerSocket;
import java.net.Socket;
import java.net.UnknownHostException;

public class Server implements AutoCloseable {
  private final ServerSocket server;

  public Server(String host, int port, int backlogConnectionQueueLength)
      throws UnknownHostException, IOException {
    InetAddress inetAddress = InetAddress.getByName(host);
    System.out.println("inetAddress: " + inetAddress);
    server = new ServerSocket(port, backlogConnectionQueueLength, inetAddress);
    System.out.println(Thread.currentThread() + " Created Server");
  }

  public static void main(String[] args) {
    try (Server server = new Server("localhost", 8080, 50)) {
      server.start();
    } catch (IOException e) {
      System.out.println(e);
    }
  }

  public void start() {
    System.out.println(Thread.currentThread() + " Server Ready: " + server);
    while (true) {
      acceptAndHandleClient(server);
    }
  }

  private void acceptAndHandleClient(ServerSocket server) {
    System.out.println(Thread.currentThread() + " Waiting for Incoming connections...");
    try (Socket clientSocket = server.accept()) {
      handleNewClient(clientSocket);
    } catch (IOException e) {
      e.printStackTrace();
    }
  }

  private void handleNewClient(Socket clientSocket) throws IOException {
    System.out.println(Thread.currentThread() + " Received Connection from " + clientSocket);
    BufferedReader is = new BufferedReader(new InputStreamReader(clientSocket.getInputStream()));
    PrintStream os = new PrintStream(clientSocket.getOutputStream());
    // echo that data back to the client, except for QUIT.
    String line = null;
    while ((line = is.readLine()) != null) {
      System.out.println(Thread.currentThread() + " Server Got => " + line);
      if (line.equalsIgnoreCase("QUIT")) break;
      else {
        System.out.println(Thread.currentThread() + " Server echoing line back => " + line);
        os.println("Response: " + line);
        os.flush();
      }
    }
    System.out.println(Thread.currentThread() + " Server Closing Connection by Sending => Ok");
    os.println("Ok");
    os.flush();
    is.close();
    os.close();
  }

  public void close() throws IOException {
    server.close();
  }
}
```
