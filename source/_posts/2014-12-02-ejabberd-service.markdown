---
layout: post
title: "ejabberd中Jabber组件协议实现"
date: 2014-12-02 15:12:16 +0800
comments: true
categories: XMPP
---
[XEP-0114](http://xmpp.org/extensions/xep-0114.html)中定义了Jabber组件协议(Jabber Componet Protocol)。XMPP网络外的可信组件可以使用这个协议和XMPP网络内实体进行通信。

组件协议定义了两种模式：

* accept：外部组件向XMPP服务器发起连接
* connect：XMPP服务器向外部组件发起连接

其中, accept方式使用比较广泛，ejabberd中只实现了accept方式。

组件协议像XMPP一样，也是基于XML流，使用的XMLNS为`jabber:componet:accept`或者`jabber:component:connect`。

accept方式的协议流程：

* 外部组件建立到XMPP服务器的TCP连接，发送流头。
```XML
<stream:stream
    xmlns='jabber:component:accept'
    xmlns:stream='http://etherx.jabber.org/streams'
    to='plays.shakespeare.lit'>
```
* XMPP服务器回应，也发送流头，其中必须包括流ID属性:
```XML
<stream:stream
    xmlns:stream='http://etherx.jabber.org/streams'
    xmlns='jabber:component:accept'
    from='plays.shakespeare.lit'
    id='3BF96D32'>
```
* 外部组件发送身份验证摘要信息。
```XML
<handshake>aaee83c26aeeafcbabeabfcbcd50df997e0a2a1e</handshake>
```
组件协议身份验证不使用SASL，也不使用已废弃的[XEP-0078](http://xmpp.org/extensions/xep-0078.html)。它使用双方共享密钥计算摘要信息来验证身份。计算方法如下：

    1. 将服务器流头中的流ID属性和共享密钥拼接成字符串
    2. 计算该字符串的SHA1哈希值，并转换成小写16进制字符串

* XMPP服务器用同样方法计算进行校验。通过后，返回一个空的handshake元素。
```XML
<handshake/>
```
至此，外部组件和XMPP服务器就可以交换XMPP消息了。

我们来看ejabberd中组件协议实现，位于ejabberd_service.erl模块中。

ejabberd中的ejabberd_service的默认配置为:
```erlang
{8888, ejabberd_service, [
                          {access, all},
                          {shaper_rule, fast},
                          {ip, {127, 0, 0, 1}},
                          {hosts, ["icq.example.org", "sms.example.org"],
                                  [{password, "secret"}]
                          }
                         ]},
```
ejabberd_service是端口8888的处理模块。当有ejabberd接收端口上的TCP连接后，ejabberd_socket:start/4调用处理模块的socket_type/0, 根据返回值进行不同处理。ejabberd_service:socket_type/0返回xml_stream。它的处理流程和ejabberd_c2s模块相同。ejabberd为每个TCP连接分别创建一个receiver进程和一个处理进程(这里是ejabberd_service进程)。receiver进程接收消息并解析，然后发送相应的消息给处理进程。具体不再详述，请参考:`ejabberd消息处理流程分析`。

service进程为gen_fsm进程，初始状态为wait_for_stream。receiver进程接收到XML流头后发送xmlstreamstart消息给service进程。service进程调用wait_for_stream函数进行处理。
```erlang
wait_for_stream({xmlstreamstart, _Name, Attrs}, StateData) ->
    case xml:get_attr_s("xmlns", Attrs) of
    "jabber:component:accept" ->
        %% Note: XEP-0114 requires to check that destination is a Jabber
        %% component served by this Jabber server.
        %% However several transports don't respect that,
        %% so ejabberd doesn't check 'to' attribute (EJAB-717)
        To = xml:get_attr_s("to", Attrs),
        Header = io_lib:format(?STREAM_HEADER,
                   [StateData#state.streamid, xml:crypt(To)]),
        send_text(StateData, Header),
        {next_state, wait_for_handshake, StateData};
    _ ->
        send_text(StateData, ?INVALID_HEADER_ERR),
        {stop, normal, StateData}
    end;
```
wait_for_stream检测到XML流头中XMLNS为"jabber:component:accept"后，向组件发送流头，状态变更为wait_for_handshake。

receiver进程收到handshake消息后，发送xmlstreamelement消息给service进程，service调用wait_for_handshake处理。
```erlang
wait_for_handshake({xmlstreamelement, El}, StateData) ->
    {xmlelement, Name, _Attrs, Els} = El,
    case {Name, xml:get_cdata(Els)} of
    {"handshake", Digest} ->
        case sha:sha(StateData#state.streamid ++
             StateData#state.password) of
        Digest ->
            send_text(StateData, "<handshake/>"),
            lists:foreach(
              fun(H) ->
                  ejabberd_router:register_route(H),
                  ?INFO_MSG("Route registered for service ~p~n", [H])
              end, StateData#state.hosts),
            {next_state, stream_established, StateData};
        _ ->
            send_text(StateData, ?INVALID_HANDSHAKE_ERR),
            {stop, normal, StateData}
        end;
    _ ->
        {next_state, wait_for_handshake, StateData}
    end;
```
wait_for_handshake使用XML流ID和密码计算身份验证摘要，和组件所发的摘要信息进行对比判断是否通过。检查通过后，发送空的handshake元素。然后调用ejabberd_router:register_route/1依次注册配置的所有service域名。这样，XMPP实体发往这些域名的消息都将被ejabberd_router路由给该service进程。service进程状态变更为stream_established。

至此，外部组件就可以和XMPP服务器交换XMPP消息了。

组件向XMPP服务器发送消息后，receiver进程解析后向service进程发送xmlstreamelement消息，service进程调用stream_established处理。
```erlang
stream_established({xmlstreamelement, El}, StateData) ->
    NewEl = jlib:remove_attr("xmlns", El),
    {xmlelement, Name, Attrs, _Els} = NewEl,
    From = xml:get_attr_s("from", Attrs),
    ...
    To = xml:get_attr_s("to", Attrs),
    ToJID = case To of
        "" -> error;
        _ -> jlib:string_to_jid(To)
        end,
    if
    ((Name == "iq") or
     (Name == "message") or
     (Name == "presence")) and
    (ToJID /= error) and (FromJID /= error) ->
        ejabberd_router:route(FromJID, ToJID, NewEl);
    true ->
        Err = jlib:make_error_reply(NewEl, ?ERR_BAD_REQUEST),
        send_element(StateData, Err),
        error
    end,
    {next_state, stream_established, StateData};
```
stream_established进行一系列检查后，调用ejabberd_router:route转发消息。

XMPP实体发送给service域名的消息会由ejabberd_router以route消息的格式发给service进程。service进程调用handle_info处理。
```erlang
handle_info({route, From, To, Packet}, StateName, StateData) ->
    case acl:match_rule(global, StateData#state.access, From) of
    allow ->
        {xmlelement, Name, Attrs, Els} = Packet,
        Attrs2 = jlib:replace_from_to_attrs(jlib:jid_to_string(From),
                        jlib:jid_to_string(To),
                        Attrs),
        Text = xml:element_to_binary({xmlelement, Name, Attrs2, Els}),
        send_text(StateData, Text);
    deny ->
        Err = jlib:make_error_reply(Packet, ?ERR_NOT_ALLOWED),
        ejabberd_router:route_error(To, From, Err, Packet)
    end,
    {next_state, StateName, StateData};
```
handle_info首先进行ACL检查，通过后，修改From和To属性，将消息发送给组件。

使用telnet演示简单登录过程:
```
[root@flygoast flygoast]# telnet 127.0.0.1 8888
Trying 127.0.0.1...
Connected to 127.0.0.1.
Escape character is '^]'.
<stream:stream
    xmlns='jabber:component:accept'
    xmlns:stream='http://etherx.jabber.org/streams'
    to='sms.example.com'>
<?xml version='1.0'?><stream:stream xmlns:stream='http://etherx.jabber.org/streams' xmlns='jabber:component:accept' id='2744762983' from='sms.example.com'><handshake>cffc7fab4feae018a325ea834d2dca8c3b707a51</handshake>
<handshake/>
```
身份校验信息计算:
```
[root@flygoast flygoast]# echo -n "2744762983secret" | sha1sum
cffc7fab4feae018a325ea834d2dca8c3b707a51  -
```
