<h3 align="center">A springboot websocket component written using netty。</h3>
<p align="center">
    <a href="https://github.com/JDK-Plus/cli-plus/blob/master/LICENSE"><img src="https://img.shields.io/github/license/JDK-Plus/cli-plus.svg" /></a>
    <a href="https://github.com/JDK-Plus/cli-plus/releases"><img src="https://img.shields.io/github/release/JDK-Plus/cli-plus.svg" /></a>
    <a href="https://github.com/JDK-Plus/cli-plus/stargazers"><img src="https://img.shields.io/github/stars/JDK-Plus/cli-plus.svg" /></a>
    <a href="https://github.com/JDK-Plus/cli-plus/network/members"><img src="https://img.shields.io/github/forks/JDK-Plus/cli-plus.svg" /></a>
</p>

- [中文文档](README.md)

## Maven Dependencies

```xml
<dependency>
    <groupId>plus.jdk</groupId>
    <artifactId>spring-boot-starter-websocket</artifactId>
    <version>1.0.1</version>
</dependency>
```

## Configuration

```
plus.jdk.websocket.enabled=true
plus.jdk.websocket.port=10001
plus.jdk.websocket.cors-allow-credentials=true
```

## Example of use

### Implement session authentication related logic according to your own business

#### Define the implementation of the business on the session

It is enough to implement the `IWsSession` interface for session authentication.

This interface encapsulates a `userId`, the parameter user can customize its type. There is also a channel, 
and subsequent user interactions with the object depend on this object

```java
package plus.jdk.broadcast.test.session;

import io.netty.channel.Channel;
import lombok.AllArgsConstructor;
import lombok.Data;
import plus.jdk.websocket.model.IWsSession;

@Data
@AllArgsConstructor
public class MyWsSession implements IWsSession<String> {

    private String userId;

    private Channel channel;
}
```

#### Custom `Session` validator

The authenticator must implement the `IWSSessionAuthenticator` interface. 
The usage example is as follows:

```java
package plus.jdk.broadcast.test.session;

import io.netty.channel.Channel;
import io.netty.handler.codec.http.FullHttpRequest;
import lombok.Getter;
import org.springframework.stereotype.Component;
import plus.jdk.websocket.common.HttpWsRequest;
import plus.jdk.websocket.global.IWSSessionAuthenticator;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@Component
public class WSSessionAuthenticator implements IWSSessionAuthenticator<MyWsSession> {

    @Getter
    private final Map<String, MyWsSession> ourWsSessionMap = new ConcurrentHashMap<>();

    @Override
    public MyWsSession authenticate(Channel channel, FullHttpRequest req, String path) throws Exception{
        HttpWsRequest httpWsRequest = new HttpWsRequest(req);
        
        // 此处的uid可根据业务实际情况写入，用于后续的业务逻辑
        String uid = httpWsRequest.getQueryValue("uid");
        if(uid == null) {
            throw new Exception("invalid connect"); // 验证失败，抛出异常
        }
        return new MyWsSession(uid, channel);
    }
}
```

**Specify the validator component via configuration**

```
plus.jdk.websocket.session-authenticator=plus.jdk.broadcast.test.session.WSSessionAuthenticator
```


### Write relevant business logic

As above, after implementing Session authentication, you can continue to write the business logic of websocket.

```java
package plus.jdk.broadcast.test.websocket;

import io.netty.channel.Channel;
import io.netty.handler.codec.http.FullHttpRequest;
import io.netty.handler.codec.http.HttpHeaders;
import org.springframework.stereotype.Component;
import org.springframework.web.bind.annotation.RequestParam;
import plus.jdk.broadcast.test.session.MyWsSession;
import plus.jdk.websocket.annotations.*;

@Component
@WebsocketHandler(values = {"/ws/message"})
public class DemoHandler {

    @OnWsHandshake
    public void doHandshake(FullHttpRequest req, MyWsSession session) {
    }

    @OnWsOpen
    public void onOpen(Channel channel, FullHttpRequest req, MyWsSession session, HttpHeaders headers, @RequestParam String uid) {
        session.sendText("onOpen" + System.currentTimeMillis());
    }

    @OnWsMessage
    public void onWsMessage(MyWsSession session, String data) {
        session.sendText("onWsMessage" + System.currentTimeMillis());
        session.sendText("receive data" + data);
        session.sendText("onWsMessage, id:" + session.getChannel().id().asShortText());
    }

    @OnWsEvent
    public void onWsEvent(Object data, MyWsSession session) {
        session.sendText("onWsEvent" + System.currentTimeMillis());
    }

    @OnWsBinary
    public void OnWsBinary(MyWsSession session, byte[] data) {
        session.sendText("OnWsBinary" + System.currentTimeMillis());
    }

    @OnWsError
    public void onWsError(MyWsSession session, Throwable throwable) {
        session.sendText("onWsError" + throwable.getMessage());
    }

    @OnWsClose
    public void onWsClose(MyWsSession session){
        session.sendText("onWsClose" + System.currentTimeMillis());
    }
}
```

