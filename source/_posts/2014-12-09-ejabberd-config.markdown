---
layout: post
title: "ejabberd配置模块分析"
date: 2014-12-09 19:24:12 +0800
comments: true
categories: XMPP
---
ejabberd_config模块负责加载ejabberd配置文件，存储相应的配置选项，并提供添加和获取配置选项的API。

比如, ejabberd_app:start_modules函数会使用ejabber_config:get_local_option获取配置文件中的modules选项:
{% raw %}
```erlang
%% Start all the modules in all the hosts
start_modules() ->
    lists:foreach(
      fun(Host) ->
          case ejabberd_config:get_local_option({modules, Host}) of
          undefined ->
              ok;
          Modules ->
              lists:foreach(
            fun({Module, Args}) ->
                gen_mod:start_module(Host, Module, Args)
            end, Modules)
          end
      end, ?MYHOSTS).
```
{% endraw %}
下面分析ejbberd_config模块实现。

ejabberd启动时，ejabberd_app:start/0会调用ejabberd_config:start/0。
{% raw %}
```erlang
start() ->
    mnesia:create_table(config,
            [{disc_copies, [node()]},
             {attributes, record_info(fields, config)}]),
    mnesia:add_table_copy(config, node(), ram_copies),
    mnesia:create_table(local_config,
            [{disc_copies, [node()]},
             {local_content, true},
             {attributes, record_info(fields, local_config)}]),
    mnesia:add_table_copy(local_config, node(), ram_copies),
    Config = get_ejabberd_config_path(),
    load_file(Config),
    %% This start time is used by mod_last:
    add_local_option(node_start, now()),
    ok.
```
{% endraw %}
start函数首先创建config和local_config两个mnesia表，接着调用get_ejabberd_config_path获取配置文件路径。
{% raw %}
```erlang
get_ejabberd_config_path() ->
    case application:get_env(config) of
    {ok, Path} -> Path;
    undefined ->
        case os:getenv("EJABBERD_CONFIG_PATH") of
        false ->
            ?CONFIG_PATH;
        Path ->
            Path
        end
    end.
```
{% endraw %}
get_ejabberd_config_path首先使用application:get_env从ejabberd.app的env或erlang命令行中的config选项中获取值:
如：
```
{env, [{config, "/etc/ejabberd/ejabberd.cfg"]}
```
或者:
```
erl -config "/path/to/ejabberd.cfg"
```
如果没有设置这两个选项，则尝试从系统环境变量"EJABBERD_CONFIG_PATH"读取文件路径。ejabberdctl会设置该环境变量。若也没有设置该环境变量，get_ejabberd_config_path则返回`?CONFIG_PATH`, 这个宏被定义为"ejabberd.cfg"。
```erlang
-define(CONFIG_PATH, "ejabberd.cfg").
```
接下来, start函数调用load_file来加载配置文件:
```erlang
load_file(File) ->
    Terms = get_plain_terms_file(File),
    State = lists:foldl(fun search_hosts/2, #state{}, Terms),
    Terms_macros = replace_macros(Terms),
    Res = lists:foldl(fun process_term/2, State, Terms_macros),
    set_opts(Res).
```
load_file首先调用get_plain_terms_file来获取所有配置选项的列表。
```erlang
get_plain_terms_file(File1) ->
    File = get_absolute_path(File1),
    case file:consult(File) of
    {ok, Terms} ->
        include_config_files(Terms);
    {error, {LineNumber, erl_parse, _ParseMessage} = Reason} ->
        ExitText = describe_config_problem(File, Reason, LineNumber),
        ?ERROR_MSG(ExitText, []),
        exit_or_halt(ExitText);
    {error, Reason} ->
        ExitText = describe_config_problem(File, Reason),
        ?ERROR_MSG(ExitText, []),
        exit_or_halt(ExitText)
    end.
```
我们来看get_plain_terms_file实现。首先，调用get_absolute_path，期望得到配置文件的绝对路径。不过，get_absolute_path实现上存在BUG:
```erlang
get_absolute_path(File) ->
    case filename:pathtype(File) of
    absolute ->
        File;
    relative ->
        Config_path = get_ejabberd_config_path(),
        Config_dir = filename:dirname(Config_path),
        filename:absname_join(Config_dir, File)
    end.
```
当File为相对路径时，使用filename:absname_join不能得到绝对路径。我提了一个PATCH，使用当前目录来转成绝对路径。官方已接受。
PATCH地址：
https://github.com/processone/ejabberd/commit/62ccf1cf0e13954ee5207bc6288afbc669247d14

接着，get_plain_terms_file调用file:consult读取配置文件中的所有Erlang Terms到列表中，再调用include_config_files函数来处理include_config_file选项。
```erlang
include_config_files([{include_config_file, Filename, Options} | Terms], Res) ->
    Included_terms = get_plain_terms_file(Filename),
    Disallow = proplists:get_value(disallow, Options, []),
    Included_terms2 = delete_disallowed(Disallow, Included_terms),
    Allow_only = proplists:get_value(allow_only, Options, all),
    Included_terms3 = keep_only_allowed(Allow_only, Included_terms2),
    include_config_files(Terms, Res ++ Included_terms3);
```
include_config_file选项格式为：
```erlang
{include_config_file, [{disallow, foo}, {allow_only, bar}], "/path/to/included_config"}.
```
include_config_files递归调用get_plain_terms_file获取被引用的配置文件中所有配置，接着检查include_config_file选项中是否有disallow选项。如果有，调用delete_disallowed将disallow指定的配置选项从被引用文件的配置列表中删除。接着检查其中是否存在allow_only选项，如果有，则调用keep_only_allowed只保留下allow_only中指定的配置，将其和外部配置合并，再递归调用include_config_files/2处理剩余的选项，最终返回所有配置文件中所有选项列表。
```erlang
include_config_files([], Res) ->
    Res;
```
load_file接着遍历配置列表调用search_host, 最终调用add_option来添加hosts选项。
{% raw %}
```erlang
add_option(Opt, Val, State) ->
    Table = case Opt of
        hosts ->
            config;
        language ->
            config;
        _ ->
            local_config
        end,
    case Table of
    config ->
        State#state{opts = [#config{key = Opt, value = Val} |
                State#state.opts]};
    local_config ->
        case Opt of
        {{add, OptName}, Host} ->
            State#state{opts = compact({OptName, Host}, Val,
                           State#state.opts, [])};
        _ ->
            State#state{opts = [#local_config{key = Opt, value = Val} |
                    State#state.opts]}
        end
    end.
```
{% endraw %}

add_option将指定选项以Key/Value形式添加进状态结构的opts域中。其中，hosts和language使用记录config, 其他选项使用local_config。这与最初创建的MNESIA表相对应。
状态结构如下:
{% raw %}
```erlang
-record(state, {opts = [],
        hosts = [],
        override_local = false,
        override_global = false,
        override_acls = false}).
```
{% endraw %}
接下来，load_file调用replace_macros来替换配置中的宏为相应的值。我们来看replace_macros实现。
{% raw %}
```erlang
replace_macros(Terms) ->
    {TermsOthers, Macros} = split_terms_macros(Terms),
    replace(TermsOthers, Macros).
```
{% endraw %}
首先, 调用split_terms_macros将宏选项和其他选项分开。
{% raw %}
```erlang
split_terms_macros(Terms) ->
    lists:foldl(
      fun(Term, {TOs, Ms}) ->
          case Term of
          {define_macro, Key, Value} ->
              case is_atom(Key) and is_all_uppercase(Key) of
              true ->
                  {TOs, Ms++[{Key, Value}]};
              false ->
                  exit({macro_not_properly_defined, Term})
              end;
          Term ->
              {TOs ++ [Term], Ms}
          end
      end,
      {[], []},
      Terms).
```
{% endraw %}
宏定义选项格式为:
```erlang
{define_macro, 'KEY', bar}.
```
其中key必须为atom类型且必须全部为大写字母，得到的宏选项列表为`{key, value}`格式的列表。
接着, replace_macros调用replace/2。
```erlang
replace([], _) ->
    [];
replace([Term|Terms], Macros) ->
    [replace_term(Term, Macros) | replace(Terms, Macros)].
```
replace通过递归对每个选项调用replace_term/2。
```erlang
replace_term(Key, Macros) when is_atom(Key) ->
    case is_all_uppercase(Key) of
    true ->
        case proplists:get_value(Key, Macros) of
        undefined -> exit({undefined_macro, Key});
        Value -> Value
        end;
    false ->
        Key
    end;
replace_term({use_macro, Key, Value}, Macros) ->
    proplists:get_value(Key, Macros, Value);
replace_term(Term, Macros) when is_list(Term) ->
    replace(Term, Macros);
replace_term(Term, Macros) when is_tuple(Term) ->
    List = tuple_to_list(Term),
    List2 = replace(List, Macros),
    list_to_tuple(List2);
replace_term(Term, _) ->
    Term.
```
replace_term遍历选项中的所有子项，如果在宏列表中查找到相应的值，则替换该子项为找到的值。
另外，如果配置中某子项指定了{use_macro, Key, Value}这种格式的配置，在替换时优先从宏列表中查找相应的值，找不到再使用use_macro指定的Value来替换。

至此，获得了所有配置选项的列表，接下来对每个选项调用process_term。

选项存储主要分为3种类型, process_term分别进行不同处理:

* 在状态结构中以独立域进行存储，如`override_global`选项:
```erlang
    override_global ->
        State#state{override_global = true};
```
* 以选项名做为key进行存储, 如`max_fsm_queue`选项:
```erlang
    {max_fsm_queue, N} ->
        add_option(max_fsm_queue, N, State);
```
* 以选项名和Host一起做为key进行存储，如`domain_certfile`选项:
```erlang
    {domain_certfile, Domain, CertFile} ->
        case ejabberd_config:is_file_readable(CertFile) of
        true -> add_option({domain_certfile, Domain}, CertFile, State);
        false ->
            ErrorText = "There is a problem in the configuration: "
            "the specified file is not readable: ",
            throw({error, ErrorText ++ CertFile})
        end;
```
其中由host_config指定的绝大多数配置选项都以这种方式存储:
```erlang
    {host_config, Host, Terms} ->
        lists:foldl(fun(T, S) -> process_host_term(T, Host, S) end,
            State, Terms);
```
process_terms对于没有明确列出的选项，给配置的每个HOST都调用process_host_term添加了一个以选项名和HOST一起做为Key的配置选项。
```erlang
    {_Opt, _Val} ->
        lists:foldl(fun(Host, S) -> process_host_term(Term, Host, S) end,
            State, State#state.hosts)
```
这样，如果我们自定义配置选项dummy_config：
```erlang
{dummy_config, [foo, bar]}.
```
查询时应使用如下提供Host参数的语句:
```erlang
ejabberd_config:get_local_option({dummy_config, Host})
```

此外，`acl`, `access`和`shaper`三个选项比较特殊。
`acl`存储acl记录结构在状态结构的opts域中, 在set_opts中这些记录会被写入acl表中。
`access`和`shaper`选项的KEY中除了选项名，HOST，还包括了规则名称。

至此，load_file将所有选项保存到了状态结构的opts域中，最后调用set_opts进行存储:
```erlang
set_opts(State) ->
    Opts = lists:reverse(State#state.opts),
    F = fun() ->
        if
            State#state.override_global ->
            Ksg = mnesia:all_keys(config),
            lists:foreach(fun(K) ->
                          mnesia:delete({config, K})
                      end, Ksg);
            true ->
            ok
        end,
        if
            State#state.override_local ->
            Ksl = mnesia:all_keys(local_config),
            lists:foreach(fun(K) ->
                          mnesia:delete({local_config, K})
                      end, Ksl);
            true ->
            ok
        end,
        if
            State#state.override_acls ->
            Ksa = mnesia:all_keys(acl),
            lists:foreach(fun(K) ->
                          mnesia:delete({acl, K})
                      end, Ksa);
            true ->
            ok
        end,
        lists:foreach(fun(R) ->
                      mnesia:write(R)
                  end, Opts)
    end,
    case mnesia:transaction(F) of
    ...
    end.
```
如果配置了`override_global`, `override_local`, `override_acls`选项，set_opts首先会分别删除表config, local_config和acl中的所有内容。接着分别将状态结构opts域中的配置写入config, local_config和acl三个表中。

配置加载过程结束。

ejabberd_config模块提供了添加和查询选项的API:
{% raw %}
```erlang
add_global_option(Opt, Val) ->
    mnesia:transaction(fun() ->
                   mnesia:write(#config{key = Opt,
                            value = Val})
               end).

add_local_option(Opt, Val) ->
    mnesia:transaction(fun() ->
                   mnesia:write(#local_config{key = Opt,
                              value = Val})
               end).

get_global_option(Opt) ->
    case ets:lookup(config, Opt) of
    [#config{value = Val}] ->
        Val;
    _ ->
        undefined
    end.

get_local_option(Opt) ->
    case ets:lookup(local_config, Opt) of
    [#local_config{value = Val}] ->
        Val;
    _ ->
        undefined
    end.
```
{% endraw %}
add_global_option和add_local_option分别向config和local_config表中添加选项记录。get_global_option和get_local_option直接使用`ets:lookup`查找相应配置。这是由于mnesia底层由ETS实现，直接使用`ets:lookup`性能会更高。不过，我个人不太欣赏这种写法。
