> 以下代码均出自 lua-nginx-module v0.10.7 版本

> 本文主要介绍 `init_by_lua ` 这个指令，通过这个指令的透析，我们还可以了解 Lua code 到底是怎么运行起来的，本文着眼于整体步骤因此没有入到很深的代码细节


### init_by_lua 指令

`init_by_lua` 是 `lua-nginx-module` 提供的一条配置指令，先来看下其配置项

```c
    { ngx_string("init_by_lua"),
      NGX_HTTP_MAIN_CONF|NGX_CONF_TAKE1,
      ngx_http_lua_init_by_lua,
      NGX_HTTP_MAIN_CONF_OFFSET,
      0,
      (void *) ngx_http_lua_init_by_inline }
```

从这个配置项我们可以掌握到的信息有：

- 该指令工作在 http 的 main 配置块下且必须携带一个参数
- 该指令的解析函数是 `ngx_http_lua_init_by_lua`；
- post 指针指向了 `ngx_http_lua_init_by_inline` 函数

接着阅读下 `ngx_http_lua_init_by_lua `

```c
char *
ngx_http_lua_init_by_lua(ngx_conf_t *cf, ngx_command_t *cmd,
    void *conf)
{
    u_char                      *name;
    ngx_str_t                   *value;
    ngx_http_lua_main_conf_t    *lmcf = conf;

    dd("enter");

    /*  must specify a content handler */
    if (cmd->post == NULL) {
        return NGX_CONF_ERROR;
    }

    if (lmcf->init_handler) {
        return "is duplicate";
    }

    value = cf->args->elts;

    if (value[1].len == 0) {
        /*  Oops...Invalid location conf */
        ngx_conf_log_error(NGX_LOG_ERR, cf, 0,
                           "invalid location config: no runnable Lua code");
        return NGX_CONF_ERROR;
    }

    lmcf->init_handler = (ngx_http_lua_main_conf_handler_pt) cmd->post;

    if (cmd->post == ngx_http_lua_init_by_file) {
        /* get the full name of init lua file */
        name = ngx_http_lua_rebase_path(cf->pool, value[1].data,
                                        value[1].len);
        if (name == NULL) {
            return NGX_CONF_ERROR;
        }

        lmcf->init_src.data = name;
        lmcf->init_src.len = ngx_strlen(name);
        
    } else {
        /* inline code */
        lmcf->init_src = value[1];
    }

    return NGX_CONF_OK;
}
```

这个函数让 `init_hander` 指向了配置项的 post 指针指向的函数；然后把对应 Lua 代码串赋值给 `init_src` 这个字符串（如果 post 指针指向`ngx_http_lua_init_by_file` 那么则是把对应 Lua 代码的绝对路径赋值给 `init_src`）

实际上这条指令对应的 Lua 代码将会在配置项解析完之后被执行（见 ngx_http_lua_module.c:594，lua-nginx-module 模块上下文中 `postconfiguartion`），被执行的函数是 `ngx_http_lua_init`

```c
/* in postconfiguration */
static ngx_int_t
ngx_http_lua_init(ngx_conf_t *cf)
{
    int                         multi_http_blocks;
    ngx_int_t                   rc;
    ngx_array_t                *arr;
    ngx_http_handler_pt        *h;
    volatile ngx_cycle_t       *saved_cycle;
    ngx_http_core_main_conf_t  *cmcf;
    ngx_http_lua_main_conf_t   *lmcf;
#ifndef NGX_LUA_NO_FFI_API
    ngx_pool_cleanup_t         *cln;
#endif

    lmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_lua_module);

    if (ngx_http_lua_prev_cycle != ngx_cycle) {
        ngx_http_lua_prev_cycle = ngx_cycle;
        multi_http_blocks = 0;

    } else {
        multi_http_blocks = 1;
    }

    if (multi_http_blocks || lmcf->requires_capture_filter) {
        rc = ngx_http_lua_capture_filter_init(cf);
                if (rc != NGX_OK) {
            return rc;
        }
    }

    if (lmcf->postponed_to_rewrite_phase_end == NGX_CONF_UNSET) {
        lmcf->postponed_to_rewrite_phase_end = 0;
    }

    if (lmcf->postponed_to_access_phase_end == NGX_CONF_UNSET) {
        lmcf->postponed_to_access_phase_end = 0;
    }

    cmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_core_module);

    if (lmcf->requires_rewrite) {
        /* insert the rewrite phase handler */
        h = ngx_array_push(&cmcf->phases[NGX_HTTP_REWRITE_PHASE].handlers);
        if (h == NULL) {
            return NGX_ERROR;
        }

        *h = ngx_http_lua_rewrite_handler;
    }
    
    if (lmcf->requires_access) {
        /* insert the access phase handler */
        h = ngx_array_push(&cmcf->phases[NGX_HTTP_ACCESS_PHASE].handlers);
        if (h == NULL) {
            return NGX_ERROR;
        }

        *h = ngx_http_lua_access_handler;
    }

    dd("requires log: %d", (int) lmcf->requires_log);

    if (lmcf->requires_log) {
        /* insert the log phase handler */
        arr = &cmcf->phases[NGX_HTTP_LOG_PHASE].handlers;
        h = ngx_array_push(arr);
        if (h == NULL) {
            return NGX_ERROR;
        }

        if (arr->nelts > 1) {
            h = arr->elts;
            ngx_memmove(&h[1], h,
                        (arr->nelts - 1) * sizeof(ngx_http_handler_pt));
        }

        *h = ngx_http_lua_log_handler;
    }

    if (multi_http_blocks || lmcf->requires_header_filter) {
        rc = ngx_http_lua_header_filter_init();
        if (rc != NGX_OK) {
            return rc;
        }
    }
    if (multi_http_blocks || lmcf->requires_body_filter) {
        rc = ngx_http_lua_body_filter_init();
        if (rc != NGX_OK) {
            return rc;
        }
    }

#ifndef NGX_LUA_NO_FFI_API
    /* add the cleanup of semaphores after the lua_close */
    cln = ngx_pool_cleanup_add(cf->pool, 0);
    if (cln == NULL) {
        return NGX_ERROR;
    }

    cln->data = lmcf;
    cln->handler = ngx_http_lua_sema_mm_cleanup;
#endif

    if (lmcf->lua == NULL) {
        dd("initializing lua vm");

        ngx_http_lua_content_length_hash =
                                  ngx_http_lua_hash_literal("content-length");
        ngx_http_lua_location_hash = ngx_http_lua_hash_literal("location");

        lmcf->lua = ngx_http_lua_init_vm(NULL, cf->cycle, cf->pool, lmcf,
                                         cf->log, NULL);
        if (lmcf->lua == NULL) {
            ngx_conf_log_error(NGX_LOG_ERR, cf, 0,
                               "failed to initialize Lua VM");
            return NGX_ERROR;
        }

        if (!lmcf->requires_shm && lmcf->init_handler) {
            /* run the init phase code */
            saved_cycle = ngx_cycle;
            ngx_cycle = cf->cycle;

            rc = lmcf->init_handler(cf->log, lmcf, lmcf->lua);

            ngx_cycle = saved_cycle;

            if (rc != NGX_OK) {
                /* an error happened */
                return NGX_ERROR;
            }
        }

        dd("Lua VM initialized!");
    }

    return NGX_OK;
}
```

嗯，这个函数比较长，让我们看看它到底做了什么事：

- 判断 `lua-nginx-module` 是否介入 rewrite、access 和 log 阶段，如果介入，则放到相应阶段的 handlers 动态数组内
- 判断 `lua-nginx-module` 是否介入到 header_filter 和 body_filter 阶段，如果介入，则需要把 head_filter 或者 body_filter 的“链”设置下
- 判断是否需要初始化 Lua VM，如果需要，则调用 `ngx_http_lua_init_vm` 得到一个 lua_State，并调用因 `init_by_lua` 这个指令而设置的 init_handler

那么 `init_by_lua` 的使命就结束了，但是，究竟这里的 Lua 代码是怎么被执行的呢？

### Lua Code 加载和执行

我们还没有对 `ngx_http_lua_init_by_inline` 一探究竟，接下来就来看看这个函数吧

```c
ngx_int_t
ngx_http_lua_init_by_inline(ngx_log_t *log, ngx_http_lua_main_conf_t *lmcf,
    lua_State *L)
{
    int         status;

    status = luaL_loadbuffer(L, (char *) lmcf->init_src.data,
                             lmcf->init_src.len, "=init_by_lua")
             || ngx_http_lua_do_call(log, L);

    return ngx_http_lua_report(log, L, status, "init_by_lua");
}
```

从这里看到，它只是调用了 `luaL_loadbuffer` 这个 Lua 的 API，把代码加载成 Lua chunk，然后调用 `ngx_http_lua_do_call` 来真正执行代码

```c
int
ngx_http_lua_do_call(ngx_log_t *log, lua_State *L)
{
    int                 status, base;
#if (NGX_PCRE)
    ngx_pool_t         *old_pool;
#endif

    base = lua_gettop(L);  /* function index */
    lua_pushcfunction(L, ngx_http_lua_traceback);  /* push traceback function */
    lua_insert(L, base);  /* put it under chunk and args */

#if (NGX_PCRE)
    old_pool = ngx_http_lua_pcre_malloc_init(ngx_cycle->pool);
#endif

    status = lua_pcall(L, 0, 0, base);

#if (NGX_PCRE)
    ngx_http_lua_pcre_malloc_done(old_pool);
#endif

    lua_remove(L, base);

    return status;
}
```

这个函数把 `ngx_http_lua_traceback` 压入到 Lua 的虚拟栈，作为异常处理函数，然后调用了 `lua_pcall` 来执行刚才加载的 chunk，得到运行之后的状态，如果出错则会得到一个描述错误的字符串（在栈底），`status` 为 0 表示运行成功，否则返回对应的错误码

最终这个状态码被传给 `ngx_http_lua_report` 这个函数

```c
ngx_int_t
ngx_http_lua_report(ngx_log_t *log, lua_State *L, int status,
    const char *prefix)
{
    const char      *msg;

    if (status && !lua_isnil(L, -1)) {
        msg = lua_tostring(L, -1);
        if (msg == NULL) {
            msg = "unknown error";
        }

        ngx_log_error(NGX_LOG_ERR, log, 0, "%s error: %s", prefix, msg);
        lua_pop(L, 1);
    }

    /* force a full garbage-collection cycle */
    lua_gc(L, LUA_GCCOLLECT, 0);

    return status == 0 ? NGX_OK : NGX_ERROR;
}
```

这个函数只是在 `status` 非 0 的时候进行了一次日志记录，然后强制进行一次垃圾回收。而通常我们看到的这样的错误日志则如下：

```
stack traceback:
 coroutine 0:
...t/WorkStation/marco/nginx//app/lib/resty/globalcache.lua: in function 'get_data'
 ...nt/WorkStation/marco/nginx//app/src/modules/requests.lua:998: in function 'set_dyn_parents_header'
...vagrant/WorkStation/marco/nginx/app/src/marco_access.lua:111: in function <...vagrant/WorkStation/marco/nginx/app/src/marco_access.lua:1>, client: 127.0.0.1, server: *       .b0.upaiyun.com, request: "GET /dyn-parents HTTP/1.1", host: "hbimg.b0.upaiyun.com"
```

那么在这个阶段假如出现 Lua 代码出错，Nginx 是无法正常启动起来的，比如我随便在 `init_by_lua` 的 Lua 代码里写点错误的东西，Nginx 启动就把对应的日志打到了屏幕上

```bash
→ sudo make start                                                                                                                                                          
./nginx/sbin/nginx
nginx: [error] init_by_lua_file error: ...e/vagrant/WorkStation/marco/nginx/app/src/marco_init.lua:14: attempt to index global 'cccc' (a nil value)
stack traceback:
        ...e/vagrant/WorkStation/marco/nginx/app/src/marco_init.lua:14: in main chunk
make: *** [start] Error 1
```

最后来看下 `ngx_http_lua_traceback` 这个异常处理函数

```c
int
ngx_http_lua_traceback(lua_State *L)
{
    if (!lua_isstring(L, 1)) { /* 'message' not a string? */
        return 1;  /* keep it intact */
    }

    lua_getglobal(L, "debug");
    if (!lua_istable(L, -1)) {
        lua_pop(L, 1);
        return 1;
    }

    lua_getfield(L, -1, "traceback");
    if (!lua_isfunction(L, -1)) {
        lua_pop(L, 2);
        return 1;
    }

    lua_pushvalue(L, 1);  /* pass error message */
    lua_pushinteger(L, 2);  /* skip this function and traceback */
    lua_call(L, 2, 1);  /* call debug.traceback */
    return 1;
}
```

这个函数调用 Lua 的 `debug.traceback`，然后把之前得到的错误信息压入栈中，将出错登记设置为 2（也就是调用 `ngx_http_lua_traceback` 的函数，1 是调用 `debug.traceback` 的函数）


### 总结

本文从 `init_by_lua` 这个指令由小见大，初步了解了 Lua 代码的执行逻辑，以及其相应的错误处理方式
