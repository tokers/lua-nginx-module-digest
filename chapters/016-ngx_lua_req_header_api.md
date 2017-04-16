> 以下代码均出自 lua-nginx-module v0.10.7 版本

> 这篇文章主要深入介绍 `lua-nginx-module` 里和请求头相关的一些 API，充分了解在 C 层面这些函数究竟是如何实现的
> 

### ngx_http_lua_inject_req_header_api

这个函数完成了请求头相关的 API 的注入，请求头相关的 API 放置在 `ngx.req` 这个表中。相关 API 有 `ngx.req.http_version()`、`ngx.req.raw_header`、`ngx.req.clear_header`、`ngx.req.set_header` 和 `ngx.req.get_headers()`

### ngx.req.http_version
'
这个 API，返回请求所用的 HTTP 协议版本， 底层实现函数是 `ngx_http_lua_ngx_req_http_version`

```c
static int
ngx_http_lua_ngx_req_http_version(lua_State *L)
{
    ngx_http_request_t          *r;

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ngx_http_lua_check_fake_request(L, r);

    switch (r->http_version) {
    case NGX_HTTP_VERSION_9:
        lua_pushnumber(L, 0.9);
        break;

    case NGX_HTTP_VERSION_10:
        lua_pushnumber(L, 1.0);
        break;

    case NGX_HTTP_VERSION_11:
        lua_pushnumber(L, 1.1);
        break;

#ifdef NGX_HTTP_VERSION_20
    case NGX_HTTP_VERSION_20:
        lua_pushnumber(L, 2.0);
        break;
#endif

    default:
        lua_pushnil(L);
        break;
    }

    return 1;
}
```

其实就是利用 `r->http_version` 来实现，非常简单。

### ngx.req.raw_header

这个函数返回没有经过序列化的、从 TCP 缓冲区里接收到的原始的 HTTP 请求行+请求头，比如 `lua-nginx-module` github 上给的示例（包含请求行）：

```http
GET /t HTTP/1.1
Host: localhost
Connection: close
Foo: bar
```

其底层实现是 `ngx_http_lua_ngx_req_raw_header`。

```c
static int
ngx_http_lua_ngx_req_raw_header(lua_State *L)
{
    int                          n, line_break_len;
    u_char                      *data, *p, *last, *pos;
    unsigned                     no_req_line = 0, found;
    size_t                       size;
    ngx_buf_t                   *b, *first = NULL;
    ngx_int_t                    i, j;
    ngx_connection_t            *c;
    ngx_http_request_t          *r, *mr;
    ngx_http_connection_t       *hc;

    n = lua_gettop(L);
    if (n > 0) {
        no_req_line = lua_toboolean(L, 1);
    }

    dd("no req line: %d", (int) no_req_line);

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    /* r->connectino->fd != (ngx_socket_t) -1*/
    ngx_http_lua_check_fake_request(L, r);

    mr = r->main;
    hc = mr->http_connection;
    c = mr->connection;

#if (NGX_HTTP_V2)
    if (mr->stream) {
        return luaL_error(L, "http v2 not supported yet");
    }
#endif

#if 1
    dd("hc->nbusy: %d", (int) hc->nbusy);

    if (hc->nbusy) {
        dd("hc->busy: %p %p %p %p", hc->busy[0]->start, hc->busy[0]->pos,
           hc->busy[0]->last, hc->busy[0]->end);
    }

    dd("request line: %p %p", mr->request_line.data,
       mr->request_line.data + mr->request_line.len);

    dd("header in: %p %p %p %p", mr->header_in->start,
       mr->header_in->pos, mr->header_in->last,
       mr->header_in->end);

    dd("c->buffer: %p %p %p %p", c->buffer->start,
       c->buffer->pos, c->buffer->last,
       c->buffer->end);
#endif

    size = 0;
    b = c->buffer;

    if (mr->request_line.data[mr->request_line.len] == CR) {
        line_break_len = 2;

    } else {
        line_break_len = 1;
    }

    if (mr->request_line.data >= b->start
        && mr->request_line.data + mr->request_line.len
           + line_break_len <= b->pos)
    {
        first = b;
        size += b->pos - mr->request_line.data;
    }

    dd("size: %d", (int) size);

    if (hc->nbusy) {
        b = NULL;
        for (i = 0; i < hc->nbusy; i++) {
            b = hc->busy[i];

            dd("busy buf: %d: [%.*s]", (int) i, (int) (b->pos - b->start),
               b->start);

            if (first == NULL) {
                if (mr->request_line.data >= b->pos
                    || mr->request_line.data
                       + mr->request_line.len + line_break_len
                       <= b->start)
                {
                    continue;
                }

                dd("found first at %d", (int) i);
                first = b;
            }

            dd("adding size %d", (int) (b->pos - b->start));
            size += b->pos - b->start;
        }
    }
    
    size++;  /* plus the null terminator, as required by the later
                ngx_strstr() call */

    dd("header size: %d", (int) size);

    data = lua_newuserdata(L, size);
    last = data;

    b = c->buffer;
    found = 0;

    if (first == b) {
        found = 1;
        pos = b->pos;

        if (no_req_line) {
            last = ngx_copy(data,
                            mr->request_line.data
                            + mr->request_line.len + line_break_len,
                            pos - mr->request_line.data
                            - mr->request_line.len - line_break_len);

        } else {
            last = ngx_copy(data, mr->request_line.data,
                            pos - mr->request_line.data);
        }

        if (b != mr->header_in) {
            /* skip truncated header entries (if any) */
            while (last > data && last[-1] != LF) {
                last--;
            }
        }

        i = 0;
        for (p = data; p != last; p++) {
            if (*p == '\0') {
                i++;
                if (p + 1 != last && *(p + 1) == LF) {
                    *p = CR;

                } else if (i % 2 == 1) {
                    *p = ':';

                } else {
                    *p = LF;
                }
            }
        }
    }
    
    if (hc->nbusy) {
        for (i = 0; i < hc->nbusy; i++) {
            b = hc->busy[i];

            if (!found) {
                if (b != first) {
                    continue;
                }

                dd("found first");
                found = 1;
            }

            p = last;

            pos = b->pos;

            if (b == first) {
                dd("request line: %.*s", (int) mr->request_line.len,
                   mr->request_line.data);

                if (no_req_line) {
                    last = ngx_copy(last,
                                    mr->request_line.data
                                    + mr->request_line.len + line_break_len,
                                    pos - mr->request_line.data
                                    - mr->request_line.len - line_break_len);

                } else {
                    last = ngx_copy(last,
                                    mr->request_line.data,
                                    pos - mr->request_line.data);

                }

            } else {
                last = ngx_copy(last, b->start, pos - b->start);
            }
            
#if 1
            /* skip truncated header entries (if any) */
            while (last > p && last[-1] != LF) {
                last--;
            }
#endif

            j = 0;
            for (; p != last; p++) {
                if (*p == '\0') {
                    j++;
                    if (p + 1 == last) {
                        /* XXX this should not happen */
                        dd("found string end!!");

                    } else if (*(p + 1) == LF) {
                        *p = CR;

                    } else if (j % 2 == 1) {
                        *p = ':';

                    } else {
                        *p = LF;
                    }
                }
            }

            if (b == mr->header_in) {
                break;
            }
        }
    }

    *last++ = '\0';

    if (last - data > (ssize_t) size) {
        return luaL_error(L, "buffer error: %d", (int) (last - data - size));
    }

    /* strip the leading part (if any) of the request body in our header.
     * the first part of the request body could slip in because nginx core's
     * ngx_http_request_body_length_filter and etc can move r->header_in->pos
     * in case that some of the body data has been preread into r->header_in.
     */
     
    if ((p = (u_char *) ngx_strstr(data, CRLF CRLF)) != NULL) {
        last = p + sizeof(CRLF CRLF) - 1;

    } else if ((p = (u_char *) ngx_strstr(data, CRLF "\n")) != NULL) {
        last = p + sizeof(CRLF "\n") - 1;

    } else if ((p = (u_char *) ngx_strstr(data, "\n" CRLF)) != NULL) {
        last = p + sizeof("\n" CRLF) - 1;

    } else {
        for (p = last - 1; p - data >= 2; p--) {
            if (p[0] == LF && p[-1] == CR) {
                p[-1] = LF;
                last = p + 1;
                break;
            }

            if (p[0] == LF && p[-1] == LF) {
                last = p + 1;
                break;
            }
        }
    }

    lua_pushlstring(L, (char *) data, last - data);
    return 1;
}
```

下面就来解析下这个函数：

/* TODO parse and digest the function */


### ngx_http_lua_ngx_req_get_headers

这个函数将某请求的所有请求头封装在一个 Lua table 里，然后返回给调用ngx.req.get_headers(max_headers?, raw?) 的调用者，特别地，当 raw 是 false 的时候，该 table 的 key 全为小写。

```c
static int
ngx_http_lua_ngx_req_get_headers(lua_State *L)
{
    ngx_list_part_t              *part;
    ngx_table_elt_t              *header;
    ngx_http_request_t           *r;
    ngx_uint_t                    i;
    int                           n;
    int                           max;
    int                           raw = 0;
    int                           count = 0;

    n = lua_gettop(L);

    if (n >= 1) {
        if (lua_isnil(L, 1)) {
            max = NGX_HTTP_LUA_MAX_HEADERS;

        } else {
            max = luaL_checkinteger(L, 1);
        }

        if (n >= 2) {
            raw = lua_toboolean(L, 2);
        }

    } else {
        max = NGX_HTTP_LUA_MAX_HEADERS;
    }

    /* NGX_HTTP_LUA_MAX_HEADERS default to 100 */

    r = ngx_http_lua_get_req(L);
    if (r == NULL) {
        return luaL_error(L, "no request object found");
    }

    ngx_http_lua_check_fake_request(L, r);
    
    part = &r->headers_in.headers.part;
    count = part->nelts;
    while (part->next) {
        part = part->next;
        count += part->nelts;
    }

    if (max > 0 && count > max) {
        count = max;
        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua exceeding request header limit %d", max);
    }

    lua_createtable(L, 0, count);

    if (!raw) {
        lua_pushlightuserdata(L, &ngx_http_lua_headers_metatable_key);
        lua_rawget(L, LUA_REGISTRYINDEX);
        lua_setmetatable(L, -2);
    }

    part = &r->headers_in.headers.part;
    header = part->elts;

    for (i = 0; /* void */; i++) {

        dd("stack top: %d", lua_gettop(L));

        if (i >= part->nelts) {
            if (part->next == NULL) {
                break;
            }

            part = part->next;
            header = part->elts;
            i = 0;
        }

        if (raw) {
            lua_pushlstring(L, (char *) header[i].key.data, header[i].key.len);

        } else {
            lua_pushlstring(L, (char *) header[i].lowcase_key,
                            header[i].key.len);
        }

        /* stack: table key */

        lua_pushlstring(L, (char *) header[i].value.data,
                                header[i].value.len); /* stack: table key value */

        ngx_http_lua_set_multi_value_table(L, -3);

        ngx_log_debug2(NGX_LOG_DEBUG_HTTP, r->connection->log, 0,
                       "lua request header: \"%V: %V\"",
                       &header[i].key, &header[i].value);

        if (--count == 0) {
            return 1;
        }
    }

    return 1;
}
```

首先该函数检查了传入的参数，max_headers 默认为 100 （当前版本下），raw 默认为 `false`，然后创建一张表以便存放结果；其次，当 raw 为 `false` 的时候，设置一个元表；之后就是遍历 `r->headers_in.headers` 这个 list，把对应的 key、value 设置到 table 中，特别地，这里对于重名的头，进行了整合，将这些值放在一个表中，比如：

```http
Foo: abc
Foo: def
Foo: xyz
```

`ngx.req.get_headers()["Foo"]` 返回的时候 `{"abc", "def", "xyz"}`，这个过程在函数 `ngx_http_lua_set_multi_value_table` 实现。

```c
void
ngx_http_lua_set_multi_value_table(lua_State *L, int index)
{
    if (index < 0) {
        index = lua_gettop(L) + index + 1;
    }

    lua_pushvalue(L, -2); /* stack: table key value key */
    lua_rawget(L, index);
    if (lua_isnil(L, -1)) {
        lua_pop(L, 1); /* stack: table key value */
        lua_rawset(L, index); /* stack: table */

    } else {
        if (!lua_istable(L, -1)) {
            /* just inserted one value */
            lua_createtable(L, 4, 0);
                /* stack: table key value value table */
            lua_insert(L, -2);
                /* stack: table key value table value */
            lua_rawseti(L, -2, 1);
                /* stack: table key value table */
            lua_insert(L, -2);
                /* stack: table key table value */

            lua_rawseti(L, -2, 2); /* stack: table key table */

            lua_rawset(L, index); /* stack: table */

        } else {
            /* stack: table key value table */
            lua_insert(L, -2); /* stack: table key table value */

            lua_rawseti(L, -2, lua_objlen(L, -2) + 1);
                /* stack: table key table  */
            lua_pop(L, 2); /* stack: table */
        }
    }
}
```

这个函数做的事情很简单：
先尝试访问下目标表 t1 里对应的 k1，如果得到的值 v2 为 `nil`，简单地把 v1 设置进去即可；如果 v2 不是 `nil`，说明 k1 重复了，需要整合为表，此时，如果 v2 不是一个表，那么只需创建一个表 t2，然后将 t2[1] 设置为 v2、t2[2] 设置为 v1；如果 v2 本身就是一个表了，那么只要把 v1 放到这个表 v2 的最后（数组部分）。

### ngx_http_lua_create_headers_metatable

刚才上面说到过，当 API `ngx.req.get_headers()` 的第二个参数 `raw` 不为 `true` 的时候，会设置一个元表，而该元表的 `__index` 域对应的函数就是在 `ngx_http_lua_create_headers_metatable ` 这个函数里设置的。

```c
void
ngx_http_lua_create_headers_metatable(ngx_log_t *log, lua_State *L)
{
    int rc;
    const char buf[] =
        "local tb, key = ...\n"
        "local new_key = string.gsub(string.lower(key), '_', '-')\n"
        "if new_key ~= key then return tb[new_key] else return nil end";

    lua_pushlightuserdata(L, &ngx_http_lua_headers_metatable_key);

    /* metatable for ngx.req.get_headers(_, true) and
     * ngx.resp.get_headers(_, true) */
    lua_createtable(L, 0, 1);

    rc = luaL_loadbuffer(L, buf, sizeof(buf) - 1, "=headers metatable");
    if (rc != 0) {
        ngx_log_error(NGX_LOG_ERR, log, 0,
                      "failed to load Lua code for the metamethod for "
                      "headers: %i: %s", rc, lua_tostring(L, -1));

        lua_pop(L, 3);
        return;
    }

    lua_setfield(L, -2, "__index");
    lua_rawset(L, LUA_REGISTRYINDEX);
}
```

可以看到，实际上 `__index` 对应的函数就是一串 lua chunk，把大写字母转换为小写，把 `-` 转换为 `_`，然后用转换后的 key 去寻找对应的 value。
值得一提的时候，这个元表存在 Lua 注册表的 `&ngx_http_lua_headers_metatable_key ` 这个域的位置（一个全局变量的地址，不会重复）


