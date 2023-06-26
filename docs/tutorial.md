# 🌟 Netty Socket.io Tutorial

<p>
  <img src= "https://user-images.githubusercontent.com/82090641/167614453-fe0777e1-0250-479b-a24d-b2532f2c5cae.jpg" width="340">
  <img src= "https://user-images.githubusercontent.com/82090641/167600725-6ff6fb4e-2af2-4e42-9ffa-12fede95ce0f.png" width="250">
</p>

#### Web에서 Spring boot로 Socket 통신을 하겠다고 한다면 Spring 기반 Socket인 SockJS를 사용할 생각이었다.

#### 하지만 Spring 기반인 SockJS와는 다르게 Node.js 기반인 Socket io를 자주 사용한다고 한다...

#### 그 이유로 Socket io는 Node.js 기반이기 때문에 모든 코드가 javascript로 작성되어 있다.

#### 서버, 클라이언트 간의 언어가 같은 언어라면 개발이 쉽고 편하기 때문에 Socket io를 사용하는 것이였다.

#### Spring boot를 사용하는 나는 프로젝트 내에서 Socket 통신을 매우 하고 싶었기 때문에 Socket io를 공부하게 되었다.

------

- ### 커다란 벽

#### 그렇게 Socket io에 관한 자료를 전부 찾아보았다.

#### 하지만 역시 Node.js 기반이라 Spring boot + Socket io에 대한 자료가 매우 부족했다.

#### 그래도 간단한 채팅 프로젝트 자료를 몇 개 찾아서 개발을 해볼 수 있었다.

------

### 1. Dependencies

```groovy
implementation 'com.corundumstudio.socketio:netty-socketio:1.7.19'
```

### 2. Config

#### Host와 Port를 정해 설정해주면 된다.

#### SocketConfig 설정을 통해 다양한 옵션을 선택할 수 있다.

```java
private final SocketIoProperties properties;

@Bean
public SocketIOServer socketIOServer() {
    com.corundumstudio.socketio.Configuration config = new com.corundumstudio.socketio.Configuration();
    SocketConfig socketConfig = new SocketConfig();
    config.setSocketConfig(socketConfig);
    config.setHostname(properties.getHost());
    config.setPort(properties.getPort());
    return new SocketIOServer(config);
}
```

### 3. Start와 Stop

#### 위에서 Bean 설정한 SocketIOServer를 가져와 Connect Event와 Disconnect Event를 만들어 주겠다.

#### 서버가 열리고 닫힐 때만 실행되게끔 @PostConstruct와 @PreDestroy를 사용해 만들어준다.

```java
@PostConstruct
private void start() {
    socketIOServer.addConnectListener(connectListener);
    socketIOServer.addDisconnectListener(disconnectListener);
    socketIOServer.start();
}

@PreDestroy
private void stop() {
    if (socketIOServer != null) {
        socketIOServer.stop();
    }
}
```

- #### Connect & Disconnect

```java
public static final Map<String, SocketIOClient> connectMap = new ConcurrentHashMap<>();

private final ConnectListener connectListener = client -> {
        String username = client.getHandshakeData().getSingleUrlParam("username");
        connectMap.put(username,client);
        client.set("username",username);
};

private final DisconnectListener disconnectListener = client -> {
    String username = client.getHandshakeData().getSingleUrlParam("username");
    connectMap.remove(username);
    client.disconnect();
};
```

#### Event Data의 UrlParameter로 값을 불러와 client를 ConnectMap에 등록해준다.

### 4. NameSpace

<img src= "https://user-images.githubusercontent.com/82090641/167740114-c5f1208c-69a7-4cc9-b6ca-ae7aafb5b5ab.png" width="300">

#### NameSpace란 위 사진과 같은 모형이다.

```java
namespace = server.addNamespace("/chat");
```

#### server.addNamespace로 추가할 수 있고 A Url과 B Url로 각각 연결한 client가 있다면 A와 B는 다른 Namespace에 있는 것이다.

### 5. EventListener

#### 메인 개발 단계인 Event 부분이다.

```java
@Autowired
public ChatController(SocketIOServer server) {
    namespace = server.addNamespace("/chat");
    namespace.addEventListener("send", ChatMessage.class, onMessage());
    namespace.addEventListener("userJoin", EventMessage.class, onJoin());
    namespace.addEventListener("userTyping", EventMessage.class, onTyping());
    namespace.addEventListener("userStopTyping", EventMessage.class, onStopTyping());
}

```

#### namespace에 addEventListener로 Event를 설정해 주었다.

- #### 첫번째 Args는 어떤 이벤트인가?

- #### 두번째 Args는 어떤 Data를 받을 것인가?

- #### 세번째 Args는 Event 구현이다.

```java
private DataListener<ChatMessage> onMessage() {
    return (client, data, ackSender) -> {
        namespace.getBroadcastOperations().sendEvent("newMessage", client, data);
        messageRepository.save(data);
    };
}
```

#### Event가 호출되면 client와 data, ackSender가 들어오게 되는데

#### data는 위 EventListener에서 설정한 Object 형태로 들어오게 된다.

#### namespace의 Broadcast를 통해 연결된 client에게 Event를 보내는 형태이다.

### 6. SendEvent

#### Event를 보내는 방식은 여러 방식이 있다.

- #### Room을 특정해 보내기

- #### Client를 특정해 보내기

- #### Namespace 전체에 보내기

#### 여기에 보낸 Client이외의 다른 Client에게만 보낼 건지 선택할 수 있다.

- #### BoradcastOperations

```java
package com.corundumstudio.socketio;

import com.corundumstudio.socketio.protocol.Packet;

import java.util.Collection;

/**
 * broadcast interface
 *
 */
public interface BroadcastOperations extends ClientOperations {

    Collection<SocketIOClient> getClients();

    <T> void send(Packet packet, BroadcastAckCallback<T> ackCallback);

    void sendEvent(String name, SocketIOClient excludedClient, Object... data);

    <T> void sendEvent(String name, Object data, BroadcastAckCallback<T> ackCallback);

    <T> void sendEvent(String name, Object data, SocketIOClient excludedClient, BroadcastAckCallback<T> ackCallback);

}
```

#### excludeClient 부분에 파라미터로 Client를 넣어주면 excludeClient이외의 Client에만 전송이 된다.

- #### ClientOperations

```java
package com.corundumstudio.socketio;

import com.corundumstudio.socketio.protocol.Packet;

/**
 * Available client operations
 *
 */
public interface ClientOperations {

    /**
     * Send custom packet.
     * But {@link ClientOperations#sendEvent} method
     * usage is enough for most cases.
     *
     * @param packet - packet to send
     */
    void send(Packet packet);

    /**
     * Disconnect client
     *
     */
    void disconnect();

    /**
     * Send event
     *
     * @param name - event name
     * @param data - event data
     */
    void sendEvent(String name, Object ... data);

}
```

#### 간단하게 Spring boot + Netty Socket io에 대해 알아보았다 :)
