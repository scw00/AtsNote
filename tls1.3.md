# 2.  协议综述

   TLS握手协议会产生一个加密参数，这个参数被用于加密的管道，保证双发通讯安全性。client和 server会在第一次通讯时使用这个子协议。握手协议允许对端协商协议版本号，选择加密算法，可选的对端验证和建立共享秘钥的材料。一旦握手成功，对端使用协商的key来保护应用层的数据，

   如果我握手失败，可以选择性发送一个警告信息之后关闭连接，或者直接关闭连接。

   TLS 支持三种基本的key交换模式：
   - (EC)DHE (Diffie-Hellman over either finite fields or elliptic
   - PSK-only
   - PSK with (EC)DHE

```

       Client                                               Server

Key  ^ ClientHello
Exch | + key_share*
     | + signature_algorithms*
     | + psk_key_exchange_modes*
     v + pre_shared_key*         -------->
                                                       ServerHello  ^ Key
                                                      + key_share*  | Exch
                                                 + pre_shared_key*  v
                                             {EncryptedExtensions}  ^  Server
                                             {CertificateRequest*}  v  Params
                                                    {Certificate*}  ^
                                              {CertificateVerify*}  | Auth
                                                        {Finished}  v
                                 <--------     [Application Data*]
     ^ {Certificate*}
Auth | {CertificateVerify*}
     v {Finished}                -------->
       [Application Data]        <------->      [Application Data]

              +  Indicates noteworthy extensions sent in the
                 previously noted message.

              *  Indicates optional or situation-dependent
                 messages/extensions that are not always sent.

              {} Indicates messages protected using keys
                 derived from a [sender]_handshake_traffic_secret.

              [] Indicates messages protected using keys
                 derived from [sender]_application_traffic_secret_N

               Figure 1: Message flow for full TLS Handshake
```


   The handshake can be thought of as having three phases (indicated in
   the diagram above):

   握手可以分成三个阶段(如上图)：
   - key交换：建立共享key的材料和选择加密参数。自此阶段之后的所用阶段都是加密的
   - server参数：建立(ps: 这里建立应该是和对端协商的意思)其他的加密参数(这些参数由app提供，并且无视client是否被认证)
   - 认证：对server的认证(client端的认证可选)，对key和握手的确认。
   
   在kye交换阶段，client发送ClientHello(Section 4.1.2)消息，这个消息包含了一个随机的nonce(ClientHello.random)，并且提供洗衣版本号，一个此时的cipher/HKDF哈希对，一个Diffie-Hellman共享key(存放在key_share扩展项，Section 4.2.8)或者一套PSK标记(存放在pre_shared_key 扩展项，Section 4.2.8)，或者两者都提供。和一些可能的添加扩展项。

   Server 处理ClientHello消息并且选择适当的加密参数。然后发送ServerHello(Section 4.1.3)，消息，这个消息包含协商的链接参数。并使用ClientHello和ServerHello拼接成共享秘钥。
   如果使用(EC)DHE key，那么SeverHello将包含key_share可选项和临时的Diffle-Hellman share，这个share的组必须与client的share 的组一致(ps:client 可以提供一组 组，server需要选择一个组)。如果使用PSK，那么ServerHello将包含pre_shared_key扩展项，这个扩展项指明哪一个client 提供的PSK被使用。注意：实现者可以一起使用(EC)DHE和PSK，此时必须包含上述两个扩展项。

   然后server发送两个消息来交换server 参数：
   - 加密扩展项： 响应ClientHello扩展项，决定加密参数不是必须的。其他的项将会指明独立的证书
   - 证书请求：如果需要基于证书的客户端认证，和证书需要的一些参数，可以使用这个消息。如果不需要client端验证，这个消息可以不省略。
   
   最后，client和server交换认证信息。如果需要认证，TLS使用每次都会使用相同的消息组。特别是：
   - 证书：一端的证书和每个证书的扩展项，如果不需要client端的认证，server端的证书请求可以被省略，server 可以不发送CertificateRequest下次(这样可以指定client必须不认证证书？？wtf)。注意：如果原始公钥或者或存的扩展选项信息被使用，那么这个消息将不会包含证书，但是会提供与server长期key相对应的信息。
   - 证书确认：通过使用Certificate消息中包含的公钥的对应的私钥生成一个签名，整个握手阶段都会使用这个签名。如果一段不需要验证证书，此消息可以被省略。
   - 结束：握手阶段使用的MAC(Message Authentication Code)。这个消息对key确认，将交换的key与端绑定，PSK模式中确认握手[Section 4.4.4]

   Upon receiving the server's messages, the client responds with its
   Authentication messages, namely Certificate and CertificateVerify (if
   requested), and Finished.

   通过接收server消息，client回应自己证书和证书确认(如果需要的话)，然后结束。

   此时，握手完成，client和server生成记录层需要的key材料。并通过认证过的加密方式来保护app层的数据。app数据不能够在结束消息之前和记录层开始使用加密key之前发送。注意如果server在接收client认证消息之前发送app数据，这些数据将会被发送到未被认证的对端。

## 2.1.  Incorrect DHE Share

   如果客户端没有提供足够的key_share扩展项(比如， 只包含server不支持或者不接受DHE和ECDHE组)，server发送HelloRetryRequest来纠正不匹配的client消息。然后client需要使用正确key_share扩展项重新握手(如图2所示)。如果没有共同的加密参数，server必须使用适当的警报来终止握手。

```
            Client                                               Server

            ClientHello
            + key_share             -------->
                                    <--------         HelloRetryRequest
                                                            + key_share

            ClientHello
            + key_share             -------->
                                                            ServerHello
                                                            + key_share
                                                  {EncryptedExtensions}
                                                  {CertificateRequest*}
                                                         {Certificate*}
                                                   {CertificateVerify*}
                                                             {Finished}
                                    <--------       [Application Data*]
            {Certificate*}
            {CertificateVerify*}
            {Finished}              -------->
            [Application Data]      <------->        [Application Data]

        Figure 2: Message flow for a full handshake with mismatched
                                parameters
```

   注意：上图交换初始的ClientHello和HelloRetryRequest，这并不意味着原始的clientHello被重置(使用新的ClientHello来重置旧的ClientHello)。

   TLS提供的一些优化基本握手的参数将在下面几章说明。

## 2.2.  Resumption and Pre-Shared Key (PSK)

   尽管TLS PSKs是由带外数据建立的，实现者也可以复用前面链接生成的PSKs(汇话复用)。一旦握手完成，server可以向client发送一个PSK id，这个id对应一个第一无二的由initial handshake(see Section 4.6.1)生成的key。client可以在以后的握手中使用这个psk id来协商与之相对应的psk。如果server端接受了，那么新链接的加密上下文将会被关联到原始的链接，initail handshake生成的key将会启动加密状态，这样避免了完整的握手。在TLS1.2和以下的版本中。这个功能可以由汇话id和汇话ticket提供[RFC5077]。这两个状态机都被包含在TLS1.3中。

   PSKs和(EC)DHE key交换一起使用，用于提供shared key拼接的前置安全性，或者单独使用。这将会导致缺乏对前置的app 数据缺乏保护。

   Figure 3 shows a pair of handshakes in which the first establishes a
   PSK and the second uses it:

   图3 说明使用pSK完成握手

```
          Client                                               Server

   Initial Handshake:
          ClientHello
          + key_share               -------->
                                                          ServerHello
                                                          + key_share
                                                {EncryptedExtensions}
                                                {CertificateRequest*}
                                                       {Certificate*}
                                                 {CertificateVerify*}
                                                           {Finished}
                                    <--------     [Application Data*]
          {Certificate*}
          {CertificateVerify*}
          {Finished}                -------->
                                    <--------      [NewSessionTicket]
          [Application Data]        <------->      [Application Data]


   Subsequent Handshake:
          ClientHello
          + key_share*
          + pre_shared_key          -------->
                                                          ServerHello
                                                     + pre_shared_key
                                                         + key_share*
                                                {EncryptedExtensions}
                                                           {Finished}
                                    <--------     [Application Data*]
          {Finished}                -------->
          [Application Data]        <------->      [Application Data]

               Figure 3: Message flow for resumption and PSK
```


   如果server通过PSK认证，server 不会发送证书和证书确认消息。当客户端使用PSK来复用时，client必须提供key_share扩展项，这样可以被用于server在需要的时候拒绝并恢复到完整的握手。server 回应pre_shared_key扩展项来协商PSK key的使用或者回复key_share扩展项来进行(EC)DHE key的接力，这样来提供前置加密。











