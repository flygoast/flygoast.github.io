---
layout: post
title: "ejabberd中XMPP协议PING实现"
date: 2014-11-24 18:09:43 +0800
comments: true
categories: XMPP
---
根据[XEP-0199](http://xmpp.org/extensions/xep-0199.html), XMPP客户端和服务器都可以在XML流上发送应用层PING请求。因为XMPP依赖底层的TCP连接，有可能TCP连接意外中断，而上层的XMPP并不知晓，从而影响消息传递。通过发送应用层PING请求可以来确认对端的连接可用性。

以服务器发给客户端为例，协议如下：

发送的PING请求：
```XML
<iq from='capulet.lit' to='juliet@capulet.lit/balcony' id='s2c1' type='get'>
  <ping xmlns='urn:xmpp:ping'/>
</iq>
```
如果对端支持PING请求，则返回对应的"PONG"回应。
```XML
<iq from='juliet@capulet.lit/balcony' to='capulet.lit' id='s2c1' type='result'/>
```
如果对端不支持则返回<service-unavailable/>错误。
```XML
<iq from='juliet@capulet.lit/balcony' to='capulet.lit' id='s2c1' type='error'>
  <ping xmlns='urn:xmpp:ping'/>
  <error type='cancel'>
    <service-unavailable xmlns='urn:ietf:params:xml:ns:xmpp-stanzas'/>
  </error>
</iq>
```
ejabberd中PING功能实现位于mod_ping.erl。它主要支持3个配置：

* **send_pings**: *true|false*

如果这个选项设置为true, 当客户端在给定时间间隔内没有活动，则向客户端发送一个ping请求。

* **ping_interval**: *Seconds*

设置上述send_pings选项中客户端没有活动的时间间隔。

* **timeout_action**: *none|kill*

表示当PING请求发出32秒后，ejabberd依然没有收到PING响应，服务端如何处理。none表示什么也不做，kill表示关闭客户端连接。

当ejabberd启动时会调用mod_ping:start/2。
```erlang
start(Host, Opts) ->
    Proc = gen_mod:get_module_proc(Host, ?MODULE),
    PingSpec = {Proc, {?MODULE, start_link, [Host, Opts]},
                transient, 2000, worker, [?MODULE]},
    supervisor:start_child(?SUPERVISOR, PingSpec).
```
start函数调用supervisor:start_child/2为每个支持的host创建一个负责该host的worker进程。

进程树模型如下:
```Raw token data
+------------+
|ejabberd_sup|
+-----+------+
      |
      |    +------------------+
      +--->|Other processes...|
      |    +------------------+
      |
      |    +------------------------+
      +--->|ping(im.just4coding.com)|
      |    +------------------------+
      |
      |    +------------------------+
      +--->|ping(localhost)         |
      |    +------------------------+
      |
      |    +------------------------+
      +--->|ping(Other host)        |
           +------------------------+
```
每个worker是一个gen_server进程，进程调用init函数进行初始化。
```erlang
init([Host, Opts]) ->
    SendPings = gen_mod:get_opt(send_pings, Opts, ?DEFAULT_SEND_PINGS),
    PingInterval = gen_mod:get_opt(ping_interval, Opts, ?DEFAULT_PING_INTERVAL),
    TimeoutAction = gen_mod:get_opt(timeout_action, Opts, none),
    IQDisc = gen_mod:get_opt(iqdisc, Opts, no_queue),
    mod_disco:register_feature(Host, ?NS_PING),
    gen_iq_handler:add_iq_handler(ejabberd_sm, Host, ?NS_PING,
                                  ?MODULE, iq_ping, IQDisc),
    gen_iq_handler:add_iq_handler(ejabberd_local, Host, ?NS_PING,
                                  ?MODULE, iq_ping, IQDisc),
    case SendPings of
    true ->
        %% Ping requests are sent to all entities, whether they
        %% announce 'urn:xmpp:ping' in their caps or not
        ejabberd_hooks:add(sm_register_connection_hook, Host,
                           ?MODULE, user_online, 100),
        ejabberd_hooks:add(sm_remove_connection_hook, Host,
                           ?MODULE, user_offline, 100),
        ejabberd_hooks:add(user_send_packet, Host,
                           ?MODULE, user_send, 100);
    _ ->
        ok
    end,
    {ok, #state{host = Host,
                send_pings = SendPings,
                ping_interval = PingInterval,
                timeout_action = TimeoutAction,
                timers = ?DICT:new()}}.
```
* 首先获取相关配置

* 接着调用mod_disco:register_feature注册PING功能的XMLNS。这样当客户端请求"Service Discovery"信息时，ejabberd返回的特征中会包括"urn:xmpp:ping"。

ServiceDiscovery请求:
```XML
<iq type='get'
    from='juliet@capulet.lit/balcony'
    to='capulet.lit'
    id='disco1'>
  <query xmlns='http://jabber.org/protocol/disco#info'/>
</iq>
```

ServiceDiscovery响应:
```XML
<iq type='result'
    from='capulet.lit'
    to='juliet@capulet.lit/balcony'
    id='disco1'>
  <query xmlns='http://jabber.org/protocol/disco#info'>
    ...
    <feature var='urn:xmpp:ping'/>
    ...
  </query>
</iq>
```

ServiceDiscovery相关信息参考[XEP-0030](http://xmpp.org/extensions/xep-0030.html)。

* 接下来，注册IQ处理器，令XMLNS为"urn:xmpp:ping"的IQ请求由函数iq_ping处理。iq_ping简单地返回相应响应或者错误。
```erlang
iq_ping(_From, _To, #iq{type = Type, sub_el = SubEl} = IQ) ->
    case {Type, SubEl} of
        {get, {xmlelement, "ping", _, _}} ->
            IQ#iq{type = result, sub_el = []};
        _ ->
            IQ#iq{type = error, sub_el = [SubEl, ?ERR_FEATURE_NOT_IMPLEMENTED]}
    end.
```
如果send_pings配置为true, mod_ping在ejabberd中注册n以下3个hook函数:

* `sm_register_connection_hook`: 它在客户端完成登录验证，建立session信息时调用。

```erlang
open_session(SID, User, Server, Resource, Info) ->
    set_session(SID, User, Server, Resource, undefined, Info),
    mnesia:dirty_update_counter(session_counter,
                jlib:nameprep(Server), 1),
    check_for_sessions_to_replace(User, Server, Resource),
    JID = jlib:make_jid(User, Server, Resource),
    ejabberd_hooks:run(sm_register_connection_hook, JID#jid.lserver,
               [SID, JID, Info]).
```
* `sm_remove_connection_hook`: 在用户退出,关闭session时调用。
```erlang
close_session(SID, User, Server, Resource) ->
    Info = case mnesia:dirty_read({session, SID}) of
    [] -> [];
    [#session{info=I}] -> I
    end,
    F = fun() ->
        mnesia:delete({session, SID}),
        mnesia:dirty_update_counter(session_counter,
                        jlib:nameprep(Server), -1)
    end,
    mnesia:sync_dirty(F),
    JID = jlib:make_jid(User, Server, Resource),
    ejabberd_hooks:run(sm_remove_connection_hook, JID#jid.lserver,
               [SID, JID, Info]).
```
* `user_send_packet`: 在C2S进程收到客户端发送的消息时被调用。

`sm_register_connection_hook`的hook函数`user_online`和`user_send_packet`的hook函数`user_send`都会调用start_ping函数。
```erlang
start_ping(Host, JID) ->
    Proc = gen_mod:get_module_proc(Host, ?MODULE),
    gen_server:cast(Proc, {start_ping, JID}).
```
start_ping向该HOST的worker进程发送一个{start_ping, JID}消息。worker进程调用handle_cast进行处理:
```erlang
handle_cast({start_ping, JID}, State) ->
    Timers = add_timer(JID, State#state.ping_interval, State#state.timers),
    {noreply, State#state{timers = Timers}};
```
handle_cast调用add_timer为该客户端创建一个timer。
```erlang
add_timer(JID, Interval, Timers) ->
    LJID = jlib:jid_tolower(JID),
    NewTimers = case ?DICT:find(LJID, Timers) of
            {ok, OldTRef} ->
            cancel_timer(OldTRef),
            ?DICT:erase(LJID, Timers);
            _ ->
            Timers
        end,
    TRef = erlang:start_timer(Interval * 1000, self(), {ping, JID}),
    ?DICT:store(LJID, TRef, NewTimers).
```
由于用户每次发送消息时都会调用add_timer函数，因而add_timer中需要检查之前是否已经存在timer。如果存在，则先取消旧的timer, 再创建新的Timer。

当timer超时后，即客户若干时间内没有活动，进程收到{ping, JID}消息，此时ejabberd应向客户端发送PING消息。进程调用handle_info处理。
```erlang
handle_info({timeout, _TRef, {ping, JID}}, State) ->
    IQ = #iq{type = get,
             sub_el = [{xmlelement, "ping", [{"xmlns", ?NS_PING}], []}]},
    Pid = self(),
    F = fun(Response) ->
        gen_server:cast(Pid, {iq_pong, JID, Response})
    end,
    From = jlib:make_jid("", State#state.host, ""),
    ejabberd_local:route_iq(From, JID, IQ, F),
    Timers = add_timer(JID, State#state.ping_interval, State#state.timers),
    {noreply, State#state{timers = Timers}};
```
handle_info创建IQ消息后，设置回调函数F，调用ejabberd_local:route_iq/4消息IQ消息发送给客户端。当收到该IQ消息的响应或者超过32秒依然没有收到客户端的响应，回调函数F将会被调用。如果响应超时，Response为timeout，F将向进程发送{iq_pong, JID, timeout}消息。进程调用handle_cast处理。
```erlang
handle_cast({iq_pong, JID, timeout}, State) ->
    Timers = del_timer(JID, State#state.timers),
    ejabberd_hooks:run(user_ping_timeout, State#state.host, [JID]),
    case State#state.timeout_action of
    kill ->
        #jid{user = User, server = Server, resource = Resource} = JID,
        case ejabberd_sm:get_session_pid(User, Server, Resource) of
        Pid when is_pid(Pid) ->
            ejabberd_c2s:stop(Pid);
        _ ->
            ok
        end;
    _ ->
        ok
    end,
    {noreply, State#state{timers = Timers}};
```
如果timeout_action设置为kill, 则调用ejabberd_c2s:stop关闭相应的客户端连接。

因为在`sm_remove_connection_hook`注册了hook函数`user_offline`, 当用户退出时会调用stop_ping函数，向worker进程发送{stop_ping, JID}消息。
```erlang
stop_ping(Host, JID) ->
    Proc = gen_mod:get_module_proc(Host, ?MODULE),
    gen_server:cast(Proc, {stop_ping, JID}).
```
worker进程调用del_timer函数将该客户端的timer删除。
```erlang
handle_cast({stop_ping, JID}, State) ->
    Timers = del_timer(JID, State#state.timers),
    {noreply, State#state{timers = Timers}};
```
```erlang
del_timer(JID, Timers) ->
    LJID = jlib:jid_tolower(JID),
    case ?DICT:find(LJID, Timers) of
        {ok, TRef} ->
        cancel_timer(TRef),
        ?DICT:erase(LJID, Timers);
        _ ->
        Timers
    end.
```
模块及进程停止的逻辑与模块和进程初始化的逻辑相反，本文略过。
