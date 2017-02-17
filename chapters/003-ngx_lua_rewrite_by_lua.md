> 以下代码均出自 lua-nginx-module v0.10.7 版本

> 这篇文章主要要介绍 `rewrite_by_lua` 这个指令 <br>
> rewrite 是 Nginx HTTP 框架划分的 11 个阶段的其中之一，通常在这个阶段我们可以实现 uri 的修改（进行内部重定向）或者 301、302 的 HTTP 重定向
> 


### rewrite_by_lua

`lua-nginx-module` 通过这个指令来介入到一个请求的 rewrite 阶段。先来看下其配置项结构

```c
     { ngx_string("rewrite_by_lua"),
     NGX_HTTP_MAIN_CONF|NGX_HTTP_SRV_CONF|NGX_HTTP_LOC_CONF|NGX_HTTP_LIF_CONF|NGX_CONF_TAKE1,
     ngx_http_lua_rewrite_by_lua,
     NGX_HTTP_LOC_CONF_OFFSET,
     0,
     (void *) ngx_http_lua_rewrite_handler_inline },
```

从这里可以了解到

- 这个指令可以出现在 main 配置块下、server 配置块下、location 配置块下和 location 下的 if 块
- 必须携带一个参数
- 解析函数是 `ngx_http_lua_rewrite_by_lua`

同样地，我们来看下 `ngx_http_lua_rewrite_by_lua ` 这个函数

```c
/* parse `rewrite_by_lua` */
char *
ngx_http_lua_rewrite_by_lua(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    u_char                      *p, *chunkname;
    ngx_str_t                   *value;
    ngx_http_lua_main_conf_t    *lmcf;
    ngx_http_lua_loc_conf_t     *llcf = conf;

    ngx_http_compile_complex_value_t         ccv;

    dd("enter");

#if defined(nginx_version) && nginx_version >= 8042 && nginx_version <= 8053
    return "does not work with " NGINX_VER;
#endif

    /*  must specify a content handler */
    if (cmd->post == NULL) {
        return NGX_CONF_ERROR;
    }

    if (llcf->rewrite_handler) {
        return "is duplicate";
    }

    value = cf->args->elts;

    if (value[1].len == 0) {
        /*  Oops...Invalid location conf */
        ngx_conf_log_error(NGX_LOG_ERR, cf, 0,
                           "invalid location config: no runnable Lua code");

        return NGX_CONF_ERROR;
    }
    
    if (cmd->post == ngx_http_lua_rewrite_handler_inline) {
        chunkname = ngx_http_lua_gen_chunk_name(cf, "rewrite_by_lua",
                                                sizeof("rewrite_by_lua") - 1);
        if (chunkname == NULL) {
            return NGX_CONF_ERROR;
        }

        llcf->rewrite_chunkname = chunkname;

        /* Don't eval nginx variables for inline lua code */

        llcf->rewrite_src.value = value[1];

        p = ngx_palloc(cf->pool, NGX_HTTP_LUA_INLINE_KEY_LEN + 1);
        if (p == NULL) {
            return NGX_CONF_ERROR;
        }

        llcf->rewrite_src_key = p;

        p = ngx_copy(p, NGX_HTTP_LUA_INLINE_TAG, NGX_HTTP_LUA_INLINE_TAG_LEN);
        p = ngx_http_lua_digest_hex(p, value[1].data, value[1].len);
        *p = '\0';
    } else {
        ngx_memzero(&ccv, sizeof(ngx_http_compile_complex_value_t));
        ccv.cf = cf;
        ccv.value = &value[1];
        ccv.complex_value = &llcf->rewrite_src;

        if (ngx_http_compile_complex_value(&ccv) != NGX_OK) {
            return NGX_CONF_ERROR;
        }

        if (llcf->rewrite_src.lengths == NULL) {
            /* no variable found */
            p = ngx_palloc(cf->pool, NGX_HTTP_LUA_FILE_KEY_LEN + 1);
            if (p == NULL) {
                return NGX_CONF_ERROR;
            }

            llcf->rewrite_src_key = p;

            p = ngx_copy(p, NGX_HTTP_LUA_FILE_TAG, NGX_HTTP_LUA_FILE_TAG_LEN);
            p = ngx_http_lua_digest_hex(p, value[1].data, value[1].len);
            *p = '\0';
        }
    }

    llcf->rewrite_handler = (ngx_http_handler_pt) cmd->post;

    lmcf = ngx_http_conf_get_module_main_conf(cf, ngx_http_lua_module);

    lmcf->requires_rewrite = 1;
    lmcf->requires_capture_filter = 1;

    return NGX_CONF_OK;
}
```

首先这个函数计算了个叫做 `chunkname` 的字符串，可以看到它是在函数 `ngx_http_lua_gen_chunk_name` 里得到的，通过 gdb 调试，我们可以看到，最终得到的 `chunkname` 是 `"rewrite_by_lua(marco.conf:148)"`，这个字符串的作用后面会介绍。

<img src="../images/003-ngx_http_lua_rewrite_by_lua-001.png" alt="chunkname">

接下来根据配置项配置的 post 指针指向的函数不同，记录下 Lua code 或者 Lua code path；最后为 Lua code cache 这个功能计算了一个 cache key，通过 gdb 我们来看下计算出来的 key 是什么样的（这边实例走的是 `rewrite_by_lua_file` 指令）

<img src="../images/003-ngx_http_lua_rewrite_by_lua-002.png" alt="rewrite_src_key">

后面就是用这个 key 来检索缓存 Lua code，当然前提是 `lua_code_cache` 是 `on`

恩，指令解析到这里就结束了，那么，`rewrite_by_lua` 究竟是如何让 Lua code 介入到 HTTP 请求里的呢？下面就来一探究竟

### rewrite_by_lua_handler

这是真正被插入到 rewrite 阶段的回调方法（见 [ngx_lua_init_by_lua](001-ngx_lua_init_by_lua.md) 关于`ngx_http_lua_init` 的介绍）

```c
ngx_int_t
ngx_http_lua_rewrite_handler_inline(ngx_http_request_t *r)
{
    lua_State                   *L;
    ngx_int_t                    rc;
    ngx_http_lua_loc_conf_t     *llcf;

    dd("rewrite by lua inline");

    llcf = ngx_http_get_module_loc_conf(r, ngx_http_lua_module);
    L = ngx_http_lua_get_lua_vm(r, NULL);

    /*  load Lua inline script (w/ cache) sp = 1 */
    rc = ngx_http_lua_cache_loadbuffer(r->connection->log, L,
                                       llcf->rewrite_src.value.data,
                                       llcf->rewrite_src.value.len,
                                       llcf->rewrite_src_key,
                                       (const char *)
                                       llcf->rewrite_chunkname);
    if (rc != NGX_OK) {
        return NGX_HTTP_INTERNAL_SERVER_ERROR;
    }

    return ngx_http_lua_rewrite_by_chunk(L, r);
}
```

这个函数通过函数 `ngx_http_lua_get_lua_vm` 获取到 Lua VM，然后进行 rewrite 阶段的 Lua code 的“载入”（从缓存里取或者调用 `lua_loadbuffer` 载入）
最后调用 `ngx_http_lua_rewrite_by_chunk ` 运行 Lua chunk。下面就来分析下这些过程

### ngx_http_lua_get_lua_vm

首先，Lua VM 是怎么取到的呢？请看 `ngx_http_lua_get_lua_vm`

```c
static ngx_inline lua_State *
ngx_http_lua_get_lua_vm(ngx_http_request_t *r, ngx_http_lua_ctx_t *ctx)
{
    ngx_http_lua_main_conf_t    *lmcf;

    if (ctx == NULL) {
        ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    }

    if (ctx && ctx->vm_state) {
        return ctx->vm_state->vm;
    }

    lmcf = ngx_http_get_module_main_conf(r, ngx_http_lua_module);
    dd("lmcf->lua: %p", lmcf->lua);
    return lmcf->lua;
}
```

代码非常简单，首先是尝试从 `lua-nginx-module` 模块上下文里取 Lua VM，但是有可能此时 ctx 还没有被创建出来，所以在 ctx 没有被创建出来的情况下，就从 `lua-nginx-module` 的 main 配置结构体里取出 Lua VM

### ngx_http_lua_cache_loadbuffer

接下来我们看下 得到 Lua chunk 的过程，也就是 `ngx_http_lua_cache_loadbuffer`
这个函数

```c
ngx_int_t
ngx_http_lua_cache_loadbuffer(ngx_log_t *log, lua_State *L,
    const u_char *src, size_t src_len, const u_char *cache_key,
    const char *name)
{
    int          n;
    ngx_int_t    rc;
    const char  *err = NULL;

    n = lua_gettop(L);

    dd("XXX cache key: [%s]", cache_key);

    rc = ngx_http_lua_cache_load_code(log, L, (char *) cache_key);
    if (rc == NGX_OK) {
        /*  code chunk loaded from cache, sp++ */
        dd("Code cache hit! cache key='%s', stack top=%d, script='%.*s'",
           cache_key, lua_gettop(L), (int) src_len, src);
        return NGX_OK;
    }

    if (rc == NGX_ERROR) {
        return NGX_ERROR;
    }

    /* rc == NGX_DECLINED */

    dd("Code cache missed! cache key='%s', stack top=%d, script='%.*s'",
       cache_key, lua_gettop(L), (int) src_len, src);

    /* load closure factory of inline script to the top of lua stack, sp++ */
    rc = ngx_http_lua_clfactory_loadbuffer(L, (char *) src, src_len, name);

    if (rc != 0) {
        /*  Oops! error occurred when loading Lua script */
        if (rc == LUA_ERRMEM) {
            err = "memory allocation error";

        } else {
            if (lua_isstring(L, -1)) {
                err = lua_tostring(L, -1);
                
            } else {
                err = "unknown error";
            }
        }

        goto error;
    }

    /*  store closure factory and gen new closure at the top of lua stack to
     *  code cache */
    rc = ngx_http_lua_cache_store_code(L, (char *) cache_key);
    if (rc != NGX_OK) {
        err = "fail to generate new closure from the closure factory";
        goto error;
    }

    return NGX_OK;

error:

    ngx_log_error(NGX_LOG_ERR, log, 0,
                  "failed to load inlined Lua code: %s", err);
    lua_settop(L, n);
    return NGX_ERROR;
}
```

还记得之前所说的 `chunkname` 和 `rewrite_src_key`？没错，那两个字符串的用武之地就是在这里！我们来分析下这个函数，这里有个缓存状态机，步骤如下

1. 调用 `ngx_http_lua_cache_load_code`，判断当前的 Lua chunk 有没有缓存，得到返回码，如果返回码为 NGX_OK，跳到第二步；如果返回码是 `NGX_ERROR`，跳到第三步；否则跳到第四步
2. 从缓存中拿到 Lua chunk 且被压入到栈，返回 NGX_OK
3. 出错，返回 NGX_ERROR
4. 缓存 Miss，从原生的 Lua 代码加载，然后压栈，如果出错，记录错误日志然后返回 `NGX_ERROR`；否则返回 `NGX_OK`

这里对 `ngx_http_lua_cache_load_code `，`ngx_http_lua_clfactory_loadbuffer ` 和 `ngx_http_lua_cache_store_code ` 这三个具体的函数不展开分析，有兴趣可以自行阅读


### ngx_http_lua_rewrite_by_chunk

恩，现在 Lua chunk 拿到了而且已经在栈顶了，下面的任务就是把它跑起来了
