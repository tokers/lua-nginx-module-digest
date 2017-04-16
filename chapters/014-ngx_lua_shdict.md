> 在 `ngx_lua` 的世界里，共享内存被组织成“共享内存字典”，也就是 `ngx.shared.DICT`，截至 0.10.8 版本，`ngx_lua` 提供了 17 个操作共享内存字典的 API，这些 API 提高了编程的灵活性，让共享内存以 Lua table 的形式展现在我们的面前，而且无需我们关心竞态问题，所有的操作都是原子性的，这大大提高了 `ngx_lua` 工程师的开发效率.  <br> <br>
本文将会着眼于共享内存字典的内部实现，对源码进行透彻的分析. 这里认为读者在阅读此文时，已经对 `Nginx` 和 `ngx_lua` 有了一定的认识. 注意，下文将会涉及到共享内存字典相关的每个函数（暂时不包括 `ffi`），因此代码量比较大！
> 

<!-- more !-->

### API

`lua_shared_dict` 允许我们添加共享内存字典. 这条指令工作在 http 配置块下.

```nginx
lua_shared_dict dog 1m;
```
<br><br><br>


`ngx_lua` 共享内存字典一共有 17 个 API.

* ngx.shared.DICT.get
* ngx.shared.DICT.get_stale
* ngx.shared.DICT.set
* ngx.shared.DICT.add
* ngx.shared.DICT.safe_add
* ngx.shared.DICT.safe_set
* ngx.shared.DICT.incr
* ngx.shared.DICT.flush_all
* ngx.shared.DICT.flush_expired
* ngx.shared.DICT.get_keys
* ngx.shared.DICT.replace
* ngx.shared.DICT.delete
* ngx.shared.DICT.lpush
* ngx.shared.DICT.rpush
* ngx.shared.DICT.lpop
* ngx.shared.DICT.rpop
* ngx.shared.DICT.llen

详细介绍请参考[文档](https://github.com/openresty/lua-nginx-module#toc179)


### internel structure

直观来看，共享内存字典就是一张 Lua table，每一对 key-value 在内部都以一个 `ngx_http_lua_shdict_node_t` 的实例存在（下文称之为节点），为了提高检索效率，所有节点被组织成红黑树，同时这些节点又都在一个 LRU 队列上，这是为了快速淘汰节点而设计的. 事实上这种设计也在其他的 Nginx 模块里出现过，比如 `ngx_http_limit_req_module`.

```c
typedef struct {
	u_char 		color; /* 事实上这块内存总是连在 ngx_rbtree_node_t 后面 */
	uint8_t 		value_type; /* 值的类型 */
	u_short 		key_len; /* 键的长度 */
	uint32_t 		value_len; /* 值的长度 */
	uint64_t 		expires; /* 过期时间 */
	ngx_queue_t  queue;   /* LRU 队列节点 */
	uint32_t		user_flags; /* 用户设置的标志 */
	u_char 		data[1]; /* 键字符串的第一个字符 */
};
```
<br><br><br>


> value_type 用一个枚举类型来表示，目前值类型有

```c
enum {
	SHDICT_TNIL = 0, /* nil */
	SHDICT_TBOOLEAN = 1, /* bool */
	SHDICT_TNUMBER = 3, /* number */
	SHDICT_TSTRING = 4, /* string */
	SHDICT_TLIST = 5, /* list，这是一个特殊的双端队列类型 */
};
```
<br><br><br>


> 针对每个共享内存字典，ngx_lua 对其设置了上下文信息
> 

```c
 typedef struct {
      ngx_rbtree_t                  rbtree; /* 整棵红黑树的根节点 */
      ngx_rbtree_node_t             sentinel; /* 哨兵节点 */
      ngx_queue_t                   lru_queue; /* LRU 队列的哨兵节点 */
 } ngx_http_lua_shdict_shctx_t; /* ctx in the shared memory */
 
 
 typedef struct {
      ngx_http_lua_shdict_shctx_t  *sh;
      ngx_slab_pool_t              *shpool; /* slab 分配器 */
      ngx_str_t                     name; /* 共享内存名 */
      ngx_http_lua_main_conf_t     *main_conf;
      ngx_log_t                    *log; /* log 对象 */
 } ngx_http_lua_shdict_ctx_t;
```

这两个结构体是为了维护一整块共享内存字典而设计的.<br><br><br>


> 而 ngx_http_lua_shm_zone_t 则是面向于维护共享内存本身的结构
> 

```c
typedef struct {
    ngx_log_t                   *log;
    ngx_http_lua_main_conf_t    *lmcf;
    ngx_cycle_t                 *cycle;
    ngx_shm_zone_t               zone; /* 指向的共享内存对象 */
} ngx_http_lua_shm_zone_ctx_t;
```

### Zone Initialize

`lua_shared_dict` 指令可以添加一块共享内存字典，下面来分析下这条指令的回调解析函数 `ngx_http_lua_shared_dict`.

```c
char *
ngx_http_lua_shared_dict(ngx_conf_t *cf, ngx_command_t *cmd, void *conf)
{
    ngx_http_lua_main_conf_t   *lmcf = conf;

    ngx_str_t                  *value, name;
    ngx_shm_zone_t             *zone;
    ngx_shm_zone_t            **zp;
    ngx_http_lua_shdict_ctx_t  *ctx;
    ssize_t                     size;

	/* 每一块共享内存字典都被串联在这个 shdict_zones 数组里 */
    if (lmcf->shdict_zones == NULL) {
        lmcf->shdict_zones = ngx_palloc(cf->pool, sizeof(ngx_array_t));
        if (lmcf->shdict_zones == NULL) {
            return NGX_CONF_ERROR;
        }

        if (ngx_array_init(lmcf->shdict_zones, cf->pool, 2,
                           sizeof(ngx_shm_zone_t *))
            != NGX_OK)
        {
            return NGX_CONF_ERROR;
        }
    }

    value = cf->args->elts;

    ctx = NULL;

    if (value[1].len == 0) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid lua shared dict name \"%V\"", &value[1]);
        return NGX_CONF_ERROR;
    }

    name = value[1];

    size = ngx_parse_size(&value[2]);

	/* 最小是 8k */
    if (size <= 8191) {
        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "invalid lua shared dict size \"%V\"", &value[2]);
        return NGX_CONF_ERROR;
    }
	
	/* 创建上下文结构体 */
    ctx = ngx_pcalloc(cf->pool, sizeof(ngx_http_lua_shdict_ctx_t));
    if (ctx == NULL) {
        return NGX_CONF_ERROR;
    }

    ctx->name = name;
    ctx->main_conf = lmcf;
    ctx->log = &cf->cycle->new_log;
	
	/* 调用封装了 ngx_http_shared_memory_add 的方法 */
    zone = ngx_http_lua_shared_memory_add(cf, &name, (size_t) size,
                                          &ngx_http_lua_module);
    if (zone == NULL) {
        return NGX_CONF_ERROR;
    }

    if (zone->data) {
        ctx = zone->data;

        ngx_conf_log_error(NGX_LOG_EMERG, cf, 0,
                           "lua_shared_dict \"%V\" is already defined as "
                           "\"%V\"", &name, &ctx->name);
        return NGX_CONF_ERROR;
    }

	/* 设置好初始化函数和参数 */
    zone->init = ngx_http_lua_shdict_init_zone;
    zone->data = ctx;

    zp = ngx_array_push(lmcf->shdict_zones);
    if (zp == NULL) {
        return NGX_CONF_ERROR;
    }

    *zp = zone;

    lmcf->requires_shm = 1;

    return NGX_CONF_OK;
}
```

这个函数功能比较简单，主要工作就是解析出当前添加的共享内存的名字和大小，然后将其挂到 `ngx_lua` 模块 main 配置文件下的 `shdict_zones` 数组上. 我们知道共享内存最终是通过 `ngx_cycle_t` 的 `shared_memory` 数组维护的，而这里调用了`ngx_http_lua_shared_memory_add`，这个函数则封装了 Nginx 原生的 `ngx_http_shared_memory_add` 接口. <br><br><br>


> ngx_http_lua_shared_memory_add 

```c
ngx_shm_zone_t *
ngx_http_lua_shared_memory_add(ngx_conf_t *cf, ngx_str_t *name, size_t size,
    void *tag)
{
	
	......
	
	/* 调用原生的接口添加一块共享内存，tag 则是 ngx_http_lua_module 模块结构的地址，这是 Nginx 的老技巧了 */
    zone = ngx_shared_memory_add(cf, name, (size_t) size, tag);
    if (zone == NULL) {
        return NULL;
    }

	/* 说明对应共享内存已经存在 */
    if (zone->data) {
        ctx = (ngx_http_lua_shm_zone_ctx_t *) zone->data;
        return &ctx->zone;
    }

    n = sizeof(ngx_http_lua_shm_zone_ctx_t);

    ctx = ngx_pcalloc(cf->pool, n);
    if (ctx == NULL) {
        return NULL;
    }

    ctx->lmcf = lmcf;
    ctx->log = &cf->cycle->new_log;
    ctx->cycle = cf->cycle;
	
    ngx_memcpy(&ctx->zone, zone, sizeof(ngx_shm_zone_t));
	
	/* main conf 里同样维护了一个 ngx_http_lua_zone_ctx_t 的数组 */
    zp = ngx_array_push(lmcf->shm_zones);
    if (zp == NULL) {
        return NULL;
    }

    *zp = zone;

    /* set zone init */
    zone->init = ngx_http_lua_shared_memory_init;
    zone->data = ctx;

    lmcf->requires_shm = 1;

    return &ctx->zone;
}
```

<br><br><br>


`ngx_http_lua_shared_memory_add ` 函数里我们看到了共享内存初始化的函数，即 `ngx_http_lua_shared_memory_init`，这个函数将在 `ngx_init_cycle` 里被调用.

> ngx_http_lua_shared_memory_init
> 

```c
static ngx_int_t
ngx_http_lua_shared_memory_init(ngx_shm_zone_t *shm_zone, void *data)
{

	......

    ctx = (ngx_http_lua_shm_zone_ctx_t *) shm_zone->data;
    zone = &ctx->zone;

    odata = NULL;
    /* reload */
    if (octx) {
        ozone = &octx->zone;
        odata = ozone->data;
    }

    zone->shm = shm_zone->shm;
#if defined(nginx_version) && nginx_version >= 1009000
    zone->noreuse = shm_zone->noreuse;
#endif
	
	/* 这里的 init 方法实际上就是 ngx_http_lua_shdict_init_zone */
    if (zone->init(zone, odata) != NGX_OK) {
        return NGX_ERROR;
    }
	
	......
	
    return NGX_OK;
}
```

这个函数简介调用了 `ngx_http_lua_shm_dict_init`，也就是站在共享内存字典的角度上进行初始化，所以才有了 `ngx_http_lua_shared_memory_add ` 函数里对分配出来的 zone 结构进行的复制，一个由 `ngx_cycle_t` 维护，而另外一个存在于 `ngx_http_lua_shm_zone_t`，二者的初始化方式不同.<br><br><br>



> ngx_http_lua_shdict_init_zone

```c
ngx_int_t
ngx_http_lua_shdict_init_zone(ngx_shm_zone_t *shm_zone, void *data)
{
    ngx_http_lua_shdict_ctx_t  *octx = data;

    size_t                      len;
    ngx_http_lua_shdict_ctx_t  *ctx;

    dd("init zone");

    ctx = shm_zone->data;
	
	/* reload */
    if (octx) {
        ctx->sh = octx->sh;
        ctx->shpool = octx->shpool;

        return NGX_OK;
    }
	
	/* slab 管理器存放在整块共享内存的最前面 */
    ctx->shpool = (ngx_slab_pool_t *) shm_zone->shm.addr;

    if (shm_zone->shm.exists) {
        ctx->sh = ctx->shpool->data;

        return NGX_OK;
    }
	
    ctx->sh = ngx_slab_alloc(ctx->shpool, sizeof(ngx_http_lua_shdict_shctx_t));
    if (ctx->sh == NULL) {
        return NGX_ERROR;
    }
    
   /* 
    * slab 的 data 成员用法通常是由模块自己决定的，
    * 这里用来存放 ngx_http_lua_shm_dict_shctx_t
    */
    ctx->shpool->data = ctx->sh;

	/* 初始化红黑树和 LRU 队列 */
    ngx_rbtree_init(&ctx->sh->rbtree, &ctx->sh->sentinel,
                    ngx_http_lua_shdict_rbtree_insert_value);

    ngx_queue_init(&ctx->sh->lru_queue);

    len = sizeof(" in lua_shared_dict zone \"\"") + shm_zone->shm.name.len;

    ctx->shpool->log_ctx = ngx_slab_alloc(ctx->shpool, len);
    if (ctx->shpool->log_ctx == NULL) {
        return NGX_ERROR;
    }

    ngx_sprintf(ctx->shpool->log_ctx, " in lua_shared_dict zone \"%V\"%Z",
                &shm_zone->shm.name);

#if defined(nginx_version) && nginx_version >= 1005013
    ctx->shpool->log_nomem = 0;
#endif

    return NGX_OK;
}
``` 

这个函数完成了共享内存字典的初始化，主要是红黑树和 LRU 队列的初始化.

### API Inject

`ngx_http_lua_inject_shdict_api` 函数将 API 注入到各个对应的共享内存字典.

```c
void
ngx_http_lua_inject_shdict_api(ngx_http_lua_main_conf_t *lmcf, lua_State *L)
{
    ngx_http_lua_shdict_ctx_t   *ctx;
    ngx_uint_t                   i;
    ngx_shm_zone_t             **zone;

    if (lmcf->shdict_zones != NULL) {
        lua_createtable(L, 0, lmcf->shdict_zones->nelts /* nrec */);
                /* ngx.shared */
		
		  /* 事实上所有的 API 都存放在一个新建的表，这个表将作为所有共享内存字典的元表 */
        lua_createtable(L, 0 /* narr */, 18 /* nrec */); /* shared mt */

        lua_pushcfunction(L, ngx_http_lua_shdict_get);
        lua_setfield(L, -2, "get");

        /* 此处注入其他的 API */

        lua_pushvalue(L, -1); /* shared mt mt */
        lua_setfield(L, -2, "__index"); /* shared mt */

        zone = lmcf->shdict_zones->elts;

        for (i = 0; i < lmcf->shdict_zones->nelts; i++) {
            ctx = zone[i]->data;
			  
            lua_pushlstring(L, (char *) ctx->name.data, ctx->name.len);
                /* shared mt key */
             
            /* 创建出每个共享内存对应的共享内存字典，也就是一张 Lua table */
            lua_createtable(L, 1 /* narr */, 0 /* nrec */);
                /* table of zone[i] */
            lua_pushlightuserdata(L, zone[i]); /* shared mt key ud */
            /* 这里将每个共享内存字典对应的 ngx_http_lua_shm_zone_ctx_t 存放在新建的表数组部分，为了后续使用的 API 函数里可以方便拿到这个结构 */
            lua_rawseti(L, -2, SHDICT_USERDATA_INDEX); /* {zone[i]} */
            lua_pushvalue(L, -3); /* shared mt key ud mt */
            lua_setmetatable(L, -2); /* shared mt key ud */
            lua_rawset(L, -4); /* shared mt */
        }

        lua_pop(L, 1); /* shared */

    } else {
        lua_newtable(L);    /* ngx.shared */
    }

    lua_setfield(L, -2, "shared");
}
```

该函数将共享内存字典和 API 注入到 ngx.shared 表里，涉及到一些 Lua 和 C 的交互，建议读者拿纸和笔模拟下 Lua 栈的变化.

### Normal API

初始化结束后，我们便可以在 Lua 代码里使用这些 API 了，在 `ngx_lua 0.10.6` 以前，还没有模拟 list 的操作，这里先来介绍下这些常规的 API.

> ngx.shared.DICT.set <br>
> ngx.shared.DICT.add <br>
> ngx.shared.DICT.safe_set <br>
> ngx.shared.DICT.replace <br>
> ngx.shared.DICT.delete <br>
> ngx.shared.DICT.safe_add

这 6 个 API 是向共享内存字典里设置数据.

```c
static int
ngx_http_lua_shdict_add(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_ADD);
}


static int
ngx_http_lua_shdict_safe_add(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_ADD
                                          |NGX_HTTP_LUA_SHDICT_SAFE_STORE);
}


static int
ngx_http_lua_shdict_replace(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_REPLACE);
}


static int
ngx_http_lua_shdict_set(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, 0);
}


static int
ngx_http_lua_shdict_safe_set(lua_State *L)
{
    return ngx_http_lua_shdict_set_helper(L, NGX_HTTP_LUA_SHDICT_SAFE_STORE);
}


static int
ngx_http_lua_shdict_delete(lua_State *L)
{
    int             n;

    n = lua_gettop(L);

    if (n != 2) {
        return luaL_error(L, "expecting 2 arguments, "
                          "but only seen %d", n);
    }

    lua_pushnil(L);

    return ngx_http_lua_shdict_set_helper(L, 0);
}
```

从上面的代码可以看到，这 6 个 API 都依赖于一个核心函数 `ngx_http_lua_shdict_set_helper `，通过参数来影响 `ngx_http_lua_shdict_set_helper` 函数行为.

```c
static int
ngx_http_lua_shdict_set_helper(lua_State *L, int flags)
{
    int                          i, n;
    ngx_str_t                    key;
    uint32_t                     hash;
    ngx_int_t                    rc;
    ngx_http_lua_shdict_ctx_t   *ctx;
    ngx_http_lua_shdict_node_t  *sd;
    ngx_str_t                    value;
    int                          value_type;
    double                       num;
    u_char                       c;
    lua_Number                   exptime = 0;
    u_char                      *p;
    ngx_rbtree_node_t           *node;
    ngx_time_t                  *tp;
    ngx_shm_zone_t              *zone;
    int                          forcible = 0;
                         /* indicates whether to foricibly override other
                          * valid entries */
    int32_t                      user_flags = 0;
    ngx_queue_t                 *queue, *q;

    n = lua_gettop(L);

    if (n != 3 && n != 4 && n != 5) {
        return luaL_error(L, "expecting 3, 4 or 5 arguments, "
                          "but only seen %d", n);
    }

    if (lua_type(L, 1) != LUA_TTABLE) {
        return luaL_error(L, "bad \"zone\" argument");
    }
	
	/* 还记得上文讲到 API 注入的过程吗？每个共享内存字典的上下文保存在
	 * 充当字典的那张表里（ngx.shared.dog）
	 */
    zone = ngx_http_lua_shdict_get_zone(L, 1);
    if (zone == NULL) {
        return luaL_error(L, "bad \"zone\" argument");
    }

    ctx = zone->data;

    if (lua_isnil(L, 2)) {
        lua_pushnil(L);
        lua_pushliteral(L, "nil key");
        return 2;
    }
	
    key.data = (u_char *) luaL_checklstring(L, 2, &key.len);

    if (key.len == 0) {
        lua_pushnil(L);
        lua_pushliteral(L, "empty key");
        return 2;
    }

    if (key.len > 65535) {
        lua_pushnil(L);
        lua_pushliteral(L, "key too long");
        return 2;
    }
	
	/* 根据 crc 循环冗余计算出来的 hash 值用在红黑树节点之间的对比上 */
    hash = ngx_crc32_short(key.data, key.len);

    value_type = lua_type(L, 3);
     
    /* 校验值类型 */
    switch (value_type) {

    case SHDICT_TSTRING:
        value.data = (u_char *) lua_tolstring(L, 3, &value.len);
        break;

    case SHDICT_TNUMBER:
        value.len = sizeof(double);
        num = lua_tonumber(L, 3);
        value.data = (u_char *) &num;
        break;

    case SHDICT_TBOOLEAN:
        value.len = sizeof(u_char);
        c = lua_toboolean(L, 3) ? 1 : 0;
        value.data = &c;
        break;
	
	/* 比如 ngx.shared.DICT.delete */
    case LUA_TNIL:
        if (flags & (NGX_HTTP_LUA_SHDICT_ADD|NGX_HTTP_LUA_SHDICT_REPLACE)) {
            lua_pushnil(L);
            lua_pushliteral(L, "attempt to add or replace nil values");
            return 2;
        }

        ngx_str_null(&value);
        break;

    default:
        lua_pushnil(L);
        lua_pushliteral(L, "bad value type");
        return 2;
    }

    if (n >= 4) {
        exptime = luaL_checknumber(L, 4);
        if (exptime < 0) {
            return luaL_error(L, "bad \"exptime\" argument");
        }
    }

    if (n == 5) {
        user_flags = (uint32_t) luaL_checkinteger(L, 5);
    }
		
	/* 以阻塞进程的方式获取锁 */
    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
	 
	 /* 
	  * 淘汰一些节点，第二个参数 n = 1 时则严格根据过期时间淘汰 1~2 个节点，
	  * n = 0 时会强行淘汰最近最久未使用的节点，然后再根据过期时间淘汰 1~2 个节点
	  */
    ngx_http_lua_shdict_expire(ctx, 1);
#endif
	
	/*
	 * 寻找 key 对应的节点
	 * 返回值 NGX_DONE 表示节点不存在
	 * NGX_DECLINED 表示节点存在但是已经过期
	 * NGX_OK 表示节点存在且未过期
	 */
    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);
	
	/* ngx.shared.dog:replace 方法 */
    if (flags & NGX_HTTP_LUA_SHDICT_REPLACE) {
			
		 /* 当节点不存在或者已经过期，直接返回 false 和 "not found"，显然 forcible 是 0 */
        if (rc == NGX_DECLINED || rc == NGX_DONE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "not found");
            lua_pushboolean(L, forcible);
            return 3;
        }

        /* rc == NGX_OK */

        goto replace;
    }
	
	/* ngx.shared.dog:add */
    if (flags & NGX_HTTP_LUA_SHDICT_ADD) {
		
		/* 当节点存在且没过期，返回 false 和 exists，这会 replace 刚好相反 */
        if (rc == NGX_OK) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "exists");
            lua_pushboolean(L, forcible);
            return 3;
        }

        if (rc == NGX_DONE) {
            /* exists but expired */

            dd("go to replace");
            goto replace;
        }

        /* rc == NGX_DECLINED */

        dd("go to insert");
        goto insert;
    }

    if (rc == NGX_OK || rc == NGX_DONE) {
		
		 /*
		  * 如果值为 nil，则删除节点
		  * 其他 API 的情况，先尝试在原来节点的基础上替代
		  * 无法替代，则删除节点，再插入节点 */
        if (value_type == LUA_TNIL) {
            goto remove;
        }

replace:
		
		 /* 当值的长度一样，可以直接复用结构，提高效率，当然当值是 list 的时候要排除 */
        if (value.data
            && value.len == (size_t) sd->value_len
            && sd->value_type != SHDICT_TLIST)
        {

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                           "lua shared dict set: found old entry and value "
                           "size matched, reusing it");
			  
			  /* 把节点移动到队首，因为刚刚访问过 */
            ngx_queue_remove(&sd->queue);
            ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

            sd->key_len = (u_short) key.len;
				
			  /* 设置好过期时间，0 则是永远不过期 */
            if (exptime > 0) {
                tp = ngx_timeofday();
                sd->expires = (uint64_t) tp->sec * 1000 + tp->msec
                              + (uint64_t) (exptime * 1000);

            } else {
                sd->expires = 0;
            }

            sd->user_flags = user_flags;

            sd->value_len = (uint32_t) value.len;

            dd("setting value type to %d", value_type);

            sd->value_type = (uint8_t) value_type;

            p = ngx_copy(sd->data, key.data, key.len);
            ngx_memcpy(p, value.data, value.len);

            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 1);
            lua_pushnil(L);
            lua_pushboolean(L, forcible);
            return 3;
        }

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: found old entry but value size "
                       "NOT matched, removing it first");

remove:
		 
		 /* list 情况比较特殊，因为此时值存放的是这个 list 的哨兵节点，需要遍历列表，删除所有元素，对 queue 不清楚的读者可以取阅读 ngx_queue_t 的相关的实现 */
        if (sd->value_type == SHDICT_TLIST) {
            queue = ngx_http_lua_shdict_get_list_head(sd, key.len);

            for (q = ngx_queue_head(queue);
                 q != ngx_queue_sentinel(queue);
                 q = ngx_queue_next(q))
            {
                p = (u_char *) ngx_queue_data(q,
                                              ngx_http_lua_shdict_list_node_t,
                                              queue);

                ngx_slab_free_locked(ctx->shpool, p);
            }
        }

		 /* 从 LRU 队列和红黑树上删除节点 */
        ngx_queue_remove(&sd->queue);

        node = (ngx_rbtree_node_t *)
                   ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));
			
        ngx_rbtree_delete(&ctx->sh->rbtree, node);
			
		 /* 当然需要让 slab 管理器回收内存 */
        ngx_slab_free_locked(ctx->shpool, node);

    }

insert:

    /* rc == NGX_DECLINED or value size unmatch */
	
	/* value 为 nil 等于不需要插入操作 */
    if (value.data == NULL) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 1);
        lua_pushnil(L);
        lua_pushboolean(L, 0);
        return 3;
    }

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict set: creating a new entry");
	
	/* 
	 * 这些结构都是存放在一块连续内存里的，大小则是一个红黑树节点的大小
	 * 外加一个 ngx_http_lua_shdict_node_t 的大小，外加键和值的长度
	 */
    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + value.len;

    dd("overhead = %d", (int) (offsetof(ngx_rbtree_node_t, color)
       + offsetof(ngx_http_lua_shdict_node_t, data)));

    node = ngx_slab_alloc_locked(ctx->shpool, n);

    if (node == NULL) {
		 /* 
		  * 当共享内存不够的时候，考虑下 flag 标志，
		  * 如果是安全操作（safe_add 或者 safe_set），则直接返回
		  */
        if (flags & NGX_HTTP_LUA_SHDICT_SAFE_STORE) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushboolean(L, 0);
            lua_pushliteral(L, "no memory");
            return 2;
        }

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict set: overriding non-expired items "
                       "due to memory shortage for entry \"%V\"", &key);
		 /* 否则尝试淘汰一些节点再插入 */
        for (i = 0; i < 30; i++) {
            /* 上文说过，n = 0 会强行淘汰一个节点 */
            if (ngx_http_lua_shdict_expire(ctx, 0) == 0) {
                /* 实际上就是节点都被清空的情况 */
                break;
            }

            forcible = 1;

            node = ngx_slab_alloc_locked(ctx->shpool, n);
            if (node != NULL) {
                goto allocated;
            }
        }

        ngx_shmtx_unlock(&ctx->shpool->mutex);
		  
		 /* 经过强行淘汰还是分配失败，直接返回 */
        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        lua_pushboolean(L, forcible);
        return 3;
    }

allocated:
	
	/* 紧接着 ngx_rbtree_node_t */
    sd = (ngx_http_lua_shdict_node_t *) &node->color;

    node->key = hash;
    sd->key_len = (u_short) key.len;

    if (exptime > 0) {
        tp = ngx_timeofday();
        sd->expires = (uint64_t) tp->sec * 1000 + tp->msec
                      + (uint64_t) (exptime * 1000);

    } else {
        sd->expires = 0;
    }

    sd->user_flags = user_flags;

    sd->value_len = (uint32_t) value.len;

    dd("setting value type to %d", value_type);

    sd->value_type = (uint8_t) value_type;
	
	/* key 存放在 sd->data 指向的内存 */
    p = ngx_copy(sd->data, key.data, key.len);
    /* value 则紧紧存放在 key 后面 */
    ngx_memcpy(p, value.data, value.len);
	
	/* 插入到红黑树和 LRU 队列 */
    ngx_rbtree_insert(&ctx->sh->rbtree, node);

    ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushboolean(L, 1);
    lua_pushnil(L);
    lua_pushboolean(L, forcible);
    return 3;
}
```

`ngx_http_lua_shdict_set_helper` 函数是插入操作的核心，它总共考虑了 5 种不同的情况，根据不同的需求做出不同的操作. 不论是 `set` 也好，`add` 也好，亦或是 `safe_set` 等等，其操作核心思路都是一致的：在已有的红黑树上遍历，如果找到，则如何操作（可以复用则复用，无法复用则先删除）；如果没找到，需要创建新节点，创建新节点的时候如果内存不够，进行淘汰，新节点创建完成并且初始化后，插入到红黑树上，挂到 LRU 队列的首部. <br><br><br>

> ngx.shared.DICT.get <br>
> ngx.shared.DICT.get_stale
> 


这两个函数内部最核心的函数则是 `ngx_http_lua_shdict_get_helper`

```c
static int
ngx_http_lua_shdict_get_helper(lua_State *L, int get_stale)
{
	......

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    if (!get_stale) {
    	/* 如果是 ngx.shared.DICT.get 操作，先尝试过期 1~2 个节点 */
        ngx_http_lua_shdict_expire(ctx, 1);
    }
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returns %d", (int) rc);
		
	/* 节点没找到，或者节点已经过期但是操作是 ngx.shared.DICT.get，直接 return nil */
    if (rc == NGX_DECLINED || (rc == NGX_DONE && !get_stale)) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushnil(L);
        return 1;
    }

    /* rc == NGX_OK || (rc == NGX_DONE && get_stale) */

    value_type = sd->value_type;

    dd("data: %p", sd->data);
    dd("key len: %d", (int) sd->key_len);
	
	/* 取出 value */
    value.data = sd->data + sd->key_len;
    value.len = (size_t) sd->value_len;

    switch (value_type) {

    case SHDICT_TSTRING:

        lua_pushlstring(L, (char *) value.data, value.len);
        break;

    case SHDICT_TNUMBER:

        if (value.len != sizeof(double)) {

            ngx_shmtx_unlock(&ctx->shpool->mutex);

            return luaL_error(L, "bad lua number value size found for key %s "
                              "in shared_dict %s: %lu", key.data, name.data,
                              (unsigned long) value.len);
        }

        ngx_memcpy(&num, value.data, sizeof(double));

        lua_pushnumber(L, num);
        break;

    case SHDICT_TBOOLEAN:

        if (value.len != sizeof(u_char)) {

            ngx_shmtx_unlock(&ctx->shpool->mutex);

            return luaL_error(L, "bad lua boolean value size found for key %s "
                              "in shared_dict %s: %lu", key.data, name.data,
                              (unsigned long) value.len);
        }

        c = *value.data;

        lua_pushboolean(L, c ? 1 : 0);
        break;
	
	/* list 有自己的一套操作，不允许使用 ngx.shared.DICT.get 和 ngx.shared.DICT.get_stale */
    case SHDICT_TLIST:

        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushnil(L);
        lua_pushliteral(L, "value is a list");
        return 2;

    default:
			
		/* 其他情况抛出异常 */
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        return luaL_error(L, "bad value type found for key %s in "
                          "shared_dict %s: %d", key.data, name.data,
                          value_type);
    }

    user_flags = sd->user_flags;

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    if (get_stale) {

        /* always return value, flags, stale */

        if (user_flags) {
            lua_pushinteger(L, (lua_Integer) user_flags);

        } else {
            lua_pushnil(L);
        }

        lua_pushboolean(L, rc == NGX_DONE);
        return 3;
    }

    if (user_flags) {
        lua_pushinteger(L, (lua_Integer) user_flags);
        return 2;
    }

    return 1;
}

```


这个函数总体来讲非常简单，只要针对是否能读取过期数据进行判断即可，具体读者可以参阅注释. <br><br><br>


> ngx.shared.DICT.inrc <br>
> ngx.shared.DICT.get_keys <br>
> ngx.shared.DICT.flush_all <br>
> ngx.shared.DICT.flush_expire
> 

这四个 API 实现也十分简单，具体请看下面的注释.

```c
static int
ngx_http_lua_shdict_incr(lua_State *L)
{
    
    ......
    
    if (n == 4) {
        init = luaL_checknumber(L, 4);
    }

    dd("looking up key %.*s in shared dict %.*s", (int) key.len, key.data,
       (int) ctx->name.len, ctx->name.data);

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    ngx_http_lua_shdict_expire(ctx, 1);
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);

	/* 节点不存在或者过期*/
    if (rc == NGX_DECLINED || rc == NGX_DONE) {
			
		/* 没有携带 init 参数 */
        if (n == 3) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushnil(L);
            lua_pushliteral(L, "not found");
            return 2;
        }

        /* add value */
        num = value + init;
			
		 /* 过期 */
        if (rc == NGX_DONE) {

            /* found an expired item */
			  
			  /* 该过期节点原本存放的值类型就是数值 */
            if ((size_t) sd->value_len == sizeof(double)
                && sd->value_type != SHDICT_TLIST)
            {
                ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                               "lua shared dict incr: found old entry and "
                               "value size matched, reusing it");

                ngx_queue_remove(&sd->queue);
                ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

                dd("go to setvalue");
                /* 设置值 */
                goto setvalue;
            }

            dd("go to remove");
            goto remove;
        }

        dd("go to insert");
        goto insert;
    }

    /* rc == NGX_OK */
	 /* 节点没过期，然而原本值不是数值 */
    if (sd->value_type != SHDICT_TNUMBER || sd->value_len != sizeof(double)) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushnil(L);
        lua_pushliteral(L, "not a number");
        return 2;
    }

    ngx_queue_remove(&sd->queue);
    ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

    dd("setting value type to %d", (int) sd->value_type);

    p = sd->data + key.len;

    ngx_memcpy(&num, p, sizeof(double));
    num += value;

    ngx_memcpy(p, (double *) &num, sizeof(double));

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushnumber(L, num);
    lua_pushnil(L);
    return 2;

remove:

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict incr: found old entry but value size "
                   "NOT matched, removing it first");
	
	/* 原本值是列表则需要遍历列表删除节点 */
    if (sd->value_type == SHDICT_TLIST) {
        queue = ngx_http_lua_shdict_get_list_head(sd, key.len);

        for (q = ngx_queue_head(queue);
             q != ngx_queue_sentinel(queue);
             q = ngx_queue_next(q))
        {
            p = (u_char *) ngx_queue_data(q,
                                          ngx_http_lua_shdict_list_node_t,
                                          queue);

            ngx_slab_free_locked(ctx->shpool, p);
        }
    }

    ngx_queue_remove(&sd->queue);

    node = (ngx_rbtree_node_t *)
               ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

    ngx_rbtree_delete(&ctx->sh->rbtree, node);

    ngx_slab_free_locked(ctx->shpool, node);

insert:

    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict incr: creating a new entry");

    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + sizeof(double);
	
	/* 分配一个节点 */
    node = ngx_slab_alloc_locked(ctx->shpool, n);

    if (node == NULL) {

        ngx_log_debug1(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict incr: overriding non-expired items "
                       "due to memory shortage for entry \"%V\"", &key);

        for (i = 0; i < 30; i++) {
        	  /* 无条件淘汰一个节点，再根据过期时间淘汰 1~2 个节点 */
            if (ngx_http_lua_shdict_expire(ctx, 0) == 0) {
                break;
            }

            forcible = 1;

            node = ngx_slab_alloc_locked(ctx->shpool, n);
            if (node != NULL) {
                goto allocated;
            }
        }
	     
	     /* 内存不足 */
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        lua_pushboolean(L, forcible);
        return 3;
    }

allocated:
	 
	 /* 设置好键和值 */
    sd = (ngx_http_lua_shdict_node_t *) &node->color;

    node->key = hash;

    sd->key_len = (u_short) key.len;

    sd->value_len = (uint32_t) sizeof(double);

    ngx_rbtree_insert(&ctx->sh->rbtree, node);

    ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

setvalue:

    sd->user_flags = 0;

    sd->expires = 0; /* 通过 incr 插入的节点，没有过期时间 */

    dd("setting value type to %d", LUA_TNUMBER);

    sd->value_type = (uint8_t) LUA_TNUMBER;

    p = ngx_copy(sd->data, key.data, key.len);
    ngx_memcpy(p, (double *) &num, sizeof(double));

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushnumber(L, num);
    lua_pushnil(L);
    lua_pushboolean(L, forcible);
    return 3;
}


static int
ngx_http_lua_shdict_get_keys(lua_State *L)
{
	 ......

    if (n == 2) {
        attempts = luaL_checkint(L, 2);
    }

    ctx = zone->data;

    ngx_shmtx_lock(&ctx->shpool->mutex);
	
	 /* 空 */
    if (ngx_queue_empty(&ctx->sh->lru_queue)) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_createtable(L, 0, 0);
        return 1;
    }

    tp = ngx_timeofday();

    now = (uint64_t) tp->sec * 1000 + tp->msec;

    /* first run through: get total number of elements we need to allocate */

    q = ngx_queue_last(&ctx->sh->lru_queue);

    while (q != ngx_queue_sentinel(&ctx->sh->lru_queue)) {
        prev = ngx_queue_prev(q);

        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);
		 
		 /* 挑出还没过期的 */
        if (sd->expires == 0 || sd->expires > now) {
            total++;
            
            /* 至多拿 attemps 个 key */
            if (attempts && total == attempts) {
                break;
            }
        }

        q = prev;
    }

    lua_createtable(L, total, 0);

    /* second run through: add keys to table */

    total = 0;
    q = ngx_queue_last(&ctx->sh->lru_queue);

    while (q != ngx_queue_sentinel(&ctx->sh->lru_queue)) {
        prev = ngx_queue_prev(q);

        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);

        if (sd->expires == 0 || sd->expires > now) {
            lua_pushlstring(L, (char *) sd->data, sd->key_len);
            lua_rawseti(L, -2, ++total);
            if (attempts && total == attempts) {
                break;
            }
        }

        q = prev;
    }

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    /* table is at top of stack */
    return 1;
}


/* 把所有节点标记为过期 */
static int
ngx_http_lua_shdict_flush_all(lua_State *L)
{
	......

    ctx = zone->data;

    ngx_shmtx_lock(&ctx->shpool->mutex);

    for (q = ngx_queue_head(&ctx->sh->lru_queue);
         q != ngx_queue_sentinel(&ctx->sh->lru_queue);
         q = ngx_queue_next(q))
    {
        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);
        sd->expires = 1; /* 设置成 1，后续比较时必然认为节点是过期的 */
    }
	
	/* */
    ngx_http_lua_shdict_expire(ctx, 0);

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    return 0;
}


/* 将过期的节点移除 */
static int
ngx_http_lua_shdict_flush_expired(lua_State *L)
{
	 ......
	 
    if (n == 2) {
        /* 至多删除 attempts 个过期节点 */
        attempts = luaL_checkint(L, 2);
    }

    ctx = zone->data;

    ngx_shmtx_lock(&ctx->shpool->mutex);

    if (ngx_queue_empty(&ctx->sh->lru_queue)) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushnumber(L, 0);
        return 1;
    }

    tp = ngx_timeofday();

    now = (uint64_t) tp->sec * 1000 + tp->msec;

    q = ngx_queue_last(&ctx->sh->lru_queue);

    while (q != ngx_queue_sentinel(&ctx->sh->lru_queue)) {
        prev = ngx_queue_prev(q);

        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);

        if (sd->expires != 0 && sd->expires <= now) {
			  
			  /* 同样地，list 需要遍历每个元素 */
            if (sd->value_type == SHDICT_TLIST) {
                list_queue = ngx_http_lua_shdict_get_list_head(sd, sd->key_len);

                for (lq = ngx_queue_head(list_queue);
                     lq != ngx_queue_sentinel(list_queue);
                     lq = ngx_queue_next(lq))
                {
                    lnode = ngx_queue_data(lq, ngx_http_lua_shdict_list_node_t,
                                           queue);

                    ngx_slab_free_locked(ctx->shpool, lnode);
                }
            }

            ngx_queue_remove(q);

            node = (ngx_rbtree_node_t *)
                ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

            ngx_rbtree_delete(&ctx->sh->rbtree, node);
            ngx_slab_free_locked(ctx->shpool, node);
            freed++;

            if (attempts && freed == attempts) {
                break;
            }
        }

        q = prev;
    }

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushnumber(L, freed);
    return 1;
}
```


### List API

`ngx_lua` `0.10.6` 版本开始，共享内存字典上可以使用双端队列，这些操作类似于 `redis`，进一步提升了开发的灵活性，其设计原理是，内部值的类型标记为 list，同样存放在红黑树上，其他 list 上的元素也是通过 `ngx_queue_t` 组织，哨兵节点就和上一节所述的普通的 API 一致，存放在值所存放的内存位置（键的后面），每个 list 上的元素都用一个 `ngx_http_lua_shdict_list_node_t` 描述.

```c
typedef struct {
    ngx_queue_t                  queue; /* 双端队列上的位置 */
    uint32_t                     value_len; /* 值长度*/
    uint8_t                      value_type; /* 值类型 */
    u_char                       data[1]; /* 值的首地址 */
} ngx_http_lua_shdict_list_node_t;
```

提供的 API 则有：
* lpop
* rpop
* lpush
* rpush
* llen

下面对这些 API 进行分析. <br><br><br>


> ngx.shared.DICT.lpop <br>
> ngx.shared.DICT.rpop

同样的套路，这两个 API 的核心函数则是 `ngx_http_lua_shdict_pop_helper`

```c
static int
ngx_http_lua_shdict_pop_helper(lua_State *L, int flags)
{
	......

    hash = ngx_crc32_short(key.data, key.len);

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
	 /* 同样地，先尝试过期 1~2 节点 */
    ngx_http_lua_shdict_expire(ctx, 1);
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);
	 
	 /* 不存在或者过期 */
    if (rc == NGX_DECLINED || rc == NGX_DONE) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);
        lua_pushnil(L);
        return 1;
    }

    /* rc == NGX_OK */
	
	/* 节点虽然存在，但是对应的不是 list，返回 nil 和错误串 */
    if (sd->value_type != SHDICT_TLIST) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushnil(L);
        lua_pushliteral(L, "value not a list");
        return 2;
    }

    if (sd->value_len <= 0) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        return luaL_error(L, "bad lua list length found for key %s "
                          "in shared_dict %s: %lu", key.data, name.data,
                          (unsigned long) sd->value_len);
    }
	
	/* 取出哨兵节点 */
    queue = ngx_http_lua_shdict_get_list_head(sd, key.len);
	
	/* lpop */
    if (flags == NGX_HTTP_LUA_SHDICT_LEFT) {
        queue = ngx_queue_head(queue);
	 
	 /* rpop */
    } else {
        queue = ngx_queue_last(queue);
    }

    lnode = ngx_queue_data(queue, ngx_http_lua_shdict_list_node_t, queue);

    value_type = lnode->value_type;

    dd("data: %p", lnode->data);
    dd("value len: %d", (int) sd->value_len);

    value.data = lnode->data;
    value.len = (size_t) lnode->value_len;

    switch (value_type) {

    case SHDICT_TSTRING:
        /* 返回字符串 */
        lua_pushlstring(L, (char *) value.data, value.len);
        break;

    case SHDICT_TNUMBER:
		 
		 /* 返回数值 */
        if (value.len != sizeof(double)) {

            ngx_shmtx_unlock(&ctx->shpool->mutex);

            return luaL_error(L, "bad lua list node number value size found "
                              "for key %s in shared_dict %s: %lu", key.data,
                              name.data, (unsigned long) value.len);
        }

        ngx_memcpy(&num, value.data, sizeof(double));

        lua_pushnumber(L, num);
        break;

    default:

        ngx_shmtx_unlock(&ctx->shpool->mutex);

        return luaL_error(L, "bad list node value type found for key %s in "
                          "shared_dict %s: %d", key.data, name.data,
                          value_type);
    }
	 
	 /* 从 list 里移除 */
    ngx_queue_remove(queue);

    ngx_slab_free_locked(ctx->shpool, lnode);
    
    /* 事实上，当共享内存字典值是 list 的时候，这个 sd->value_len 被拿来存放 list 的元素个数，等于 1 表示整个 list 空了，需要释放哨兵节点 */
    if (sd->value_len == 1) {

        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict list: empty node after pop, "
                       "remove it");

        ngx_queue_remove(&sd->queue);

        node = (ngx_rbtree_node_t *)
                    ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

        ngx_rbtree_delete(&ctx->sh->rbtree, node);

        ngx_slab_free_locked(ctx->shpool, node);

    } else {
    	 /* sd->value_len-- */
        sd->value_len = sd->value_len - 1;
		 
		 /* 移动到 LRU 队列队首，刚刚访问过 */
        ngx_queue_remove(&sd->queue);
        ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);
    }

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    return 1;
}
```
整个函数也比较简单，整体思路就是取出 list 的哨兵节点，根据是 lpop 还是 rpop 进行节点移除即可.<br><br><br>



> lpush 和 rpush 的核心函数则是 ngx_http_lua_shdict_push_helper

```c
static int
ngx_http_lua_shdict_push_helper(lua_State *L, int flags)
{
	......

    value_type = lua_type(L, 3);

    switch (value_type) {

    case SHDICT_TSTRING:
        value.data = (u_char *) lua_tolstring(L, 3, &value.len);
        break;

    case SHDICT_TNUMBER:
        value.len = sizeof(double);
        num = lua_tonumber(L, 3);
        value.data = (u_char *) &num;
        break;

    default:
        lua_pushnil(L);
        lua_pushliteral(L, "bad value type");
        return 2;
    }

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    ngx_http_lua_shdict_expire(ctx, 1);
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);

    /* exists but expired */

    if (rc == NGX_DONE) {

        if (sd->value_type != SHDICT_TLIST) {
            /* TODO: reuse when length matched */
            
            /* 过期，值类型不是 list，则先移除 */
            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                           "lua shared dict push: found old entry and value "
                           "type not matched, remove it first");

            ngx_queue_remove(&sd->queue);

            node = (ngx_rbtree_node_t *)
                        ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

            ngx_rbtree_delete(&ctx->sh->rbtree, node);

            ngx_slab_free_locked(ctx->shpool, node);

            dd("go to init_list");
            goto init_list;
        }
		 
		 /* 节点过期，值是 list */
        ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                       "lua shared dict push: found old entry and value "
                       "type matched, reusing it");

        sd->expires = 0;

        /* free list nodes */

        queue = ngx_http_lua_shdict_get_list_head(sd, key.len);

        for (q = ngx_queue_head(queue);
             q != ngx_queue_sentinel(queue);
             q = ngx_queue_next(q))
        {
            /* TODO: reuse matched size list node */
            /* 移除之前所有的节点 */
            lnode = ngx_queue_data(q, ngx_http_lua_shdict_list_node_t, queue);
            ngx_slab_free_locked(ctx->shpool, lnode);
        }

        ngx_queue_init(queue);

        ngx_queue_remove(&sd->queue);
        ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

        dd("go to push_node");
        goto push_node;
    }

    /* exists and not expired */

    if (rc == NGX_OK) {
		 
		 /* 不是 list */
        if (sd->value_type != SHDICT_TLIST) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushnil(L);
            lua_pushliteral(L, "value not a list");
            return 2;
        }

        queue = ngx_http_lua_shdict_get_list_head(sd, key.len);

        ngx_queue_remove(&sd->queue);
        ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

        dd("go to push_node");
        goto push_node;
    }

    /* rc == NGX_DECLINED, not found */

init_list:
	
	/* 分配一个节点 */
    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict list: creating a new entry");

    /* NOTICE: we assume the begin point aligned in slab, be careful */
    n = offsetof(ngx_rbtree_node_t, color)
        + offsetof(ngx_http_lua_shdict_node_t, data)
        + key.len
        + sizeof(ngx_queue_t);

    dd("length before aligned: %d", n);

    n = (int) (uintptr_t) ngx_align_ptr(n, NGX_ALIGNMENT);

    dd("length after aligned: %d", n);

    node = ngx_slab_alloc_locked(ctx->shpool, n);

    if (node == NULL) {
        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        return 2;
    }

    sd = (ngx_http_lua_shdict_node_t *) &node->color;

    queue = ngx_http_lua_shdict_get_list_head(sd, key.len);

    node->key = hash;
    sd->key_len = (u_short) key.len;

    sd->expires = 0;

    sd->value_len = 0; /* 当前队列是空 */

    dd("setting value type to %d", (int) SHDICT_TLIST);

    sd->value_type = (uint8_t) SHDICT_TLIST;

    ngx_memcpy(sd->data, key.data, key.len);

    ngx_queue_init(queue);

    ngx_rbtree_insert(&ctx->sh->rbtree, node);

    ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

push_node:
 	
 	/* 推入一个节点 */
    ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                   "lua shared dict list: creating a new list node");

    n = offsetof(ngx_http_lua_shdict_list_node_t, data)
        + value.len;

    dd("list node length: %d", n);

    lnode = ngx_slab_alloc_locked(ctx->shpool, n);

    if (lnode == NULL) {

        if (sd->value_len == 0) {

            ngx_log_debug0(NGX_LOG_DEBUG_HTTP, ctx->log, 0,
                           "lua shared dict list: no memory for create"
                           " list node and list empty, remove it");
                           
			  /* 分配失败，list 空的时候把整个节点都删除 */
            ngx_queue_remove(&sd->queue);

            node = (ngx_rbtree_node_t *)
                        ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

            ngx_rbtree_delete(&ctx->sh->rbtree, node);

            ngx_slab_free_locked(ctx->shpool, node);
        }

        ngx_shmtx_unlock(&ctx->shpool->mutex);

        lua_pushboolean(L, 0);
        lua_pushliteral(L, "no memory");
        return 2;
    }

    dd("setting list length to %d", sd->value_len + 1);

    sd->value_len = sd->value_len + 1;

    dd("setting list node value length to %d", (int) value.len);

    lnode->value_len = (uint32_t) value.len;

    dd("setting list node value type to %d", value_type);

    lnode->value_type = (uint8_t) value_type;

    ngx_memcpy(lnode->data, value.data, value.len);
	
	/* lpush */
    if (flags == NGX_HTTP_LUA_SHDICT_LEFT) {
        ngx_queue_insert_head(queue, &lnode->queue);

    } else {
    /* rpush */
        ngx_queue_insert_tail(queue, &lnode->queue);
    }

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushnumber(L, sd->value_len);
    return 1;
}
```

这个函数的流程也是先从目前的红黑树上找到节点，判断之前存放的是否过期，过期的时候，如果之前也是 list，则在移除所有的 list 元素后，复用节点，然后推入新元素；如果不是 list，则删除原来的节点，然后初始化 list、push 元素；如果不过期且原来不是 list，返回 `nil` 和错误串，否则直接在原来基础上 push 元素即可. <br><br><br>


> ngx.shared.DICT.llen

```c
static int
ngx_http_lua_shdict_llen(lua_State *L)
{
    ......

    hash = ngx_crc32_short(key.data, key.len);

    ngx_shmtx_lock(&ctx->shpool->mutex);

#if 1
    ngx_http_lua_shdict_expire(ctx, 1);
#endif

    rc = ngx_http_lua_shdict_lookup(zone, hash, key.data, key.len, &sd);

    dd("shdict lookup returned %d", (int) rc);

    if (rc == NGX_OK) {
		  
		  /* 节点存在且不是 list */
        if (sd->value_type != SHDICT_TLIST) {
            ngx_shmtx_unlock(&ctx->shpool->mutex);

            lua_pushnil(L);
            lua_pushliteral(L, "value not a list");
            return 2;
        }

        ngx_queue_remove(&sd->queue);
        ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

        ngx_shmtx_unlock(&ctx->shpool->mutex);
        
        /* sd->value_len 压栈 */
        lua_pushnumber(L, (lua_Number) sd->value_len);
        return 1;
    }

    ngx_shmtx_unlock(&ctx->shpool->mutex);

    lua_pushnumber(L, 0);
    return 1;
}
```


### auxiliary function 

上面介绍的这些函数，用到了一些辅助函数，有：

> ngx_http_lua_shdict_get_zone <br>
> ngx_http_lua_shdict_expire <br>
> ngx_http_lua_shdict_get_list_head <br> 
> ngx_http_lua_shdict_lookup


下面来对这些辅助函数进行下分析.

```c
static ngx_inline ngx_shm_zone_t *
ngx_http_lua_shdict_get_zone(lua_State *L, int index)
{
    ngx_shm_zone_t      *zone;
    
    /*
     * 记得上文讲 API 的 inject 的部分吗，
     * 每个共享内存字典的上下文结构体起始就放在提供给用户的
     * Lua table 的数组部分
     */
    lua_rawgeti(L, index, SHDICT_USERDATA_INDEX);
    zone = lua_touserdata(L, -1);
    lua_pop(L, 1);

    return zone;
}


static int
ngx_http_lua_shdict_expire(ngx_http_lua_shdict_ctx_t *ctx, ngx_uint_t n)
{
    ngx_time_t                      *tp;
    uint64_t                         now;
    ngx_queue_t                     *q, *list_queue, *lq;
    int64_t                          ms;
    ngx_rbtree_node_t               *node;
    ngx_http_lua_shdict_node_t      *sd;
    int                              freed = 0;
    ngx_http_lua_shdict_list_node_t *lnode;

    tp = ngx_timeofday();

    now = (uint64_t) tp->sec * 1000 + tp->msec;

    /*
     * n == 1 deletes one or two expired entries
     * n == 0 deletes oldest entry by force
     *        and one or two zero rate entries
     */

    while (n < 3) {
        
        /* 整个共享内存字典空了 */
        if (ngx_queue_empty(&ctx->sh->lru_queue)) {
            return freed;
        }
		 
		 /* 队尾的节点是最近最久未使用的 */
        q = ngx_queue_last(&ctx->sh->lru_queue);

        sd = ngx_queue_data(q, ngx_http_lua_shdict_node_t, queue);
		 
		 /* 参数 n 大于 0 */
        if (n++ != 0) {
            
            /* 永不过期的节点 */
            if (sd->expires == 0) {
                return freed;
            }

            ms = sd->expires - now;
            if (ms > 0) {
                /* 过期时间还没到 */
                return freed;
            }
        }
        /* 如果参数 n 传入时为 0，则必然会淘汰一个节点 */
        /* 对于 list 需要特殊处理，遍历整个 list，删除元素 */
        if (sd->value_type == SHDICT_TLIST) {
            list_queue = ngx_http_lua_shdict_get_list_head(sd, sd->key_len);

            for (lq = ngx_queue_head(list_queue);
                 lq != ngx_queue_sentinel(list_queue);
                 lq = ngx_queue_next(lq))
            {
                lnode = ngx_queue_data(lq, ngx_http_lua_shdict_list_node_t,
                                       queue);

                ngx_slab_free_locked(ctx->shpool, lnode);
            }
        }
        
        /* 删除节点 */
        ngx_queue_remove(q);

        node = (ngx_rbtree_node_t *)
                   ((u_char *) sd - offsetof(ngx_rbtree_node_t, color));

        ngx_rbtree_delete(&ctx->sh->rbtree, node);

        ngx_slab_free_locked(ctx->shpool, node);

        freed++;
    }

    return freed;
}


static ngx_inline ngx_queue_t *
ngx_http_lua_shdict_get_list_head(ngx_http_lua_shdict_node_t *sd, size_t len)
{
	 /* 其实就是把节点存放值的地址取出来（有对齐），转换成 ngx_queue_t 的类型 */
    return (ngx_queue_t *) ngx_align_ptr(((u_char *) &sd->data + len),
                                         NGX_ALIGNMENT);
}


/* 检索节点 */
static ngx_int_t
ngx_http_lua_shdict_lookup(ngx_shm_zone_t *shm_zone, ngx_uint_t hash,
    u_char *kdata, size_t klen, ngx_http_lua_shdict_node_t **sdp)
{
    ngx_int_t                    rc;
    ngx_time_t                  *tp;
    uint64_t                     now;
    int64_t                      ms;
    ngx_rbtree_node_t           *node, *sentinel;
    ngx_http_lua_shdict_ctx_t   *ctx;
    ngx_http_lua_shdict_node_t  *sd;

    ctx = shm_zone->data;

    node = ctx->sh->rbtree.root;
    sentinel = ctx->sh->rbtree.sentinel;

    while (node != sentinel) {

        if (hash < node->key) { /* 左子树 */
            node = node->left;
            continue;
        }

        if (hash > node->key) { /* 右子树 */
            node = node->right;
            continue;
        }

        /* hash == node->key */

        sd = (ngx_http_lua_shdict_node_t *) &node->color;

        rc = ngx_memn2cmp(kdata, sd->data, klen, (size_t) sd->key_len);

        if (rc == 0) {
            /* HIT */
            ngx_queue_remove(&sd->queue);
            ngx_queue_insert_head(&ctx->sh->lru_queue, &sd->queue);

            *sdp = sd;

            dd("node expires: %lld", (long long) sd->expires);

            if (sd->expires != 0) {
            	   /* 判断是否过期 */
                tp = ngx_timeofday();

                now = (uint64_t) tp->sec * 1000 + tp->msec;
                ms = sd->expires - now;

                dd("time to live: %lld", (long long) ms);

                if (ms < 0) {
                    /* 过期 */
                    dd("node already expired");
                    return NGX_DONE;
                }
            }

            return NGX_OK;
        }

        node = (rc < 0) ? node->left : node->right;
    }

    *sdp = NULL;
    
    /* 没有找到 */
    return NGX_DECLINED;
}
```

