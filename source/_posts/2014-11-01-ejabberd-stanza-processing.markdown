---
layout: post
title: "ejabberd消息处理流程分析"
date: 2014-11-01 23:48:07 +0800
comments: true
categories: XMPP
---
ejabberd的程序入口为ejabberd_app.erl中的start/2，简化后逻辑如下:
```erlang
start(normal, _Args) ->
    ...
    Sup = ejabberd_sup:start_link(),
    ...
    ejabberd_listener:start_listeners(),
    ?INFO_MSG("ejabberd ~s is started in the node ~p", [?VERSION, node()]),
    Sup;
start(_, _) ->
    {error, badarg}.
```
ejabberd_sup是一个supervisor，调用start_link时，它会生成各功能组件进程，如下图所示:
```Red
------------------------------------------------

                +------------+
                |ejabberd_sup|
                +-----+------+
                      |
+---------------------+------------------------+
|                                              |
|  +-----+              +-------+              |
|  |hooks|              |c2s_sup|              |
|  +-----+              +-------+              |
|                                              |
|  +-----------+        +---------+            |
|  |node_groups|        |s2sin_sup|            |
|  +-----------+        +---------+            |
|                                              |
|  +--------------+     +----------+           |
|  |system_monitor|     |s2sout_sup|           |
|  +--------------+     +----------+           |
|                                              |
|  +------+             +-----------+          |
|  |router|             |service_sup|          |
|  +------+             +-----------+          |
|                                              |
|  +--+                 +--------+             |
|  |sm|                 |http_sup|             |
|  +--+                 +--------+             |
|                                              |
|  +---+                +-------------+        |
|  |s2s|                |http_poll_sup|        |
|  +---+                +-------------+        |
|                                              |
|  +-----+              +-------------------+  |
|  |local|              |frontend_socket_sup|  |
|  +-----+              +-------------------+  |
|                                              |
|  +-------+            +------+               |
|  |captcha|            |iq_sup|               |
|  +-------+            +------+               |
|                                              |
|  +--------+           +--------+             |
|  |listener|           |stun_sup|             |
|  +--------+           +--------+             |
|                                              |
|  +------------+       +-------------+        |
|  |receiver_sup|       |cache_tab_sup|        |
|  +------------+       +-------------+        |
|                                              |
+----------------------------------------------+
```
部分关键组件功能：

* hooks: 执行各模块注册的HOOK函数
* router: 分发用户发送的消息
* sm: session manager, 处理用户到用户的消息分发
* local: 处理发送给Server本身的消息。
* listener: supervisor进程，创建子进程监听网络端口
* receiver_sup: supervisor进程, 它为每一个TCP连接创建一个进程来接收该TCP连接的数据
* c2s_sup: supervisor进程, 它为每一个TCP连接创建一个进程来处理协议状态，并负责向TCP连接发送数据

ejabberd_listener:start_listerners/0会从配置文件(ejabberd.cfg)中获取“listen"选项。
listen配置的一般形式为:
```
{listen,
 [
  {5222, ejabberd_c2s, [
            {access, c2s},
            {shaper, c2s_shaper},
            {max_stanza_size, 65536}
               ]},
  ...
 ]
}
```
每一个端口需要指定一个处理模块。

start_listeners/0遍历listen配置中指定的所有端口，为每个端口调用start_listener/3。
{% raw %}
```erlang
start_listeners() ->
    case ejabberd_config:get_local_option(listen) of
    undefined ->
        ignore;
    Ls ->
        Ls2 = lists:map(
            fun({Port, Module, Opts}) ->
                case start_listener(Port, Module, Opts) of
                {ok, _Pid} = R -> R;
                {error, Error} ->
                throw(Error)
            end
        end, Ls),
        report_duplicated_portips(Ls),
        {ok, {{one_for_one, 10, 1}, Ls2}}
    end.
```
{% endraw %}
start_listerner最终会调用start_listener_sup/3。它通过supervisor:start_child令ejabberd_listeners进程创建子进程执行ejabberd_listener:start/3。
```erlang
start_listener_sup(Port, Module, Opts) ->
    ChildSpec = {Port,
         {?MODULE, start, [Port, Module, Opts]},
         transient,
         brutal_kill,
         worker,
         [?MODULE]},
    supervisor:start_child(ejabberd_listeners, ChildSpec).
```
ejabberd_listener:start/3依据该端口处理模块的socket_type/0函数的返回值进行相应处理。如果返回值为independent时，则表示该处理模块自行处理监听端口a。否则，调用start_dependent/3。
```erlang
start(Port, Module, Opts) ->
    %% Check if the module is an ejabberd listener or an independent listener
    ModuleRaw = strip_frontend(Module),
    case ModuleRaw:socket_type() of
    independent -> ModuleRaw:start_listener(Port, Opts);
    _ -> start_dependent(Port, Module, Opts)
    end.
```
start_dependent/3中会创建子进程执行init/3, init/3函数根据配置参数调用init_tcp/6或init_udp/6。
init_tcp/6开始监听相应端口，然后调用accept/0等待TCP连接。
```erlang
init_tcp(PortIP, Module, Opts, SockOpts, Port, IPS) ->
    ...
    Res = gen_tcp:listen(Port, [binary,
                {packet, 0},
                {active, false},
                {reuseaddr, true},
                {nodelay, true},
                {send_timeout, ?TCP_SEND_TIMEOUT},
                {keepalive, true} |
                SockOpts2]),
    case Res of
    {ok, ListenSocket} ->
        %% Inform my parent that this port was opened succesfully
        proc_lib:init_ack({ok, self()}),
        %% And now start accepting connection attempts
        accept(ListenSocket, Module, Opts);
    {error, Reason} ->
        socket_error(Reason, PortIP, Module, SockOpts, Port, IPS)
    end.
```
如果监听UDP端口，则init/3调用init_udp/6。init/6打开一个UDP端口，调用udp_recv/3等待接收UDP数据。
```erlang
init_udp(PortIP, Module, Opts, SockOpts, Port, IPS) ->
    case gen_udp:open(Port, [binary,
                 {active, false},
                 {reuseaddr, true} |
                 SockOpts]) of
    {ok, Socket} ->
        %% Inform my parent that this port was opened succesfully
        proc_lib:init_ack({ok, self()}),
        udp_recv(Socket, Module, Opts);
    {error, Reason} ->
        socket_error(Reason, PortIP, Module, SockOpts, Port, IPS)
    end.
```
至此ejabberd启动完成。

当UDP数据到来后，udp_recv/3调用该端口处理模块的udp_recv/5处理，接着递归调用udp_recv继续等待接收UDP数据。处理模块的udp_recv/5函数中需要处理返回UDP响应等逻辑。
```erlang
udp_recv(Socket, Module, Opts) ->
    case gen_udp:recv(Socket, 0) of
    {ok, {Addr, Port, Packet}} ->
        case catch Module:udp_recv(Socket, Addr, Port, Packet, Opts) of
        {'EXIT', Reason} ->
            ?ERROR_MSG("failed to process UDP packet:~n"
                   "** Source: {~p, ~p}~n"
                   "** Reason: ~p~n** Packet: ~p",
                   [Addr, Port, Reason, Packet]);
        _ ->
            ok
        end,
        udp_recv(Socket, Module, Opts);
    {error, Reason} ->
        ?ERROR_MSG("unexpected UDP error: ~s", [format_error(Reason)]),
        throw({error, Reason})
    end.
```
当有TCP连接成功后，gen_tcp:accept/1返回，accept/3根据配置中处理模块是否指定了frontend, 选择调用ejabberd_frontend_socket:start/4或ejabberd_socket:start/4, 然后递归调用accept再次等待TCP连接。
{% raw %}
```erlang
accept(ListenSocket, Module, Opts) ->
    case gen_tcp:accept(ListenSocket) of
    {ok, Socket} ->
        case {inet:sockname(Socket), inet:peername(Socket)} of
        {{ok, Addr}, {ok, PAddr}} ->
            ?INFO_MSG("(~w) Accepted connection ~w -> ~w",
                  [Socket, PAddr, Addr]);
        _ ->
            ok
        end,
        CallMod = case is_frontend(Module) of
              true -> ejabberd_frontend_socket;
              false -> ejabberd_socket
              end,
        CallMod:start(strip_frontend(Module), gen_tcp, Socket, Opts),
        accept(ListenSocket, Module, Opts);
    {error, Reason} ->
        ?INFO_MSG("(~w) Failed TCP accept: ~w",
              [ListenSocket, Reason]),
        accept(ListenSocket, Module, Opts)
    end.
```
{% endraw %}
ejabberd_socket:start/4函数根据处理模块的socket_type/0的返回值进行不同处理。5222端口的处理模块是ejabberd_c2s，该模块socket_type/0返回xml_stream。

ejabberd_socket:start/4对于xml_stream的处理的简化逻辑如下:
```erlang
RecPid = ejabberd_receiver:start(
       Socket, SockMod, none, MaxStanzaSize),

{ReceiverMod, Receiver, RecRef} =
    {ejabberd_receiver, RecPid, RecPid}

SocketData = #socket_state{sockmod = SockMod,
               socket = Socket,
               receiver = RecRef},

case Module:start({?MODULE, SocketData}, Opts) of
{ok, Pid} ->
    case SockMod:controlling_process(Socket, Receiver) of
    ok ->
        ok;
    {error, _Reason} ->
        SockMod:close(Socket)
    end,
    ReceiverMod:become_controller(Receiver, Pid);
{error, _Reason} ->
    SockMod:close(Socket)
end;
```
首先，它调用ejabberd_receiver:start/4, 它创建一个receiver进程，接着调用处理模块的start/2创建一个处理模块子进程。然后将新的receiver进程设为该TCP Socket的控制进程，以后从该Socket收到数据将以消息形式发送给该receiver进程。最后调用ejabberd_receiver:become_controller/2, 向该receiver进程发送become_controller消息。receiver进程处理该消息，生成一个XML解析状态并将处理进程Pid保存在状态中。
```erlang
handle_call({become_controller, C2SPid}, _From, State) ->
    XMLStreamState = xml_stream:new(C2SPid, State#state.max_stanza_size),
    NewState = State#state{c2s_pid = C2SPid,
               xml_stream_state = XMLStreamState},
    activate_socket(NewState),
    Reply = ok,
    {reply, Reply, NewState, ?HIBERNATE_TIMEOUT};
```
当该Socket有数据可读时，receiver进程将收到TCP消息，receiver进程调用process_data/1函数来处理收到的数据。它调用xml_stream:parse进行解析，当解析出XML元素后，它会通过gen_fsm:send_event向该TCP的处理进程发送消息。处理进程根据消息进行协议状态的转换。

当XMPP协商完成后，处理进程状态为session_established。此时收到XMPP消息，处理进程解析出From和To属性, 调用ejabberd_router:route/3分发消息。ejabberd_router:route/3调用do_route进行分发。它查询route表中是否已经注册了JID所在域名。ejabberd_local进程启动时会在route表中注册配置中所添加的域名。如果已经注册，该消息则应该由当前服务器处理，否则路由至其他Server。

处理本地域名时，do_route首先获取ejabbred_local进程的Pid,如果只有一个进程，并且该进程位于当前节点，则直接调用ejabberd_local:route/3进行处理，否则发送route消息至相应的Pid。
```erlang
LDstDomain = To#jid.lserver,
case mnesia:dirty_read(route, LDstDomain) of
[] ->
    ejabberd_s2s:route(From, To, Packet);
[R] ->
    Pid = R#route.pid,
    if
    node(Pid) == node() ->
        case R#route.local_hint of
        {apply, Module, Function} ->
            Module:Function(From, To, Packet);
        _ ->
            Pid ! {route, From, To, Packet}
        end;
    is_pid(Pid) ->
        Pid ! {route, From, To, Packet};
    true ->
        drop
    end;
...
end
```
这两种方式都会调用ejabberd_local:do_route来处理。
```erlang
do_route(From, To, Packet) ->
    ?DEBUG("local route~n\tfrom ~p~n\tto ~p~n\tpacket ~P~n",
       [From, To, Packet, 8]),
    if
    To#jid.luser /= "" ->
        ejabberd_sm:route(From, To, Packet);
    To#jid.lresource == "" ->
        {xmlelement, Name, _Attrs, _Els} = Packet,
        case Name of
        "iq" ->
            process_iq(From, To, Packet);
        "message" ->
            ok;
        "presence" ->
            ok;
        _ ->
            ok
        end;
    true ->
        {xmlelement, _Name, Attrs, _Els} = Packet,
        case xml:get_attr_s("type", Attrs) of
        "error" -> ok;
        "result" -> ok;
        _ ->
            ejabberd_hooks:run(local_send_to_resource_hook,
                       To#jid.lserver,
                       [From, To, Packet])
        end
    end.
end
```
如果"To"属性的JID指定了user, 则该消息应该分发至用户，调用ejabberd_sm:route/3进行分发。ejabberd根据JID的Resource是否为空进行了不同处理。JID的Resource为空的情况本文略过。JID的Resource不为空时，ejabberd_sm:route/3的处理逻辑如下：
```erlang
USR = {LUser, LServer, LResource},
case mnesia:dirty_index_read(session, USR, #session.usr) of
[] ->
    case Name of
    "message" ->
        route_message(From, To, Packet);
    "iq" ->
        case xml:get_attr_s("type", Attrs) of
        "error" -> ok;
        "result" -> ok;
        _ ->
            Err =
            jlib:make_error_reply(
              Packet, ?ERR_SERVICE_UNAVAILABLE),
            ejabberd_router:route(To, From, Err)
        end;
    _ ->
        ?DEBUG("packet droped~n", [])
    end;
Ss ->
    Session = lists:max(Ss),
    Pid = element(2, Session#session.sid),
    ?DEBUG("sending to process ~p~n", [Pid]),
    Pid ! {route, From, To, Packet}
end
```
它从session表中读取出该用户的信息，这些信息是用户连接上由该TCP的处理进程添加到session表中的。从中获得为该用户TCP连接的处理进程，向该进程发送route消息。处理进程处理该route消息，向该用户Socket发送消息。这便完成了一个用户到另一个用户的消息传递。

本文中ejabberd代码版本为2.1.3。
