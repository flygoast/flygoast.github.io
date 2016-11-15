---
layout: post
title: "Lua代码加密"
date: 2016-11-15 23:31:43 +0800
comments: true
categories: Lua
---
我们的程序很多业务逻辑由Lua实现，为了防止业务逻辑被曝光，需要对Lua代码进行加密。

我们有两种思路：

* 自定义字节码: Lua库可以直接调用编译后生成的Lua字节码，因而我们可以将源码编译成字节码对外提供。但是因为Lua是开源的，可以通过工具将字节码反编译回源码。我们可以自定义字节码，加大反编译的难度。
* 将Lua源码文件加密，在Lua编译字节码前，对源码文件进行解密

本文主要介绍第二种思路的实现。
我们的程序使用LuaJIT来执行Lua代码，因而以LuaJIT来说明。
<!--more-->
我们的C程序使用luaL_loadfile()函数来加载Lua源码。
```c
LUALIB_API int luaL_loadfile(lua_State *L, const char *filename)
{
  return luaL_loadfilex(L, filename, NULL);
}
```
可以看到luaL_loadfile是luaL_loadfilex()的简单封装，再来看luaL_loadfilex实现:
```c
LUALIB_API int luaL_loadfilex(lua_State *L, const char *filename,
                            const char *mode)
{
  FileReaderCtx ctx;
  int status;
  const char *chunkname;
  if (filename) {
    ctx.fp = fopen(filename, "rb");
    if (ctx.fp == NULL) {
      lua_pushfstring(L, "cannot open %s: %s", filename, strerror(errno));
      return LUA_ERRFILE;
    }
    chunkname = lua_pushfstring(L, "@%s", filename);
  } else {
    ctx.fp = stdin;
    chunkname = "=stdin";
  }
  status = lua_loadx(L, reader_file, &ctx, chunkname, mode);
  if (ferror(ctx.fp)) {
    L->top -= filename ? 2 : 1;
    lua_pushfstring(L, "cannot read %s: %s", chunkname+1, strerror(errno));
    if (filename)
      fclose(ctx.fp);
    return LUA_ERRFILE;
  }
  if (filename) {
    L->top--;
    copyTV(L, L->top-1, L->top);
    fclose(ctx.fp);
  }
  return status;
}
```
当从文件加载Lua代码时，luaL_loadfilex()调用fopen()打开文件，将文件流指针存储在ctx.fp中，再调用lua_loadx()编译Lua源码。

我们可以在这里将源码文件进行解密，再打开解密后的文件，用解密后的文件流指针替换密文文件流指针，再调用lua_loadx()完成编译。我们使用一个特定文件头来标识密文文件。为了兼容未加密的文件，我们首先判断是否为密文文件。若不是，则直接将文件指针恢复到文件头。否则，调用decrypt_file()解密文件并打开解密后的文件，将文件指针存储于ctx.fp中。
修改后的源码为:
```c
LUALIB_API int luaL_loadfilex(lua_State *L, const char *filename,
			      const char *mode)
{
  FileReaderCtx ctx;
  int status;
  const char *chunkname;
  char        file_header[FILE_HEADER_LEN];
  size_t      sz;

  if (filename) {
    ctx.fp = fopen(filename, "rb");
    if (ctx.fp == NULL) {
      lua_pushfstring(L, "cannot open %s: %s", filename, strerror(errno));
      return LUA_ERRFILE;
    }

    /* check file header */
    sz = fread(file_header, 1, FILE_HEADER_LEN, ctx.fp);
    if (sz == FILE_HEADER_LEN) {
      if (memcmp(file_header, FILE_HEADER, FILE_HEADER_LEN - 1) == 0) {
        /* decrypt file */
        ctx.fp = decrypt_file(ctx.fp);
      }
    }
    fseek(ctx.fp, 0L, SEEK_SET);

    chunkname = lua_pushfstring(L, "@%s", filename);
  } else {
    ctx.fp = stdin;
    chunkname = "=stdin";
  }
  status = lua_loadx(L, reader_file, &ctx, chunkname, mode);
  if (ferror(ctx.fp)) {
    L->top -= filename ? 2 : 1;
    lua_pushfstring(L, "cannot read %s: %s", chunkname+1, strerror(errno));
    if (filename)
      fclose(ctx.fp);
    return LUA_ERRFILE;
  }
  if (filename) {
    L->top--;
    copyTV(L, L->top-1, L->top);
    fclose(ctx.fp);
  }
  return status;
}
```

接着来看decrypt_file()实现:
```c
static FILE *decrypt_file(FILE *ofp)
{
  int            fd, len;
  size_t         sz;
  FILE          *fp;
  unsigned char *buf, *obuf;
  char           file_temp[] = "/tmp/luajit-XXXXXX";

  fp = NULL;
  buf = NULL;
  obuf = NULL;
  fd = -1;

  fseek(ofp, 0L, SEEK_END);
  sz = ftell(ofp);

  obuf = malloc(sz);
  if (obuf == NULL) {
    goto failed;
  }

  fseek(ofp, 0L, SEEK_SET);
  if (fread(obuf, 1, sz, ofp) < sz) {
    goto failed;
  }

  fclose(ofp);
  ofp = NULL;

  buf = blowfish_decrypt(obuf + FILE_HEADER_LEN,
                         sz - FILE_HEADER_LEN,
                         g_key,
                         g_iv,
                         &len);
  if (buf == NULL) {
    goto failed;
  }

  free(obuf);
  obuf = NULL;

  fd = mkstemp(file_temp);
  if (fd < 0) {
    goto failed;
  }
  unlink(file_temp);

  fp = fdopen(fd, "wb+");
  if (fp == NULL) {
    goto failed;
  }
  fwrite(buf, 1, len, fp);
  free(buf);
  buf = NULL;

  return fp;

failed:

  if (fp) {
    fclose(fp);
  }

  if (ofp) {
    fclose(ofp);
  }

  if (obuf) {
    free(obuf);
  }

  if (buf) {
    free(buf);
  }

  return NULL;
}
```
首先，调用解密函数将源码文件解密到内存BUFFER中。然后，调用mkstemp()创建一个临时文件，并调用unlink()将其删除，避免解密后的文件被其他用户或程序获取到，也避免留下没用的临时文件。之后，接着将解密出的内容写入临时文件，最后返回临时文件的文件指针，完成文件指针的替换。

除了临时文件，也可以使用PIPE()来实现。将解密后的内容写入PIPE写端，调用fdopen()将PIPE读端构造成文件流指针返回。由于PIPE的大小有限制，一般为64K，不能处理大于64K的文件。因而我们采用临时文件的方法来实现。
