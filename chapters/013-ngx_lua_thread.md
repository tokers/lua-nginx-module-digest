> 以下代码均出自 lua-nginx-module v0.10.7 版本

> 这篇文章主要深入介绍 lua-nginx-module 里关于创建、运行和销毁一个协程的相关逻辑
> Lua 不支持真正的多线程，我们要介绍的是协程（类似于线程），任何时候只会有一个协程在运行
> 

### Lua 协程的 5 个状态

Lua 协程有 5 个状态，分别是 suspend（挂起）、running（运行）、dead（死亡）、 normal（正常）和 僵尸（zombie）

```c
typedef enum {
    NGX_HTTP_LUA_CO_RUNNING   = 0, /* coroutine running */
    NGX_HTTP_LUA_CO_SUSPENDED = 1, /* coroutine suspended */
    NGX_HTTP_LUA_CO_NORMAL    = 2, /* coroutine normal */
    NGX_HTTP_LUA_CO_DEAD      = 3, /* coroutine dead */
    NGX_HTTP_LUA_CO_ZOMBIE    = 4, /* coroutine zombie */
} ngx_http_lua_co_status_t;
``` 

### ngx_http_lua_new_thread

顾名思义，这个函数的功能就是创建一个新的协程

```c
lua_State *
ngx_http_lua_new_thread(ngx_http_request_t *r, lua_State *L, int *ref)
{
    int              base;
    lua_State       *co;

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua creating new thread");

    base = lua_gettop(L);

    lua_pushlightuserdata(L, &ngx_http_lua_coroutines_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    co = lua_newthread(L);

    /*  inherit coroutine's globals to main thread's globals table
     *  for print() function will try to find tostring() in current
     *  globals table.
     */
    /*  new globals table for coroutine */
    ngx_http_lua_create_new_globals_table(co, 0, 0);

    lua_createtable(co, 0, 1);
    ngx_http_lua_get_globals_table(co);
    lua_setfield(co, -2, "__index");
    lua_setmetatable(co, -2);

    ngx_http_lua_set_globals_table(co);

    *ref = luaL_ref(L, -2);

    if (*ref == LUA_NOREF) {
        lua_settop(L, base);  /* restore main thread stack */
        return NULL;
    }

    lua_settop(L, base);
    return co;
}
```

我们来剖析下这个函数。
首先，它从 lua 注册表里取出一个子表，这个子表负责存放协程的 `lua_State` 对象指针。
然后调用 `lua_newthread` 创建一个新的协程 co，注意，新创建的协程的 `lua_State` 对象会被压到父亲协程的栈顶。<br>
而接下来的几行代码，都是在为这个新协程设置全局表：

- 创建出一个新的表 t，设置 t["_G"] = t（这样我们的 Lua 代码里就可以通过 `_G` 得到整张全局表了）
- 创建一张表 mt，其 `__index` 域设置为当前的全局表，然后把表 mt 设置为表 t 的元表，并用表 t 作为新的全局表(`ngx_http_lua_set_globals_table`)，这么做你会发现，所有的子协程只能共享父协程的全局变量而不能相互共享其他子协程的全局变量，我们来简单验证下：

```nginx
server {
    listen 9999;

    location / {
        rewrite_by_lua_block {
            ngx.log(ngx.WARN, "test coroutine global var ", var)
            var = 12345
        }
    }
}
```

设置一个虚拟主机如上，另外为了排除其他的因素干扰，我们只开一个 worker 进程，请求两次，日志如下：

```bash
2017/02/23 20:18:43 [warn] 13965#13965: *3 [lua] rewrite_by_lua(test.conf:9):2: test coroutine global var nil, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", host: "127.0.0.1:9999"
2017/02/23 20:18:43 [warn] 13965#13965: *3 [lua] rewrite_by_lua(test.conf:9):2: test coroutine global var nil, client: 127.0.0.1, server: , request: "GET / HTTP/1.1", host: "127.0.0.1:9999"
2017/02/23 20:18
```

因此 `entry thread` 的全局变量都不是真正的全局变量，防止了全局变量的滥用

最后我们看到这个函数创建了一个对新协程的引用（防止垃圾回收），然后恢复父亲协程的栈，最后返回这个新协程的 `lua_State` 对象指针（或者创建引用失败就返回 `NULL`）。


### ngx_http_lua_run_thread

> 该函数比较长，这里不贴代码了。

这个函数负责运行一个 Lua 协程。<br>
首先讲下这个函数的准备工作，设置一个当 Lua VM 抛出异常时进行处理的回调函数，也就是

```c
lua_atpanic(L, ngx_http_lua_atpanic)

/*  longjmp mark for restoring nginx execution after Lua VM crashing */
jmp_buf ngx_http_lua_exception;

/**
 * Override default Lua panic handler, output VM crash reason to nginx error
 * log, and restore execution to the nearest jmp-mark.
 *
 * @param L Lua state pointer
 * @retval Long jump to the nearest jmp-mark, never returns.
 * @note nginx request pointer should be stored in Lua thread's globals table
 * in order to make logging working.
 * */
int
ngx_http_lua_atpanic(lua_State *L)
{
#ifdef NGX_LUA_ABORT_AT_PANIC
    abort();
#else
    u_char                  *s = NULL;
    size_t                   len = 0;

    if (lua_type(L, -1) == LUA_TSTRING) {
        s = (u_char *) lua_tolstring(L, -1, &len);
    }

    if (s == NULL) {
        s = (u_char *) "unknown reason";
        len = sizeof("unknown reason") - 1;
    }

    ngx_log_stderr(0, "lua atpanic: Lua VM crashed, reason: %*s", len, s);
    ngx_quit = 1;

    /*  restore nginx execution */
    NGX_LUA_EXCEPTION_THROW(1);

    /* impossible to reach here */
#endif
}
```

`ngx_http_lua_atpanic ` 这个函数会把异常信息写入到错误日志中，级别是 `error`，然后恢复抛异常的协程运行前的运行环境，这是通过 `setjmp` 和 `longjmp` 来实现的。

紧接着就是利用 `lua_resume` 来运行目标协程了，需要运行的函数及其参数在调用 `ngx_http_lua_run_thread ` 前就已经压到目标协程虚拟机的栈之上了。接着需要 `lua_resume` 不同的返回码做不同的处理。

1. 如果返回码是 `LUA_YIELD`，说明目标协程没有运行完，而是把自己挂起了。

