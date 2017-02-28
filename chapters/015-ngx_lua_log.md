> 以下代码均出自 lua-nginx-module v0.10.7 版本

> 这篇文章主要深入介绍 `lua-nginx-module` 的 `ngx.log` 和 `print` 这两个API 和一些相关的常量的工作过程和原理。 <br>
> ngx.log 这个 API 相信大家都是十分熟悉的，它的功能就是打印错误日志，在排查一些疑难杂症的时候十分有用。
> 

###  ngx.log 注入过程的调用栈

注入 API 的过程发生在 postconfiguration 过程中，也就是整个配置解析完成以后。相关调用栈为：

> ngx_http_lua_inject_log_api <br>
> ngx_http_lua_inject_ngx_api <br>
> ngx_http_lua_init_globals <br>
> ngx_http_lua_new_state <br>
> ngx_http_lua_init_vm <br>
> ngx_http_lua_init

忽略其他的细节，主要来讲讲 `ngx_http_lua_inject_log_api` 这个函数。

### ngx_http_lua_inject_log_api

这个函数只有短短几行代码，来看看它的源码：

```c
void
ngx_http_lua_inject_log_api(lua_State *L)
{
    ngx_http_lua_inject_log_consts(L);

    lua_pushcfunction(L, ngx_http_lua_ngx_log);
    lua_setfield(L, -2, "log");

    lua_pushcfunction(L, ngx_http_lua_print);
    lua_setglobal(L, "print");
}
```

这个函数的功能一目了然：

* 注入一些 `ngx.log` 相关的常量（`ngx_http_lua_inject_log_consts`）
* 注入函数 `ngx.log`
* 注入函数 `print`（不是 `ngx.print`）

### ngx_http_lua_inject_log_consts

我们来看看相关的常量注入过程

```c
static void
ngx_http_lua_inject_log_consts(lua_State *L)
{
    /* nginx log level constants */
    lua_pushinteger(L, NGX_LOG_STDERR);
    lua_setfield(L, -2, "STDERR");

    lua_pushinteger(L, NGX_LOG_EMERG);
    lua_setfield(L, -2, "EMERG");

    lua_pushinteger(L, NGX_LOG_ALERT);
    lua_setfield(L, -2, "ALERT");

    lua_pushinteger(L, NGX_LOG_CRIT);
    lua_setfield(L, -2, "CRIT");

    lua_pushinteger(L, NGX_LOG_ERR);
    lua_setfield(L, -2, "ERR");

    lua_pushinteger(L, NGX_LOG_WARN);
    lua_setfield(L, -2, "WARN");

    lua_pushinteger(L, NGX_LOG_NOTICE);
    lua_setfield(L, -2, "NOTICE");

    lua_pushinteger(L, NGX_LOG_INFO);
    lua_setfield(L, -2, "INFO");

    lua_pushinteger(L, NGX_LOG_DEBUG);
    lua_setfield(L, -2, "DEBUG");
}
```

这个函数仅仅只是把 9 个 Nginx 的日志级别注入到了 `ngx` 这个表里（在调用 `ngx_http_lua_inject_log_api` 之前，`ngx` 这个表就已经在栈顶了）。


### ngx_http_lua_ngx_log

这个函数和 `ngx.log` 关联，实际上我们在 Lua 代码里使用了 `ngx.log`，C 层面就是调用这个函数来完成相关的工作。

```c
/**
 * Wrapper of nginx log functionality. Take a log level param and varargs of
 * log message params.
 *
 * @param L Lua state pointer
 * @retval always 0 (don't return values to Lua)
 * */
int
ngx_http_lua_ngx_log(lua_State *L)
{
    ngx_log_t                   *log;
    ngx_http_request_t          *r;
    const char                  *msg;
    int                          level;

    r = ngx_http_lua_get_req(L);

    if (r && r->connection && r->connection->log) {
        log = r->connection->log;

    } else {
        log = ngx_cycle->log;
    }

    level = luaL_checkint(L, 1);
    if (level < NGX_LOG_STDERR || level > NGX_LOG_DEBUG) {
        msg = lua_pushfstring(L, "bad log level: %d", level);
        return luaL_argerror(L, 1, msg);
    }

    /* remove log-level param from stack */
    lua_remove(L, 1);

    return log_wrapper(log, "[lua] ", (ngx_uint_t) level, L);
}
```

这个函数检查了输入的日志级别是否合法，然后调用了 `log_wrapper`。

### log_wrapper

```c
static int
log_wrapper(ngx_log_t *log, const char *ident, ngx_uint_t level,
    lua_State *L)
{
    u_char              *buf;
    u_char              *p, *q;
    ngx_str_t            name;
    int                  nargs, i;
    size_t               size, len;
    size_t               src_len = 0;
    int                  type;
    const char          *msg;
    lua_Debug            ar;

    if (level > log->log_level) {
        return 0;
    }

#if 1
    /* add debug info */

    lua_getstack(L, 1, &ar);
    lua_getinfo(L, "Snl", &ar);

    /* get the basename of the Lua source file path, stored in q */
    name.data = (u_char *) ar.short_src;
    if (name.data == NULL) {
        name.len = 0;

    } else {
        p = name.data;
        while (*p != '\0') {
            if (*p == '/' || *p == '\\') {
                name.data = p + 1;
            }
            p++;
        }

        name.len = p - name.data;
    }

#endif

    nargs = lua_gettop(L);

    size = name.len + NGX_INT_T_LEN + sizeof(":: ") - 1;

    if (*ar.namewhat != '\0' && *ar.what == 'L') {
        src_len = ngx_strlen(ar.name);
        size += src_len + sizeof("(): ") - 1;
    }
    
    for (i = 1; i <= nargs; i++) {
        type = lua_type(L, i);
        switch (type) {
            case LUA_TNUMBER:
            case LUA_TSTRING:
                lua_tolstring(L, i, &len);
                size += len;
                break;

            case LUA_TNIL:
                size += sizeof("nil") - 1;
                break;

            case LUA_TBOOLEAN:
                if (lua_toboolean(L, i)) {
                    size += sizeof("true") - 1;

                } else {
                    size += sizeof("false") - 1;
                }

                break;

            case LUA_TTABLE:
                if (!luaL_callmeta(L, i, "__tostring")) {
                    return luaL_argerror(L, i, "expected table to have "
                                         "__tostring metamethod");
                }

                lua_tolstring(L, -1, &len);
                size += len;
                break;

            case LUA_TLIGHTUSERDATA:
                if (lua_touserdata(L, i) == NULL) {
                    size += sizeof("null") - 1;
                    break;
                }

                continue;

            default:
                msg = lua_pushfstring(L, "string, number, boolean, or nil "
                                      "expected, got %s",
                                      lua_typename(L, type));
                return luaL_argerror(L, i, msg);
        }
    }
    
    buf = lua_newuserdata(L, size);

    p = ngx_copy(buf, name.data, name.len);

    *p++ = ':';

    p = ngx_snprintf(p, NGX_INT_T_LEN, "%d",
                     ar.currentline ? ar.currentline : ar.linedefined);

    *p++ = ':'; *p++ = ' ';

    if (*ar.namewhat != '\0' && *ar.what == 'L') {
        p = ngx_copy(p, ar.name, src_len);
        *p++ = '(';
        *p++ = ')';
        *p++ = ':';
        *p++ = ' ';
    }

    for (i = 1; i <= nargs; i++) {
        type = lua_type(L, i);
        switch (type) {
            case LUA_TNUMBER:
            case LUA_TSTRING:
                q = (u_char *) lua_tolstring(L, i, &len);
                p = ngx_copy(p, q, len);
                break;

            case LUA_TNIL:
                *p++ = 'n';
                *p++ = 'i';
                *p++ = 'l';
                break;

            case LUA_TBOOLEAN:
                if (lua_toboolean(L, i)) {
                    *p++ = 't';
                    *p++ = 'r';
                    *p++ = 'u';
                    *p++ = 'e';

                } else {
                    *p++ = 'f';
                    *p++ = 'a';
                    *p++ = 'l';
                    *p++ = 's';
                    *p++ = 'e';
                }

                break;
                
            case LUA_TTABLE:
                luaL_callmeta(L, i, "__tostring");
                q = (u_char *) lua_tolstring(L, -1, &len);
                p = ngx_copy(p, q, len);
                break;

            case LUA_TLIGHTUSERDATA:
                *p++ = 'n';
                *p++ = 'u';
                *p++ = 'l';
                *p++ = 'l';

                break;

            default:
                return luaL_error(L, "impossible to reach here");
        }
    }

    if (p - buf > (off_t) size) {
        return luaL_error(L, "buffer error: %d > %d", (int) (p - buf),
                          (int) size);
    }

    ngx_log_error(level, log, 0, "%s%*s", ident, (size_t) (p - buf), buf);

    return 0;
}
```

我们来解析下这个函数，首先这里获取了一些调试相关的信息，如果对 `lua_Debug` 和 `lua_getstack` 不熟悉，可以看下相关的 Lua 文档。之后得到的 `name` 字符串则是这次调用 `ngx.log` 的 Lua 文件（`*_by_lua_file` 的指令，不包含路径）:

<img src="../images/015-ngx_lua_log_001.png">

接下来遍历剩下来传入 `ngx.log` 参数，对每个参数进行类型检查：如果是数字或者字符串，一律转换为字符串；如果是 nil，则得到字符串 `"nil"`；如果是布尔值，得到字符串 `"true"` 或者 `"false"`；如果是 table，尝试调用其元表的 `__tostring` 函数，假如调用失败，则会抛出异常（调用 `luaL_argerror`，异常同样可以在错误日志里看到）；如果是 userdata，则尝试获取其起始地址，获取不到则得到字符串 `"null"`；其他类型一律抛出异常。这个过程里计算出了序列化好所有的参数需要的内存大小，然后申请一块这么大的内存，将相关的序列化后的字符串全部写到这块内存，最后调用 `ngx_log_error` 打印日志。


### ngx_http_lua_print

这个函数则和 `print` 相关联。

```c
/**
 * Override Lua print function, output message to nginx error logs. Equal to
 * ngx.log(ngx.NOTICE, ...).
 *
 * @param L Lua state pointer
 * @retval always 0 (don't return values to Lua)
 * */
int
ngx_http_lua_print(lua_State *L)
{
    ngx_log_t                   *log;
    ngx_http_request_t          *r;

    r = ngx_http_lua_get_req(L);

    if (r && r->connection && r->connection->log) {
        log = r->connection->log;

    } else {
        log = ngx_cycle->log;
    }

    return log_wrapper(log, "[lua] ", NGX_LOG_NOTICE, L);
}
```

从代码里可以看到，`print` 相当于级别为 `ngx.NOTICE` 的 `ngx.log` 调用，后续操作一致。

### 总结

本节介绍了 ngx.log API相关的函数，虽然代码比较简单，但是这些 API 和我们平时用 `lua-nginx-module` 编码是分不开的，了解其内部实现，也有助于帮助我们更正确地使用这些 API。
