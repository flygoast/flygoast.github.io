---
layout: post
title: "ejabberd登录验证流程分析"
date: 2014-11-14 18:13:27 +0800
comments: true
categories: XMPP
---
XMPP使用SASL进行登录验证。ejabberd中模仿CyrusSASL库自己实现了SASL协议，API用法与CyrusSASL类似。

ejabberd启动时，ejabberd_app:start/0会调用cyrsasl:start/0。它创建了一个ets表`sasl_mechanism`，然后调用各个SASL机制模块的start函数。
```erlang
start() ->
    ets:new(sasl_mechanism, [named_table,
                             public,
                             {keypos, #sasl_mechanism.mechanism}]),
    cyrsasl_plain:start([]),
    cyrsasl_digest:start([]),
    cyrsasl_scram:start([]),
    cyrsasl_anonymous:start([]),
    ok.
```
各个SASL机制模块的start/1函数会调用cyrsasl:register_mechanism/3, 参数分别为SASL机制名称，机制处理模块，机制支持的密码存储类型。register_mechanism会将机制信息添加到sasl_mechanism表中。
```erlang
start(_Opts) ->
    cyrsasl:register_mechanism("PLAIN", ?MODULE, plain),
    ok.
```

```erlang
register_mechanism(Mechanism, Module, PasswordType) ->
    ets:insert(sasl_mechanism,
               #sasl_mechanism{mechanism = Mechanism,
                               module = Module,
                               password_type = PasswordType}).
```
客户端连接到ejabberd后，ejabberd会创建两个进程。一个ejabberd_c2s进程来处理连接状态并向客户端发送数据，另一个ejabberd_receiver进程接收客户端数据。ejabberd_c2s进程实现了gen_fsm行为，有限状态机的初始状态为wait_for_stream。ejabberd_c2s:init/1的简化代码逻辑为:
```erlang
init([{SockMod, Socket}, Opts]) ->
    ...
    IP = peerip(SockMod, Socket),
    %% Check if IP is blacklisted:
    case is_ip_blacklisted(IP) of
    true ->
        ?INFO_MSG("Connection attempt from blacklisted IP: ~s (~w)",
                  [jlib:ip_to_list(IP), IP]),
        {stop, normal};
    false ->
        ...
        {ok, wait_for_stream, #state{socket         = Socket1,
                                     sockmod        = SockMod,
                                     socket_monitor = SocketMonitor,
                                     xml_socket     = XMLSocket,
                                     zlib           = Zlib,
                                     tls            = TLS,
                                     tls_required   = StartTLSRequired,
                                     tls_enabled    = TLSEnabled,
                                     tls_options    = TLSOpts,
                                     streamid       = new_id(),
                                     access         = Access,
                                     shaper         = Shaper,
                                     ip             = IP},
         ?C2S_OPEN_TIMEOUT}
    end.
```
ejabberd_receiver进程解析到XML流头之后会调用gen_fsm:send_event向ejabberd_c2s进程发送消息。
```erlang
process_data([Element|Els], #state{c2s_pid = C2SPid} = State)
  when element(1, Element) == xmlelement;
       element(1, Element) == xmlstreamstart;
       element(1, Element) == xmlstreamelement;
       element(1, Element) == xmlstreamend ->
    if
    C2SPid == undefined ->
        State;
    true ->
        catch gen_fsm:send_event(C2SPid, element_wrapper(Element)),
        process_data(Els, State)
    end;
```
ejabberd_c2s进程接收到`{xmlstreamstart, _Name, Attrs}`消息后，调用状态函数wait_for_stream/2来处理。wait_for_stream在一系列正确性校验通过之后，回应给客户端XML流头。如果这个XML流之前还没有通过登录验证，则进行登录验证过程。简化的代码逻辑如下:
```erlang
case StateData#state.authenticated of
false ->
    SASLState = cyrsasl:server_new(
        "jabber", Server, "", [],
        fun(U) ->
            ejabberd_auth:get_password_with_authmodule(U, Server)
        end,
        fun(U, P) ->
            ejabberd_auth:check_password_with_authmodule(U, Server, P)
        end,
        fun(U, P, D, DG) ->
            ejabberd_auth:check_password_with_authmodule(U, Server, P, D, DG)
        end),
    Mechs = lists:map(
            fun(S) ->
                {xmlelement, "mechanism", [], [{xmlcdata, S}]}
            end, cyrsasl:listmech(Server)),
    ...

    send_element(StateData,
            {xmlelement, "stream:features", [],
            TLSFeature ++ CompressFeature ++
            [{xmlelement, "mechanisms",
            [{"xmlns", ?NS_SASL}],
            Mechs}] ++
            ejabberd_hooks:run_fold(
                c2s_stream_features,
                Server,
                [], [Server])}),
    fsm_next_state(wait_for_feature_request,
            StateData#state{
            server = Server,
            sasl_state = SASLState,
            lang = Lang});
_ ->
    ...
end
```
首先调用cyrsasl:server_new/7创建一个SASL验证状态, 其中存储了3个用于密码校验的回调函数。
```erlang
server_new(Service, ServerFQDN, UserRealm, _SecFlags,
       GetPassword, CheckPassword, CheckPasswordDigest) ->
    #sasl_state{service = Service,
                myname = ServerFQDN,
                realm = UserRealm,
                get_password = GetPassword,
                check_password = CheckPassword,
                check_password_digest= CheckPasswordDigest}.
```
wait_for_stream函数接着调用cyrsasl:listmech/1获取当前域名所支持的SASL验证机制。
```erlang
listmech(Host) ->
    Mechs = ets:select(sasl_mechanism,
       [{#sasl_mechanism{mechanism = '$1',
                         password_type = '$2',
                         _ = '_'},
        case catch ejabberd_auth:store_type(Host) of
        external ->
             [{'==', '$2', plain}];
        scram ->
             [{'/=', '$2', digest}];
        {'EXIT',{undef,[{Module,store_type,[]} | _]}} ->
             ?WARNING_MSG("~p doesn't implement the function store_type/0", [Module]),
             [];
        _Else ->
             []
        end,
        ['$1']}]),
    filter_anonymous(Host, Mechs).
```
listmech函数调用ejabberd_auth:store_type/1从ejabberd.cfg文件获取密码存储格式(auth_password_format)的配置。从sasl_mechanism表中查询出支持该密码存储格式的SASL机制。wait_for_stream函数将这些机制组织成XMPP协议格式发送回客户端，将当前进程状态改为wait_for_feature_request。比如，发送的机制列表为：
```XML
<stream:features>
<mechanisms xmlns="urn:ietf:params:xml:ns:xmpp-sasl">
<mechanism>PLAIN</mechanism>
<mechanism>DIGEST-MD5</mechanism>
<mechanism>SCRAM-SHA-1</mechanism>
</mechanisms>
<c xmlns="http://jabber.org/protocol/caps" node="http://www.process-one.net/en/ejabberd/" ver="yy7di5kE0syuCXOQTXNBTclpNTo=" hash="sha-1"/>
<register xmlns="http://jabber.org/features/iq-register"/>
</stream:features>
```
客户端从其中选择一个机制并发送给ejabberd服务器。如:
```XML
<auth xmlns="urn:ietf:params:xml:ns:xmpp-sasl" mechanism="PLAIN">AGFhYQAxMjM=</auth>
```
ejabberd_receiver进程解析完这个XML元素后，发送消息`{xmlstreamelement, El}`给ejabberd_c2s进程。ejabberd_c2s进程调用wait_for_feature_request函数进行处理。
```erlang
Mech = xml:get_attr_s("mechanism", Attrs),
ClientIn = jlib:decode_base64(xml:get_cdata(Els)),
case cyrsasl:server_start(StateData#state.sasl_state,
                          Mech,
                          ClientIn) of
{ok, Props} ->
    (StateData#state.sockmod):reset_stream(StateData#state.socket),
    send_element(StateData, {xmlelement, "success", [{"xmlns", ?NS_SASL}], []}),
    U = xml:get_attr_s(username, Props),
    AuthModule = xml:get_attr_s(auth_module, Props),
    ...
    fsm_next_state(wait_for_stream,
                   StateData#state{
                       streamid = new_id(),
                       authenticated = true,
                       auth_module = AuthModule,
                       user = U });
{continue, ServerOut, NewSASLState} ->
    send_element(StateData,
                 {xmlelement, "challenge",
                 [{"xmlns", ?NS_SASL}],
                 [{xmlcdata,
                 jlib:encode_base64(ServerOut)}]}),
    fsm_next_state(wait_for_sasl_response,
                   StateData#state{
                       sasl_state = NewSASLState});
{error, Error, Username} ->
    IP = peerip(StateData#state.sockmod, StateData#state.socket),
    ...
    send_element(StateData,
                 {xmlelement, "failure",
                 [{"xmlns", ?NS_SASL}],
                 [{xmlelement, Error, [], []}]}),
    {next_state, wait_for_feature_request, StateData, ?C2S_OPEN_TIMEOUT};
{error, Error} ->
    send_element(StateData,
                 {xmlelement, "failure",
                 [{"xmlns", ?NS_SASL}],
                 [{xmlelement, Error, [], []}]}),
    fsm_next_state(wait_for_feature_request, StateData)
end;
```
ejabberd_c2s进程判断收到的XML元素是auth请求后，从请求中获取客户端选择的机制mechanism，读取客户端发送的信息，然后调用cyrsasl:server_start/3。
```erlang
server_start(State, Mech, ClientIn) ->
    case lists:member(Mech, listmech(State#sasl_state.myname)) of
    true ->
        case ets:lookup(sasl_mechanism, Mech) of
        [#sasl_mechanism{module = Module}] ->
            {ok, MechState} = Module:mech_new(
                    State#sasl_state.myname,
                    State#sasl_state.get_password,
                    State#sasl_state.check_password,
                    State#sasl_state.check_password_digest),
            server_step(State#sasl_state{mech_mod = Module,
                         mech_state = MechState},
                ClientIn);
        _ ->
            {error, "no-mechanism"}
        end;
    false ->
        {error, "no-mechanism"}
    end.
```
server_start从sasl_mechanism表中查询出机制模块，并调用机制模块的mech_new/4。这个函数会创建一个机制本身的状态结构。如，PLAIN机制模块的mech_new/4：
```erlang
mech_new(_Host, _GetPassword, CheckPassword, _CheckPasswordDigest) ->
    {ok, #state{check_password = CheckPassword}}.
```
server_start函数将这个机制状态保存到SASL状态结构里的mech_state字段，调用server_step。而server_step则调用机制模块的mech_step。
```erlang
server_step(State, ClientIn) ->
    Module = State#sasl_state.mech_mod,
    MechState = State#sasl_state.mech_state,
    case Module:mech_step(MechState, ClientIn) of
    {ok, Props} ->
        case check_credentials(State, Props) of
        ok ->
            {ok, Props};
        {error, Error} ->
            {error, Error}
        end;
    {ok, Props, ServerOut} ->
        case check_credentials(State, Props) of
        ok ->
            {ok, Props, ServerOut};
        {error, Error} ->
            {error, Error}
        end;
    {continue, ServerOut, NewMechState} ->
        {continue, ServerOut,
         State#sasl_state{mech_state = NewMechState}};
    {error, Error, Username} ->
        {error, Error, Username};
    {error, Error} ->
        {error, Error}
    end.
```
Module:mech_step根据自身机制状态，返回不同的值。当验证通过时，返回ok信息。如若还需要其他信息继续验证，则返回continue信息。验证出错时，返回error信息。wait_for_feature_request函数根据不同的返回值，进行不同的处理。

* 当返回ok信息时，ejabberd_c2s进程向客户端发送验证成功的消息，登录验证流程结束。
* 当返回continue信息时，表示验证流程需要继续，因而向客户端返回服务端的chelange信息，ejabberd_c2s进程状态变为wait_for_sasl_response。当客户端再回应验证信息后，wait_for_sasl_response函数再次调用server_step进行验证处理。
* 当返回error信息时，ejabberd_c2s进程向客户端发送验证失败的消息，登录验证流程结束。

PLAIN机制只需要用户名和密码，这些信息附在客户端选择机制的AUTH请求中。因而只需要调用一次mech_step函数，mech_step也因而不会返回continue。cyrsasl_plain:mech_step调用mech_new中传入的check_password回调函数(不同机制会调用不同的回调函数)来检查用户名和密码是否正确。这部分逻辑由ejabberd_auth模块封装不同的模块完成，本文略过。
```erlang
mech_step(State, ClientIn) ->
    case prepare(ClientIn) of
    [AuthzId, User, Password] ->
        case (State#state.check_password)(User, Password) of
        {true, AuthModule} ->
            {ok, [{username, User}, {authzid, AuthzId},
              {auth_module, AuthModule}]};
        _ ->
            {error, "not-authorized", User}
        end;
    _ ->
        {error, "bad-protocol"}
    end.
```
SCRAM机制需要多次Chelange/Response交互，需要多次调用它的mech_step。因而它在机制状态内部来分步骤完成，具体可以参考cyrsasl_scram.erl，本文不详述。

当机制mech_step返回ok或error时，ejabberd_c2s进程返回给客户端相应的回应，登录验证的流程结束。
