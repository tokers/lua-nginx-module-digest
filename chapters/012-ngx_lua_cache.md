> 以下代码均出自 lua-nginx-module v0.10.7 版本

> 这篇文章主要深入介绍 `lua-nginx-module` 是怎么对 Lua 代码进行加载和缓存
> 
`ngx_http_lua_cache_loadbuffer ` 这个函数是提供给诸如 `rewrite_by_lua`，`access_by_lua` 等指令的回调方法在 load 对应的 Lua code 的时候使用的，这个函数也在 [rewrite_by_lua](003-ngx_lua_rewrite_by_lua.md) 这一节里简单讲过，本节主要介绍 `ngx_http_lua_cache_load_code`，`ngx_http_lua_clfactory_loadbuffer` 和 `ngx_http_lua_cache_store_code` 这三个函数

### 缓存状态机

首先得来回顾下 `ngx_http_lua_cache_loadbuffer ` 这个函数的缓存状态机

- 调用 ngx_http_lua_cache_load_code，判断当前的 Lua chunk 有没有缓存，得到返回码，如果返回码为 NGX_OK，跳到第二步；如果返回码是 NGX_ERROR，跳到第三步；否则跳到第四步
- 从缓存中拿到 Lua chunk 且被压入到栈，返回 NGX_OK
- 出错，返回 NGX_ERROR
- 缓存 Miss，从原生的 Lua 代码加载，然后压栈，如果出错，记录错误日志然后返回 NGX_ERROR；否则返回 NGX_OK

缓存是万金油！！！
我们总是期望从缓存里拿到加载好的 Lua chunk 而不是笨拙地从原生代码转换成 chunk

> 插一句，参加过 ACM 的同学应该会觉得这个东西有没有很像记忆化搜索呢 :)

### ngx_http_lua_cache_load_code

这个函数的功能就是字面意思，尝试从缓存里取出 Lua chunk

```c
static ngx_int_t
ngx_http_lua_cache_load_code(ngx_log_t *log, lua_State *L,
    const char *key)
{
    int          rc;
    u_char      *err;

    /*  get code cache table */
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_rawget(L, LUA_REGISTRYINDEX);    /*  sp++ */

    dd("Code cache table to load: %p", lua_topointer(L, -1));

    if (!lua_istable(L, -1)) {
        dd("Error: code cache table to load did not exist!!");
        return NGX_ERROR;
    }

    lua_getfield(L, -1, key);    /*  sp++ */

    if (lua_isfunction(L, -1)) {
        /*  call closure factory to gen new closure */
        rc = lua_pcall(L, 0, 1, 0);
        if (rc == 0) {
            /*  remove cache table from stack, leave code chunk at
             *  top of stack */
            lua_remove(L, -2);   /*  sp-- */
            return NGX_OK;
        }

        if (lua_isstring(L, -1)) {
            err = (u_char *) lua_tostring(L, -1);

        } else {
            err = (u_char *) "unknown error";
        }

        ngx_log_error(NGX_LOG_ERR, log, 0,
                      "lua: failed to run factory at key \"%s\": %s",
                      key, err);
        lua_pop(L, 2);
        return NGX_ERROR;
    }

    dd("Value associated with given key in code cache table is not code "
       "chunk: stack top=%d, top value type=%s\n",
       lua_gettop(L), lua_typename(L, -1));
       
    /*  remove cache table and value from stack */
    lua_pop(L, 2);                                /*  sp-=2 */

    return NGX_DECLINED;
}
```

首先需要明确的一点是，`lua-nginx-module` 它把所有的 Lua chunk 存放在 Lua 提供的注册表中（Registry），通过某个键，来获取到专门存放 chunk 的 code table，而这个键就是 `ngx_http_lua_code_cache_key` 这个 char 类型的变量的地址（一个全局变量，存放在全局/静态存储区），显然它是独一无二的。然后再根据参数 key（见 [rewrite_by_lua](003-ngx_lua_rewrite_by_lua.md) 里的分析）来作为 code table 的键，把对应的 Lua chunk 拿出来并存在栈顶。然后需要判断这个 chunk 是否合法（是否是一个函数），如果合法，调用 lua_pcall 运行一次，这一步可能会令人迷惑，既然我们已经拿到了 Lua chunk，为什么要先运行一次呢？这里暂时先不解释，继续看下去就会明白了。

恩，假如顺利从缓存里拿到了 Lua chunk，就不需要再继续运行那个缓存状态机了，否则还得老老实实地从 Lua 代码加载

### ngx_http_lua_clfactory_loadbuffer

当 Cache Miss 的情况下，这个函数会在 `ngx_http_lua_cache_loadbuffer` 里调用

```c
ngx_int_t
ngx_http_lua_clfactory_loadbuffer(lua_State *L, const char *buff,
    size_t size, const char *name)
{
    ngx_http_lua_clfactory_buffer_ctx_t     ls;

    ls.s = buff;
    ls.size = size;
    ls.sent_begin = 0;
    ls.sent_end = 0;

    return lua_load(L, ngx_http_lua_clfactory_getS, &ls, name);
}
```

恩，有必要看下 `ngx_http_lua_clfactory_buffer_ctx_t` 这个结构体

```c
typedef struct {
	int         sent_begin;
	int         sent_end;
	const char *s;
	size_t      size;
} ngx_http_lua_clfactory_buffer_ctx_t;
```

- `sent_begin` 标记 lua_load 是否调用过 Reader 来获取新的 chunk 片
- `sent_end` 标记 Lua chunk 已经读取完毕
- `s` 指向的是要加载的整个 Lua chunk 串的首地址
- `size` 则是这个 Lua chunk 串的大小

这个函数最终是调用了 lua_load，把 Reader 设置为函数 `ngx_http_lua_clfactory_getS`，参数是一个 `ngx_http_lua_clfactory_buffer_ctx_t ` 的变量，name 则是最终得到的 Lua chunk 的名字，也就是我们在 [rewrite_by_lua](003-ngx_lua_rewrite_by_lua.md) 里面分析过的 `chunkname` 这个串了。接着我们得来看看这个 Reader，也就是函数 `ngx_http_lua_clfactory_getS `

### ngx_http_lua_clfactory_getS

```c
static const char *
ngx_http_lua_clfactory_getS(lua_State *L, void *ud, size_t *size)
{
    ngx_http_lua_clfactory_buffer_ctx_t      *ls = ud;

    if (ls->sent_begin == 0) {
        ls->sent_begin = 1;
        *size = CLFACTORY_BEGIN_SIZE;

        return CLFACTORY_BEGIN_CODE;
    }

    if (ls->size == 0) {
        if (ls->sent_end == 0) {
            ls->sent_end = 1;
            *size = CLFACTORY_END_SIZE;
            return CLFACTORY_END_CODE;
        }

        return NULL;
    }

    *size = ls->size;
    ls->size = 0;

    return ls->s;
}


#define CLFACTORY_BEGIN_CODE "return function() "
#define CLFACTORY_BEGIN_SIZE (sizeof(CLFACTORY_BEGIN_CODE) - 1)
#define CLFACTORY_END_CODE "\nend"
#define CLFACTORY_END_SIZE (sizeof(CLFACTORY_END_CODE) - 1)
```

我们发现，当 lua_load 首次调用这个 Reader 的时候，得到的将是字符串 "return function()"；当正式的 Lua chunk 读完以后，将会得到字符串 "\nend"。这样，实际上加载的 chunk 是一个返回函数的 Lua 语句：

```lua
return function()
	...
end
```

看到这里，大家应该可以明白上面所说的，为什么要先调用加载好的 chunk 一次，因为只有调用一次以后，我们才能得到包装着 Lua 代码的函数（当时是编译好的 chunk 了）；这么做是有原因的，毕竟 lua_pcall 运行的只能是函数而用户所写的 Lua 代码则是不可控的

到这里，代码也从 Lua code 里加载出来了，下面得把 Lua chunk 给缓存起来

```c
static ngx_int_t
ngx_http_lua_cache_store_code(lua_State *L, const char *key)
{
    int rc;

    /*  get code cache table */
    lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
    lua_rawget(L, LUA_REGISTRYINDEX);

    dd("Code cache table to store: %p", lua_topointer(L, -1));

    if (!lua_istable(L, -1)) {
        dd("Error: code cache table to load did not exist!!");
        return NGX_ERROR;
    }

    lua_pushvalue(L, -2); /* closure cache closure */
    lua_setfield(L, -2, key); /* closure cache */

    /*  remove cache table, leave closure factory at top of stack */
    lua_pop(L, 1); /* closure */

    /*  call closure factory to generate new closure */
    rc = lua_pcall(L, 0, 1, 0);
    if (rc != 0) {
        dd("Error: failed to call closure factory!!");
        return NGX_ERROR;
    }

    return NGX_OK;
}
```

流程我们应该比较熟悉了，从注册表里拿出 code table，然后把参数 key 作为键，将之前得到的 chunk 存到 code table，另外值得注意的是，我们还得调用一次这个 chunk（理由同上）

> 关于 Lua 栈的使用，可以参考 Lua 文档
> 这里解释下该函数调用过程中，Lua 栈的变化

```c
/*
|----------|
|stack peak|
|----------|
|Lua chunk |
|----------|
    ||
	||	lua_pushlightuserdata(L, &ngx_http_lua_code_cache_key);
	\/

|-------------------|
|	stack peak	    |
|-------------------|
|  cache table key  |
|-------------------|
|	Lua chunk       |
|-------------------|	
	||
	|| lua_rawget(L, LUA_REGISTRYINDEX);
	\/
	
|---------------|
|  stack peak   |
|---------------|
|  cache table  |
|---------------|
|	Lua chunk   |
|---------------|		
	||
	|| lua_pushvalue(L, -2);
	\/

|---------------|
|  stack peak   |
|---------------|
|	Lua chunk   |
|---------------|
|  cache table  |
|---------------|
|	Lua chunk   |
|---------------|
	||
	||  lua_setfield(L, -2, key);
	\/

|---------------|
|  stack peak   |
|---------------|
|  cache table  |
|---------------|
|	Lua chunk   |
|---------------|
	||
	|| lua_pop(L, 1);
	\/
	
|---------------|
|  stack peak   |
|---------------|
|	Lua chunk   |
|---------------|
	
	||
	|| rc = lua_pcall(L, 0, 1, 0);
	\/

|------------------------|
|  		stack peak       |
|------------------------|
|	Lua chunk(function)  |
|------------------------|
*/
```


### 总结

本文着重介绍了这个针对代码缓存的缓存状态机，无论何时何地，缓存始终是万金油，一次加载，之后每次都只要从缓存里拿出 Lua chunk 就行了，这大大减少了后续每个请求的处理时间