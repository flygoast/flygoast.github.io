---
layout: post
title: "Lua源码分析: 1. Lua命令行程序实现"
date: 2016-06-10 01:49:53 +0800
comments: true
categories: Lua
---
默认情况下，Lua源码编译后会生成三个文件:

- lua: Lua解释器的命令行程序，在命令行下执行Lua脚本文件
- luac: Lua编译器，将Lua程序编译成Lua的字节码
- liblua.a: Lua语言的功能实现库，通过该库的API调用可以将Lua嵌入其他语言

Lua 是一种解释型语言，执行方式如图:
{% img /images/2016-06-10/1.png %}

首先将Lua源码编译成Lua字节码，然后由虚拟机来执行Lua字节码。

<!--more-->
本文来分析lua命令行程序的实现，主要实现位于lua.c。

Lua库使用一个虚拟栈与C程序进行交互。该栈内任意元素可通过索引直接访问。正数索引表示绝对位置，栈底为1。负数索引表示距离栈顶的偏移，栈顶为-1。如图:
{% img /images/2016-06-10/2.png %}

下面分析lua命令行程序的实现, main()函数如下:
```c
int main (int argc, char **argv) {
  int status, result;
  lua_State *L = luaL_newstate();  /* create state */
  if (L == NULL) {
    l_message(argv[0], "cannot create state: not enough memory");
    return EXIT_FAILURE;
  }
  lua_pushcfunction(L, &pmain);  /* to call 'pmain' in protected mode */
  lua_pushinteger(L, argc);  /* 1st argument */
  lua_pushlightuserdata(L, argv); /* 2nd argument */
  status = lua_pcall(L, 2, 1, 0);  /* do the call */
  result = lua_toboolean(L, -1);  /* get result */
  report(L, status);
  lua_close(L);
  return (result && status == LUA_OK) ? EXIT_SUCCESS : EXIT_FAILURE;
}
```
首先调用 luaL_newstate()创建一个`Lua State`。State结构包含交互所用的虚拟栈。然后调用 lua_pushcfunction()将pmain()函数指针压栈，调用 lua_pushinteger()和 lua_pushlightuserdata()将命令行参数个数和数组入栈。此时栈结构变为:
{% img /images/2016-06-10/3.png %}

`light userdata`表示`VOID *`指针，lua_pushlightuserdata()将指针压入栈中。

接下来调用 lua_pcall()执行pmain()函数, 执行后会将参数及函数出栈，将结果入栈。函数执行后栈结构为:

{% img /images/2016-06-10/4.png %}

最后调用 lua_close()释放State占用的内存。

下面分析 pmain函数。
pmain 首先处理栈上传入的命令行参数，之后调用luaL_openlibs()加载Lua标准库，如果没有’-E’选项，调用 handle_luainit()来执行环境变量指定的 Lua 程序。
```c
  if (!(args & has_E)) {  /* no option '-E'? */
    if (handle_luainit(L) != LUA_OK)  /* run LUA_INIT */
      return 0;  /* error running LUA_INIT */
  }
```
handle_luainit查找”LUA_INIT_5_3”或”LUA_INIT”环境变量是否被设置，若变量值以’@‘开头，变量值代表 Lua 程序文件名，否则为 Lua 程序本身。
```c
static int handle_luainit (lua_State *L) {
  const char *name = "=" LUA_INITVARVERSION;
  const char *init = getenv(name + 1);
  if (init == NULL) {
    name = "=" LUA_INIT_VAR;
    init = getenv(name + 1);  /* try alternative name */
  }
  if (init == NULL) return LUA_OK;
  else if (init[0] == '@')
    return dofile(L, init+1);
  else
    return dostring(L, init, name);
}
```
dofile()函数调用 luaL_loadfile()加载Lua 程序文件，而 dostring()函数调用 luaL_loadbuffer()加载字符串中的 Lua语句。最终调用 docall()来执行加载的 Lua 程序。
```c
/*
** Interface to 'lua_pcall', which sets appropriate message function
** and C-signal handler. Used to run all chunks.
*/
static int docall (lua_State *L, int narg, int nres) {
  int status;
  int base = lua_gettop(L) - narg;  /* function index */
  lua_pushcfunction(L, msghandler);  /* push message handler */
  lua_insert(L, base);  /* put it under function and args */
  globalL = L;  /* to be available to 'laction' */
  signal(SIGINT, laction);  /* set C-signal handler */
  status = lua_pcall(L, narg, nres, base);
  signal(SIGINT, SIG_DFL); /* reset C-signal handler */
  lua_remove(L, base);  /* remove message handler from the stack */
  return status;
}
```
docall() 先获取要执行的函数的索引，压入 msghandler 函数。再调用 lua_insert()将 msghandler函数放到要执行函数的下边，然后调用 lua_pcall 执行 lua 代码，当函数执行出错时，msghandler 会被调用。执行完成后移除 msghandler, 恢复栈结构。栈结构变化如图:
{% img /images/2016-06-10/5.png %}

接着分析 pmain流程。执行完环境变量中的 Lua 程序后，调用 runargs()处理’-e’和’-l’选项。
```c
/*
** Processes options 'e' and 'l', which involve running Lua code.
** Returns 0 if some code raises an error.
*/
static int runargs (lua_State *L, char **argv, int n) {
  int i;
  for (i = 1; i < n; i++) {
    int option = argv[i][1];
    lua_assert(argv[i][0] == '-');  /* already checked */
    if (option == 'e' || option == 'l') {
      int status;
      const char *extra = argv[i] + 2;  /* both options need an argument */
      if (*extra == '\0') extra = argv[++i];
      lua_assert(extra != NULL);
      status = (option == 'e')
               ? dostring(L, extra, "=(command line)")
               : dolibrary(L, extra);
      if (status != LUA_OK) return 0;
    }
  }
  return 1;
}
```
‘-e’的处理很简单，就是调用 dostring()来执行 Lua语句。’-l’是要引入指定的Lua 库，通过 dolibary()来完成。
```c
/*
** Calls 'require(name)' and stores the result in a global variable
** with the given name.
*/
static int dolibrary (lua_State *L, const char *name) {
  int status;
  lua_getglobal(L, "require");
  lua_pushstring(L, name);
  status = docall(L, 1, 1);  /* call 'require(name)' */
  if (status == LUA_OK)
    lua_setglobal(L, name);  /* global[name] = require return */
  return report(L, status);
}
```
dolibrary()调用 lua_getglobal()将 Lua 内置函数 require压入栈中，接着将库名称压栈，最后调用 docall()来执行栈中代码，相当于调用 Lua 代码:
```lua
require somelib
```
命令行参数若提供了脚本名称，则调用 handle_script()来处理，该函数也是通过 lua_loadfile和 docall()来完成。略过。
若是提供了’-i’选项或者没有提供脚本名称，则调用 doREPL()进入交互模式。
```c
/*
** Do the REPL: repeatedly read (load) a line, evaluate (call) it, and
** print any results.
*/
static void doREPL (lua_State *L) {
  int status;
  const char *oldprogname = progname;
  progname = NULL;  /* no 'progname' on errors in interactive mode */
  while ((status = loadline(L)) != -1) {
    if (status == LUA_OK)
      status = docall(L, 0, LUA_MULTRET);
    if (status == LUA_OK) l_print(L);
    else report(L, status);
  }
  lua_settop(L, 0);  /* clear stack */
  lua_writeline();
  progname = oldprogname;
}
```
doREPL()循环调用 loadline()来从终端加载 Lua语句。
```c
/*
** Read a line and try to load (compile) it first as an expression (by
** adding "return " in front of it) and second as a statement. Return
** the final status of load/call with the resulting function (if any)
** in the top of the stack.
*/
static int loadline (lua_State *L) {
  int status;
  lua_settop(L, 0);
  if (!pushline(L, 1))
    return -1;  /* no input */
  if ((status = addreturn(L)) != LUA_OK)  /* 'return ...' did not work? */
    status = multiline(L);  /* try as command, maybe with continuation lines */
  lua_remove(L, 1);  /* remove line from the stack */
  lua_assert(lua_gettop(L) == 1);
  return status;
}
```
loadline()调用 pushline()从终端读取一行或多行字符串并组合成一个字符串压入栈中，然后调用 addreturn()或者 multiline()调用 lua_loadbuffer()来加载 Lua 语句。
之后 doREPL()调用 docall()来执行 Lua 语句。

本文主要分析了如何使用 Lua库提供的 API 来实现Lua命令行程序，其他程序嵌入 Lua时，可以参考 lua.c来实现。主要的流程涉及 luaL_newstate(), luaL_openlibs(), lua_loadfile(), lua_loadbuffer(), lua_pcall()等调用，后续文章会来分析这些 API 的实现。

代码版本为5.3.2.
