---
layout:     post
title:      "kafka身份认证"
subtitle:   "sasl 实现原理"
date:       2019-01-01 14:32:30
author:     "邹盛富"
header-img: "img/moon-3666312__480.jpg"
tags:
    - kafka
---

## 身份认证介绍
身份认证指的是客户端（生产者或消费者）通过某些凭据，例如用户名/密码或是SSL证书，让服务端确认客户端的真实身份。
身份认证在Kafka中的具体体现是服务器是否允许与当前请求的客户端建立连接，在Kafka 0.10版本中，客户端与服务端支持使用SSL、SASL/PLAIN等方式进行连接，默认情况下，这些身份认证方式是不开启的。

### SSL
SSL（Secure Sockets Layer）是一种基于传输层（比如TCP/IP）或应用层（比如HTTP）的协议。SSL协议依赖于数字证书，数字证书中的核心组成部分是私钥和公钥两部分。SSL分为“握手协议”和“传输协议”两部分，其中“握手协议”是基于非对称加密的，而“传输协议”是基于对称加密的。“握手协议”的主要目的是为了在客户端与服务端之间协商和交换对称密钥。之所以这么做是因为非对称密钥的算法比较复杂，速度较慢，不适合对大量数据进行加密，而相较之下，对称加密速度较快，适合对大量数据加密。无论是对称加密还是非对称加密，使用之后都会影响系统的性能，我们需要在安全性和吞吐量之间进行适当权衡。

### SASL
SASL（Simple Authentication and Security Layer）是一种用来扩充C/S模式验证能力的认证机制，它定义了客户端和服务端之间数据的交换方式，但是并没指定数据的内容。其中，SASL/PLAIN是最简单的、也是最危险的机制，因为用户名和密码都是以明文的形式在网络中传输的，别人可以轻松地从网络中截取这些信息。一般情况，在安全的网络环境下考虑使用此机制，当然也可以将用户名密码进行加密以提高安全性。


## 客户端

客户端进行身份认证主要是以下几个流程：
- 创建`ChannelBuilder`,该对象是用来创建sokecet的
- 创建`LoginManager`,该对象中包含了`LoginContext`
- 创建`ClientAuthenticator`,该对象用来执行真正的认证操作
- 客户端与服务端进行认证

### 创建ChannelBuilder

客户端对象在刚创建的时候会调用`ClientUtils`中`createChannelBuilder(config)`函数从而创建相应的`ChannelBuilder`,其最终调用的函数如下：
```
private static ChannelBuilder create(SecurityProtocol securityProtocol,
                                     Mode mode,
                                     JaasContext.Type contextType,
                                     AbstractConfig config,
                                     ListenerName listenerName,
                                     String clientSaslMechanism,
                                     boolean saslHandshakeRequestEnable,
                                     CredentialCache credentialCache) {
    Map<String, ?> configs;
    if (listenerName == null)
        configs = config.values();
    else
        configs = config.valuesWithPrefixOverride(listenerName.configPrefix());

    //根据不同类型的协议创建不同的ChannelBuilder
    ChannelBuilder channelBuilder;
    switch (securityProtocol) {
        case SSL:
            requireNonNullMode(mode, securityProtocol);
            channelBuilder = new SslChannelBuilder(mode);
            break;
        case SASL_SSL:
        case SASL_PLAINTEXT:
            requireNonNullMode(mode, securityProtocol);
            JaasContext jaasContext = JaasContext.load(contextType, listenerName, configs);
            channelBuilder = new SaslChannelBuilder(mode, jaasContext, securityProtocol, listenerName,
                    clientSaslMechanism, saslHandshakeRequestEnable, credentialCache);
            break;
        case PLAINTEXT:
            channelBuilder = new PlaintextChannelBuilder();
            break;
        default:
            throw new IllegalArgumentException("Unexpected securityProtocol " + securityProtocol);
    }
    //根据传入的配置设置新创建的channelBuilder
    channelBuilder.configure(configs);
    return channelBuilder;
}
```
上面函数中会根据不同的协议创建不同的`ChannelBuilder`并进行相应的配置,在kafka中实现了`SslChannelBuilder`、`SaslChannelBuilder`和`PlaintextChannelBuilder`等不同类型的`ChannelBuilder`,我们这里以`SaslChannelBuilder`的`configure`函数为例：
```
public void configure(Map<String, ?> configs) throws KafkaException {
    try {
        this.configs = configs;
        boolean hasKerberos;
        // 判断是不是使用Kerberos 协议
        if (mode == Mode.SERVER) {
            @SuppressWarnings("unchecked")
            List<String> enabledMechanisms = (List<String>) this.configs.get(BrokerSecurityConfigs.SASL_ENABLED_MECHANISMS_CONFIG);
            hasKerberos = enabledMechanisms == null || enabledMechanisms.contains(SaslConfigs.GSSAPI_MECHANISM);
        } else {
            hasKerberos = clientSaslMechanism.equals(SaslConfigs.GSSAPI_MECHANISM);
        }

        if (hasKerberos) {
            String defaultRealm;
            try {
                defaultRealm = defaultKerberosRealm();
            } catch (Exception ke) {
                defaultRealm = "";
            }
            @SuppressWarnings("unchecked")
            List<String> principalToLocalRules = (List<String>) configs.get(BrokerSecurityConfigs.SASL_KERBEROS_PRINCIPAL_TO_LOCAL_RULES_CONFIG);
            if (principalToLocalRules != null)
                kerberosShortNamer = KerberosShortNamer.fromUnparsedRules(defaultRealm, principalToLocalRules);
        }
        //这里是重点，LoginManager是kafka中自定义的类，创建该类并进行验证
        this.loginManager = LoginManager.acquireLoginManager(jaasContext, hasKerberos, configs);

        if (this.securityProtocol == SecurityProtocol.SASL_SSL) {
            // Disable SSL client authentication as we are using SASL authentication
            this.sslFactory = new SslFactory(mode, "none");
            this.sslFactory.configure(configs);
        }
    } catch (Exception e) {
        close();
        throw new KafkaException(e);
    }
}

```

### 创建LoginManager
如上面函数中注释的，在配置的时候会调用静态方法创建一个`LoginManager`,先来看看`LoginManager`的构造函数如下:
```
private LoginManager(JaasContext jaasContext, boolean hasKerberos, Map<String, ?> configs,
                     Password jaasConfigValue) throws IOException, LoginException {
    this.cacheKey = jaasConfigValue != null ? jaasConfigValue : jaasContext.name();
    //在SASL/PLAIN身份认证的场景下，使用的是DefaultLogin实现
    login = hasKerberos ? new KerberosLogin() : new DefaultLogin();
    login.configure(configs, jaasContext);
    login.login();
}
```
关于`DefaultLogin`其内部定义没有什么特殊之处，直接看其父类`AbstractLogin`,`AbstractLogin`的`login`函数如下：
```
public LoginContext login() throws LoginException {
    loginContext = new LoginContext(jaasContext.name(), null, new LoginCallbackHandler(), jaasContext.configuration());
    // 完成认证操作
    loginContext.login();
    log.info("Successfully logged in.");
    return loginContext;
}
```

Kafka使用了JAAS的相关内容，JAAS在应用层与底层安全机制之间加入了一层抽象，简化了Java Security包之上的开发工作，可以为开发人员屏蔽掉具体使用的安全机制。上层应用的代码主要是面向LoginContext进行编程，在LoginContext下层是可动态配置的LoginModule组件（Kafka客户端使用的LoginModule是Kafka自定义的PlainLoginModule组件）。

我们了解到的代码只是从JAAS配置文件中读取了配置并填充Subject对象,没有进行实质性的认证操作。真正完成认证操作是通过`KafkaChannel`的`Authenticator`对象。上述过程中一共涉及到`LoginContext`、`LoginManager`、`AbstractLogin`等类，具体的关系图如下：
![](https://res.cloudinary.com/bytedance14/image/upload/v1568555427/8CFA85FC8F7F56EBA4A040637FACCCBF.jpg)

### 创建ClientAuthenticator

当生产者和消费者尝试生产或者消费的时候，会检查client与kafka broker之间是否已经建立tcp连接，没有建立连接的话先创建一个`KafkaChannel`对象,代码如下：

```
public KafkaChannel buildChannel(String id, SelectionKey key, int maxReceiveSize, MemoryPool memoryPool) throws KafkaException {
    try {
        SocketChannel socketChannel = (SocketChannel) key.channel();
        Socket socket = socketChannel.socket();
        TransportLayer transportLayer = buildTransportLayer(id, key, socketChannel);
        Authenticator authenticator;
        // 服务端Authenticator
        if (mode == Mode.SERVER)
            authenticator = buildServerAuthenticator(configs, id, transportLayer, loginManager.subject());
        else
        // 客户端Authenticator
            authenticator = buildClientAuthenticator(configs, id, socket.getInetAddress().getHostName(),
                    loginManager.serviceName(), transportLayer, loginManager.subject());
        return new KafkaChannel(id, transportLayer, authenticator, maxReceiveSize, memoryPool != null ? memoryPool : MemoryPool.NONE);
    } catch (Exception e) {
        // 如果创建连接失败, 抛出异常，上层函数会在back_off之后，进行重试
        log.info("Failed to create channel due to ", e);
        throw new KafkaException(e);
    }
}

```
`SaslClientAuthenticator`

### 认证过程

客户端会不断的判断channel上是否是可以读数据或者写数据，其代码如下：
```
void pollSelectionKeys(Set<SelectionKey> selectionKeys,
                       boolean isImmediatelyConnected,
                       long currentTimeNanos) {
    for (SelectionKey key : determineHandlingOrder(selectionKeys)) {
        KafkaChannel channel = channel(key);
        long channelStartTimeNanos = recordTimePerConnection ? time.nanoseconds() : 0;

        // register all per-connection metrics at once
        // 为每个channel 注册metrics
        sensors.maybeRegisterConnectionMetrics(channel.id());
        if (idleExpiryManager != null)
            // 更新每个channel的活跃时间
            idleExpiryManager.update(channel.id(), currentTimeNanos);

        boolean sendFailed = false;
        try {

            /* complete any connections that have finished their handshake (either normally or immediately) */
            // 对于每个已经建立tcp连接的channel加到本地客户端的链表中
            if (isImmediatelyConnected || key.isConnectable()) {
                if (channel.finishConnect()) {
                    this.connected.add(channel.id());
                    this.sensors.connectionCreated.record();
                    SocketChannel socketChannel = (SocketChannel) key.channel();
                    log.debug("Created socket with SO_RCVBUF = {}, SO_SNDBUF = {}, SO_TIMEOUT = {} to node {}",
                            socketChannel.socket().getReceiveBufferSize(),
                            socketChannel.socket().getSendBufferSize(),
                            socketChannel.socket().getSoTimeout(),
                            channel.id());
                } else
                    continue;
            }

            /* if channel is not ready finish prepare */
            // 对于已经建立tcp连接但是前期的认证还没有通过的chennel，进行认证准备
            if (channel.isConnected() && !channel.ready()) {
                try {
                    channel.prepare();
                } catch (AuthenticationException e) {
                    sensors.failedAuthentication.record();
                    throw e;
                }
                if (channel.ready())
                    sensors.successfulAuthentication.record();
            }
            // 从channel中读取数据
            attemptRead(key, channel);

            if (channel.hasBytesBuffered()) {
                //this channel has bytes enqueued in intermediary buffers that we could not read
                //(possibly because no memory). it may be the case that the underlying socket will
                //not come up in the next poll() and so we need to remember this channel for the
                //next poll call otherwise data may be stuck in said buffers forever. If we attempt
                //to process buffered data and no progress is made, the channel buffered status is
                //cleared to avoid the overhead of checking every time.
                keysWithBufferedRead.add(key);
            }

            /* if channel is ready write to any sockets that have space in their buffer and for which we have data */
            if (channel.ready() && key.isWritable()) {
                Send send = null;
                try {
                    send = channel.write();
                } catch (Exception e) {
                    sendFailed = true;
                    throw e;
                }
                if (send != null) {
                    this.completedSends.add(send);
                    this.sensors.recordBytesSent(channel.id(), send.size());
                }
            }

            /* cancel any defunct sockets */
            if (!key.isValid())
                close(channel, true, true);

        } catch (Exception e) {
            String desc = channel.socketDescription();
            if (e instanceof IOException)
                log.debug("Connection with {} disconnected", desc, e);
            else if (e instanceof AuthenticationException) // will be logged later as error by clients
                log.debug("Connection with {} disconnected due to authentication exception", desc, e);
            else
                log.warn("Unexpected error from {}; closing connection", desc, e);
            close(channel, !sendFailed, true);
        } finally {
            maybeRecordTimePerConnection(channel, channelStartTimeNanos);
        }
    }
}
```
