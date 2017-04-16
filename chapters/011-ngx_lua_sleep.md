> 以下代码均出自 lua-nginx-module v0.10.7 版本
> 
> ngx.sleep 是 lua-nginx-module 提供的一个延迟处理请求的机制，而且该机制是 
> 100% 非阻塞的，充分利用了 Nginx 提供的定时器，使得用户的业务处理更加灵活。下面就来剖析下这个 API（这里介绍的是基于经典的 LuaC_function 的实现，而非是 lua-resty-core 提供的基于 LuaJIT FFI 的实现）
> 
> 

### ngx_http_lua_ngx_sleep

`ngx.sleep` 底层就是这个 `ngx_http_lua_ngx_sleep` 函数。这个函数主要完成一些参数检查、请求阶段检查等工作。先来看下它的源码：

```c
static int
ngx_http_lua_ngx_sleep(lua_State *L)
{
    int                          n;
    ngx_int_t                    delay; /* in msec */
    ngx_http_request_t          *r;
    ngx_http_lua_ctx_t          *ctx;
    ngx_http_lua_co_ctx_t       *coctx;

    n = lua_gettop(L);
    if (n != 1) {
        return luaL_error(L, "attempt to pass %d arguments, but accepted 1", n);
    }

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request found");
    }

    delay = (ngx_int_t) (luaL_checknumber(L, 1) * 1000);

    if (delay < 0) {
        return luaL_error(L, "invalid sleep duration \"%d\"", delay);
    }

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return luaL_error(L, "no request ctx found");
    }

    ngx_http_lua_check_context(L, ctx, NGX_HTTP_LUA_CONTEXT_REWRITE
                               | NGX_HTTP_LUA_CONTEXT_ACCESS
                               | NGX_HTTP_LUA_CONTEXT_CONTENT
                               | NGX_HTTP_LUA_CONTEXT_TIMER
                               | NGX_HTTP_LUA_CONTEXT_SSL_CERT
                               | NGX_HTTP_LUA_CONTEXT_SSL_SESS_FETCH);

    coctx = ctx->cur_co_ctx;
    if (coctx == NULL) {
        return luaL_error(L, "no co ctx found");
    }

    ngx_http_lua_cleanup_pending_operation(coctx);
    coctx->cleanup = ngx_http_lua_sleep_cleanup;
    coctx->data = r;

    coctx->sleep.handler = ngx_http_lua_sleep_handler;
    coctx->sleep.data = coctx;
    coctx->sleep.log = r->connection->log;

    dd("adding timer with delay %lu ms, r:%.*s", (unsigned long) delay,
       (int) r->uri.len, r->uri.data);

    ngx_add_timer(&coctx->sleep, (ngx_msec_t) delay);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua ready to sleep for %d ms", delay);

    return lua_yield(L, 0);
}
```

这个函数的核心功能就是将协程上下文结构体里的 sleep 这个事件进行设置，包括当 sleep 事件过期时需要回调的 handler，以及所需要的参数的设置（我们知道 ngx_event_t 结构体的 data 通常用来指向这个事件所属的连接，而这里则指向了 coctx）。之后将事件放入定时器，最后将当前协程挂起。 <br>
我们知道，调用 `lua_yield` 挂起当前协程，会唤醒调用 `lua_resume` 的那个协程。而
`lua_resume` 则是在 `ngx_http_lua_run_thread` 里面被调用的（即回到 `entry thread`），之后 Nginx 即可调度其他的请求。
等一轮网络调度（epoll 等）完成后，就轮到处理定时器事件了。而假如我们通过 `ngx.sleep` 产生的定时器事件到期，那么 `ngx_http_lua_sleep_handler ` 这个回调就会被执行了。

### ngx_http_lua_sleep_handler

这个 handler 的目的就是恢复之前被挂起的协程。

```c
void
ngx_http_lua_sleep_handler(ngx_event_t *ev)
{
    ngx_connection_t        *c;
    ngx_http_request_t      *r;
    ngx_http_lua_ctx_t      *ctx;
    ngx_http_log_ctx_t      *log_ctx;
    ngx_http_lua_co_ctx_t   *coctx;

    coctx = ev->data;

    r = coctx->data;
    c = r->connection;

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);

    if (ctx == NULL) {
        return;
    }

    if (c->fd != (ngx_socket_t) -1) {  /* not a fake connection */
        log_ctx = c->log->data;
        log_ctx->current_request = r;
    }

    coctx->cleanup = NULL;

    ngx_log_debug2(NGX_LOG_DEBUG_HTTP, c->log, 0,
                   "lua sleep timer expired: \"%V?%V\"", &r->uri, &r->args);

    ctx->cur_co_ctx = coctx;

    if (ctx->entered_content_phase) {
        (void) ngx_http_lua_sleep_resume(r);

    } else {
        ctx->resume_handler = ngx_http_lua_sleep_resume;
        ngx_http_core_run_phases(r);
    }

    ngx_http_run_posted_requests(c);
}
```

/* TODO 解释下为什么这里对 ngx_http_lua_sleep_resume 在请求经历了 content_by_lua* 的情况下作了区分 */

### ngx_http_lua_sleep_resume

这个函数的任务就是恢复运行之前因为 `ngx.sleep` 而被挂起的协程。

```c
static ngx_int_t
ngx_http_lua_sleep_resume(ngx_http_request_t *r)
{
    lua_State                   *vm;
    ngx_connection_t            *c;
    ngx_int_t                    rc;
    ngx_http_lua_ctx_t          *ctx;

    ctx = ngx_http_get_module_ctx(r, ngx_http_lua_module);
    if (ctx == NULL) {
        return NGX_ERROR;
    }

    ctx->resume_handler = ngx_http_lua_wev_handler;

    c = r->connection;
    vm = ngx_http_lua_get_lua_vm(r, ctx);

    rc = ngx_http_lua_run_thread(vm, r, ctx, 0);

    ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                   "lua run thread returned %d", rc);

    if (rc == NGX_AGAIN) {
        return ngx_http_lua_run_posted_threads(c, vm, r, ctx);
    }

    if (rc == NGX_DONE) {
        ngx_http_lua_finalize_request(r, NGX_DONE);
        return ngx_http_lua_run_posted_threads(c, vm, r, ctx);
    }

    if (ctx->entered_content_phase) {
        ngx_http_lua_finalize_request(r, rc);
        return NGX_DONE;
    }

    return rc;
}
```

如源码所示的，通过 ngx_http_lua_run_thread 运行。