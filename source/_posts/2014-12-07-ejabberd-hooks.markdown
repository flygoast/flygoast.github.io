---
layout: post
title: "ejabberd中hook机制分析"
date: 2014-12-07 13:37:11 +0800
comments: true
categories: XMPP
---
ejabberd中的hook机制是ejabberd XMPP模块的基础。XMPP模块需要根据需求在相应的hook点上注册自己的处理函数，在处理函数的逻辑中实现需求。ejabberd执行到hook点时，会按注册的顺序号由小到大来执行各模块所注册的处理函数。

下面来分析具体实现。

ejabberd启动时，ejabberd_sup:init/1会通过调用ejabberd_hooks:start_link/0启动名称为ejabberd_hooks的worker进程。
{% raw %}
```erlang
init([]) ->
    Hooks =
    {ejabberd_hooks,
     {ejabberd_hooks, start_link, []},
     permanent,
     brutal_kill,
     worker,
     [ejabberd_hooks]},
    ...
    {ok, {{one_for_one, 10, 1},
      [Hooks,
       ...]}}.
```
{% endraw %}
ejabberd_hooks进程初始化时执行init/1函数创建了名为hooks的ETS表。这个表用来存储在各注册点和域名下注册的hook函数。
```erlang
init([]) ->
    ets:new(hooks, [named_table]),
    {ok, #state{}}.
```
模块一般使用ejabberd_hooks:add/5注册hook函数。
```erlang
add(Hook, Host, Module, Function, Seq) ->
    gen_server:call(ejabberd_hooks, {add, Hook, Host, Module, Function, Seq}).
```
参数:

- Hook: 注册的hook点位置
- Host: 注册的域名
- Module: hook函数所在模块
- Function: hook函数名
- Seq: hook函数顺序号，顺序号越小函数越早被执行

add函数发送`add`消息给ejabberd_hooks进程。ejabberd_hooks进程调用handle_call处理消息。
{% raw %}
```erlang
handle_call({add, Hook, Host, Module, Function, Seq}, _From, State) ->
    Reply = case ets:lookup(hooks, {Hook, Host}) of
        [{_, Ls}] ->
            El = {Seq, Module, Function},
            case lists:member(El, Ls) of
            true ->
                ok;
            false ->
                NewLs = lists:merge(Ls, [El]),
                ets:insert(hooks, {{Hook, Host}, NewLs}),
                ok
            end;
        [] ->
            NewLs = [{Seq, Module, Function}],
            ets:insert(hooks, {{Hook, Host}, NewLs}),
            ok
        end,
    {reply, Reply, State};
```
{% endraw %}
handle_call首先从hooks表中查找该hook点和域名下是否已经注册了函数。若不存在，则将顺序号、模块、函数名添加到表中。若已存在，再检查是否为重复添加。如果不是，则将顺序号、模块、函数名和之前的函数信息按顺序号排序后一并添加。

如果hook函数必须在集群内特定节点上执行，可以调用ejabberd_hooks:add_dist注册。它的处理逻辑与add函数类似，只是在hooks表中多存储了node信息，此处略过。

当需要删除hook函数时(一般是模块停止时)，调用ejabberd_hooks:delete/5。
```erlang
delete(Hook, Host, Module, Function, Seq) ->
    gen_server:call(ejabberd_hooks, {delete, Hook, Host, Module, Function, Seq}).
```
delete函数发送`delete`消息给ejabberd_hooks进程。进程执行handle_call处理。
{% raw %}
```erlang
handle_call({delete, Hook, Host, Module, Function, Seq}, _From, State) ->
    Reply = case ets:lookup(hooks, {Hook, Host}) of
        [{_, Ls}] ->
            NewLs = lists:delete({Seq, Module, Function}, Ls),
            ets:insert(hooks, {{Hook, Host}, NewLs}),
            ok;
        [] ->
            ok
        end,
    {reply, Reply, State};
```
{% endraw %}
handle_call从hooks表中获取注册在该hook点和域名上的所有函数，从中删除指定的函数，再将结果保存。
删除注册在特定节点上的函数要使用delete_dist，处理逻辑类似，略过。

ejabberd执行到hook点时会调用ejabberd_hooks:run/3或ejabberd_hooks:run_fold/4来执行注册的HOOK函数。如果这个hook点不关心各hook函数的返回结果，则调用run函数，否则调用run_fold函数。
首先看run函数:
```erlang
run(Hook, Host, Args) ->
    case ets:lookup(hooks, {Hook, Host}) of
    [{_, Ls}] ->
        run1(Ls, Hook, Args);
    [] ->
        ok
    end.
```
run函数从hooks表中查找注册在该hook点和域名上的所有函数，然后调用run1/3。

```erlang
run1([{_Seq, Module, Function} | Ls], Hook, Args) ->
    Res = if is_function(Function) ->
          catch apply(Function, Args);
         true ->
          catch apply(Module, Function, Args)
      end,
    case Res of
    {'EXIT', Reason} ->
        ?ERROR_MSG("~p~nrunning hook: ~p",
               [Reason, {Hook, Args}]),
        run1(Ls, Hook, Args);
    stop ->
        ok;
    _ ->
        run1(Ls, Hook, Args)
    end.
```
run1依次执行注册的hook函数，如果某个hook函数返回`stop`, 则run1结束返回，之后的hook函数不再被执行。

如果需要执行的是注册在某个节点上的hook函数，则通过rpc:call在该节点上执行函数，其他逻辑类似。
```erlang
run1([{_Seq, Node, Module, Function} | Ls], Hook, Args) ->
    case rpc:call(Node, Module, Function, Args, ?TIMEOUT_DISTRIBUTED_HOOK) of
    timeout ->
        ?ERROR_MSG("Timeout on RPC to ~p~nrunning hook: ~p",
               [Node, {Hook, Args}]),
        run1(Ls, Hook, Args);
    {badrpc, Reason} ->
        ?ERROR_MSG("Bad RPC error to ~p: ~p~nrunning hook: ~p",
               [Node, Reason, {Hook, Args}]),
        run1(Ls, Hook, Args);
    stop ->
        ?INFO_MSG("~nThe process ~p in node ~p ran a hook in node ~p.~n"
              "Stop.", [self(), node(), Node]), % debug code
        ok;
    Res ->
        ?INFO_MSG("~nThe process ~p in node ~p ran a hook in node ~p.~n"
              "The response is:~n~s", [self(), node(), Node, Res]), % debug code
        run1(Ls, Hook, Args)
    end;
```

再来看run_fold函数。和run函数相比，run_fold还需要一个参数，表示默认的返回结果。
```erlang
run_fold(Hook, Host, Val, Args) ->
    case ets:lookup(hooks, {Hook, Host}) of
    [{_, Ls}] ->
        run_fold1(Ls, Hook, Val, Args);
    [] ->
        Val
    end.
```
run_fold首先找到注册在该hook点和域名上的所有函数，如果没有，则返回默认结果。否则调用run_fold1。
```erlang
run_fold1([{_Seq, Module, Function} | Ls], Hook, Val, Args) ->
    Res = if is_function(Function) ->
          catch apply(Function, [Val | Args]);
         true ->
          catch apply(Module, Function, [Val | Args])
      end,
    case Res of
    {'EXIT', Reason} ->
        ?ERROR_MSG("~p~nrunning hook: ~p",
               [Reason, {Hook, Args}]),
        run_fold1(Ls, Hook, Val, Args);
    stop ->
        stopped;
    {stop, NewVal} ->
        NewVal;
    NewVal ->
        run_fold1(Ls, Hook, NewVal, Args)
    end.
```
run_fold1会将传入的结果`Val`(或者来自默认结果，或者来自前一hook函数的返回结果)和参数`Args`组成新的lists做为参数传给hook函数，依次递归调用。若某hook函数返回`stop`，结束递归调用，返回`stopped`。若hook函数返回`{stop, NewVal}`,则直接返回该hook函数的结果`NewVal`。这两种情况下，其余的hook函数不再被执行。否则，返回结果做为Val参数再次递归调用run_fold1。

注册在特定节点上的函数处理逻辑类似，只是使用rpc:call在相应节点上执行，略过。

具体的hook点和hook函数原型可以参考官方文档:

https://www.process-one.net/en/wiki/ejabberd_events_and_hooks/

这个文档写地不是很详细。不确定的地方需要参考源码。

当这些内置hook点不能满足需求时，可以在ejabberd中合适位置调用ejabberd_hooks:run或ejabberd_hooks:run_fold添加hook点。

如:
```erlang
ejabberd_hooks:run(dummy_hook, []),
```

另外，需要注意的有:
ejabberd执行某些hook点时，调用不同参数版本的run或run_fold。这种情况Host参数为`global`。注册这种hook点时，Host参数也应该使用`global`。
如:
```erlang
    case ejabberd_hooks:run_fold(filter_packet,
                 {OrigFrom, OrigTo, OrigPacket}, []) of
```
```erlang
ejabberd_hooks:add(filter_packet, global, ?MODULE, on_filter_packet, 120),
```

注: 文中代码版本为:ejabberd-2.1.13。
