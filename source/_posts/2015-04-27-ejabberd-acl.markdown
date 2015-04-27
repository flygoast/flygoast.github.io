---
layout: post
title: "ejabberd中ACL实现"
date: 2015-04-27 16:10:08 +0800
comments: true
categories: XMPP
---

ejbberd中多个模块或组件都允许管理员在配置文件中基于用户JID进行访问限制, 如在ejabberd_c2s模块中禁用某用户.

ejabberd实现了一套通用的ACL机制来满足各模块的需求.

配置方法如下:

* 添加acl规则, 如:
```erlang
{acl, blocked, {user, "test"}}.
```
acl元组第2个元素为该条acl规则的名称, 第3个元素为JID的过滤规则. 示例中的过滤规则表示用户JID的User部分为"test".

* 添加access规则, 如:
```erlang
{access, c2s, [{deny, blocked}, {allow, all}]}.
```
access元组第2个元素为access规则的名称, 第3个元素中的每个元素为一个指定值和一个acl规则名字.
当JID满足某条acl规则时, 该条access规则的值则为该acl规则的对应值. 示例中, 当用户JID中的User部分为"test"时, access规则c2s值为`deny`, 否则为`allow`.

* 为模块或组件指定access规则(不同的模块或组件所用的配置指令可能不同) 如:
```erlang
{5222, ejabberd_c2s, [
                        {access, c2s},
                        {shaper, c2s_shaper},
                        {max_stanza_size, 65536}
                       ]}.
```
按示例配置后, ejabberd_c2s会根据access规则c2s的值禁用相应用户, 如JID中User为"test"的用户全部被禁用.

下边来看具体实现:

ejabberd启动时, `ejabberd_app:start/2`会调用`acl:start/0`.
```erlang
start() ->
    mnesia:create_table(acl,
            [{disc_copies, [node()]},
             {type, bag},
             {attributes, record_info(fields, acl)}]),
    mnesia:add_table_copy(acl, node(), ram_copies),
    ok.
```
start函数创建了一个名为`acl`的表,来存储各条acl规则, `{type, bag}`使表中可以保存多个KEY相同的record, 即可以保存多个acl名称相同的acl规则, acl的record定义为:
```erlang
-record(acl, {aclname, aclspec}).
```
如配置中可以添加多条相同名字的ACL规则:
```erlang
{acl, admin, {user, "aleksey", "localhost"}}.
{acl, admin, {user, "ermine", "example.org"}}.
{acl, admin, {user, "admin", "localhost"}}.
```
ejabberd启动加载配置文件时, ejabberd_config模块会将相应`{acl, ...}`配置写入`acl`表中, `{access, ...}`配置写入`config`表中, 具体过程请参考<ejabberd配置模块分析>一文.

当各模块或组件需要基于JID对访问进行限制时, 使用`acl:match_rule/3`来获取所配置的access规则的值, 进而根据所得的access规则的值进行后续逻辑处理. 如ejabberd_c2s模块完成XMPP的BIND流程后, 为用户建立session前, 调用`match_rule/3`来获取指定的access规则的值, 如果值为`allow`则允许请用户请求继续,否则返回禁止信息.
```erlang
case acl:match_rule(StateData#state.server,
                    StateData#state.access, JID) of
allow ->
    ...
_ ->
    ...
end.
```
来看acl:match_rule/3实现:
```erlang
match_rule(global, Rule, JID) ->
    case Rule of
    all -> allow;
    none -> deny;
    _ ->
        case ejabberd_config:get_global_option({access, Rule, global}) of
        undefined ->
            deny;
        GACLs ->
            match_acls(GACLs, JID, global)
        end
    end;
```
`all`和`none`为两个特殊的access规则, 对应的值分别为`allow`和`deny`. 若规则不为这两个, 则调用
`ejabberd_config:get_global_option/1`来获取该条access规则所指定的一系列acl规则.
如果没有则返回`deny`, 否则接着调用`match_acls`.
```erlang
match_acls([], _, _Host) ->
    deny;
match_acls([{Access, ACL} | ACLs], JID, Host) ->
    case match_acl(ACL, JID, Host) of
    true ->
        Access;
    _ ->
        match_acls(ACLs, JID, Host)
    end.
```
`match_acls`对ACL列表进行递归处理, 对每条acl规则调用`match_acl`. 如果`match_acl`返回`true`, 则递归结束,返回该acl规则所对应的值, 如果没有任何一条acl规则满足, 则返回`deny`.
来看`match_acl`实现, 简化后的逻辑如下:
```erlang
match_acl(ACL, JID, Host) ->
    case ACL of
    all -> true;
    none -> false;
    _ ->
        {User, Server, Resource} = jlib:jid_tolower(JID),
        lists:any(fun(#acl{aclspec = Spec}) ->
                  case Spec of
                  all ->
                      true;
                  {user, U} ->
                      (U == User)
                      andalso
                        ((Host == Server) orelse
                         ((Host == global) andalso
                          lists:member(Server, ?MYHOSTS)));
                 ...

                  WrongSpec ->
                      ?ERROR_MSG(
                     "Wrong ACL expression: ~p~n"
                     "Check your config file and reload it with the override_acls option enabled",
                     [WrongSpec]),
                      false
                  end
              end,
              ets:lookup(acl, {ACL, global}) ++
              ets:lookup(acl, {ACL, Host}))
    end.
```
`match_acl`从`acl`表中查出该名称所有的acl规则(可能有多个), 只要其中一条acl规则匹配了用户的JID, `match_acl`就返回`true`.

具体的acl过滤条件有多个, 如
- user
- server
- user_glob
- user_regexp
等, 具体涵义请参考acl.erl源码.

注意: access规则的值可以为任意值,不一定要是allow或者deny, 这些由特定的模块自行处理.
```erlang
{access, max_user_sessions, [{10, all}]}.
```
ejabberd中acl功能实现的简单却通用, 类似逻辑可以借鉴到我们的其他项目中.
