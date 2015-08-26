
Name
=======

ngx_lua - 将lua脚本能力嵌入到Nginx

*Nginx源代码不包含该模块。* 详情看这里 [the installation instructions](#installation).


Table of Contents
=================

* [Name](#name)
* [Status](#status)
* [Version](#version)
* [Synopsis](#synopsis)
* [Description](#description)
* [Tipical Uses](#typical-uses)
* [Nginx Compatibility](#nginx-compatibility)
* [Installation](#installation)
    * [C Macro Configuration](#c-macro-configurations)
    * [Installation on Ubuntu 11.10](#installation-on-ubuntu-1110)
* [Community](#community)
    * [English Mailing List](#english-mailing-list)
    * [Chinese Mailing List](#chinese-mailing-list)
* [Code Repository](#code-repository)
* [Bugs and Patches](#bugs-and-patches)
* [Lua/LuaJIT Bytecode Support](#lualuajit-bytecode-support)
* [System Environment Variable Support](#system-environment-variable-support)
* [HTTP 1.0 Support](#http-10-support)
* [Statically Linking Pure Lua Modules](#statically-linking-pure-lua-modules)
* [Data Sharing within an Nginx Worker](#data-sharing-within-an-nginx-worker)
* [Known Issues](#known-issues)
    * [TCP socket connect operation issues](#tcp-socket-connect-operation-issues)
    * [Lua Coroutine Yielding/Resuming](#lua-coroutine-yieldingresuming)
    * [Lua Variable Scope](#lua-variable-scope)
    * [Locations Configured by Subrequest Directives of Other Modules](#locations-configured-by-subrequest-directives-of-other-modules)
    * [Cosockets Not Available Everywhere](#cosockets-not-available-everywhere)
    * [Special Escaping Sequences](#special-escaping-sequences)
    * [Mixing with SSI Not Supported](#mixing-with-ssi-not-supported)
    * [SPDY Mode Not Fully Supported](#spdy-mode-not-fully-supported)
    * [Missing data on short circuited requests](#missing-data-on-short-circuited-requests)
* [TODO](#todo)
* [Changes](#changes)
* [Test Suite](#test-suite)
* [Copyright and License](#copyright-and-license)
* [See Also](#see-also)
* [Directives](#directives)
* [Nginx API for Lua](#nginx-api-for-lua)
* [Obsolete Sections](#obsolete-sections)
    * [Special PCRE Sequences](#special-pcre-sequences)

Status
======

Production ready.

Version
=======

This document describes ngx_lua [v0.9.16](https://github.com/openresty/lua-nginx-module/tags) released on 22 June 2015.

Synopsis
========
```nginx

 # 为Lua外部库指定搜索路径（';;'为默认路径）
 lua_package_path '/foo/bar/?.lua;/blah/?.lua;;';

 # 为C写的Lua外部库指定搜索路径（也可以使用';;'）
 lua_package_cpath '/bar/baz/?.so;/blah/blah/?.so;;';

 server {
     location /inline_concat {
         # 由default_type指定的MIME类型
         default_type 'text/plain';

         set $a "hello";
         set $b "world";
         # 内联lua脚本
         set_by_lua $res "return ngx.arg[1]..ngx.arg[2]" $a $b;
         echo $res;
     }

     location /rel_file_concat {
         set $a "foo";
         set $b "bar";
         # 相对于nginx安装路径的lua脚本路径
         # $ngx_prefix/conf/concat.lua contents:
         #
         #    return ngx.arg[1]..ngx.arg[2]
         #
         set_by_lua_file $res conf/concat.lua $a $b;
         echo $res;
     }

     location /abs_file_concat {
         set $a "fee";
         set $b "baz";
         # 不可更改的脚本绝对路径
         set_by_lua_file $res /usr/nginx/conf/concat.lua $a $b;
         echo $res;
     }

     location /lua_content {
         # 由default_type指定的MIME类型
         default_type 'text/plain';

         content_by_lua "ngx.say('Hello,world!')";
     }

     location /nginx_var {
         # 由default_type指定的MIME类型
         default_type 'text/plain';

         # 访问 /nginx_var?a=hello,world
         content_by_lua "ngx.print(ngx.var['arg_a'], '\\n')";
     }

     location /request_body {
          # 强制读取请求体（默认关闭）
          lua_need_request_body on;
          client_max_body_size 50k;
          client_body_buffer_size 50k;

          content_by_lua 'ngx.print(ngx.var.request_body)';
     }

     # 使用子请求在Lua中发送非阻塞IO
     location /lua {
         # 由default_type指定的MIME类型
         default_type 'text/plain';

         content_by_lua '
             local res = ngx.location.capture("/some_other_location")
             if res.status == 200 then
                 ngx.print(res.body)
             end';
     }

     # GET /recur?num=5
     location /recur {
         # 由default_type指定的MIME类型
         default_type 'text/plain';

         content_by_lua '
            local num = tonumber(ngx.var.arg_num) or 0

            if num > 50 then
                ngx.say("num too big")
                return
            end

            ngx.say("num is: ", num)

            if num > 0 then
                res = ngx.location.capture("/recur?num=" .. tostring(num - 1))
                ngx.print("status=", res.status, " ")
                ngx.print("body=", res.body)
            else
                ngx.say("end")
            end
            ';
     }

     location /foo {
         rewrite_by_lua '
             res = ngx.location.capture("/memc",
                 { args = { cmd = "incr", key = ngx.var.uri } }
             )
         ';

         proxy_pass http://blah.blah.com;
     }

     location /blah {
         access_by_lua '
             local res = ngx.location.capture("/auth")

             if res.status == ngx.HTTP_OK then
                 return
             end

             if res.status == ngx.HTTP_FORBIDDEN then
                 ngx.exit(res.status)
             end

             ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
         ';

         # proxy_pass/fastcgi_pass/postgres_pass/...
     }

     location /mixed {
         rewrite_by_lua_file /path/to/rewrite.lua;
         access_by_lua_file /path/to/access.lua;
         content_by_lua_file /path/to/content.lua;
     }

     # 在代码路径中使用Nginx var变量
     # 警告：Nginx var变量的内容必须仔细过滤，否则会有很大的安全风险！
     location ~ ^/app/([-_a-zA-Z0-9/]+) {
         set $path $1;
         content_by_lua_file /path/to/lua/app/root/$path.lua;
     }

     location / {
        lua_need_request_body on;

        client_max_body_size 100k;
        client_body_buffer_size 100k;

        access_by_lua '
            -- 在黑名单中检查客户端IP地址
            if ngx.var.remote_addr == "132.5.72.3" then
                ngx.exit(ngx.HTTP_FORBIDDEN)
            end

            -- 检查请求体中是否含有特定单词
            if ngx.var.request_body and
                     string.match(ngx.var.request_body, "fsck")
            then
                return ngx.redirect("/terms_of_use.html")
            end

            -- 测试通过
        ';

        # proxy_pass/fastcgi_pass/etc settings
     }
 }
```

[回到目录](#table-of-contents)

Description
===========

This module embeds Lua, via the standard Lua 5.1 interpreter or [LuaJIT 2.0/2.1](http://luajit.org/luajit.html), into Nginx and by leveraging Nginx's subrequests, allows the integration of the powerful Lua threads (Lua coroutines) into the Nginx event model.

Unlike [Apache's mod_lua](https://httpd.apache.org/docs/trunk/mod/mod_lua.html) and [Lighttpd's mod_magnet](http://redmine.lighttpd.net/wiki/1/Docs:ModMagnet), Lua code executed using this module can be *100% non-blocking* on network traffic as long as the [Nginx API for Lua](#nginx-api-for-lua) provided by this module is used to handle
requests to upstream services such as MySQL, PostgreSQL, Memcached, Redis, or upstream HTTP web services.

At least the following Lua libraries and Nginx modules can be used with this ngx_lua module:

* [lua-resty-memcached](https://github.com/openresty/lua-resty-memcached)
* [lua-resty-mysql](https://github.com/openresty/lua-resty-mysql)
* [lua-resty-redis](https://github.com/openresty/lua-resty-redis)
* [lua-resty-dns](https://github.com/openresty/lua-resty-dns)
* [lua-resty-upload](https://github.com/openresty/lua-resty-upload)
* [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket)
* [lua-resty-lock](https://github.com/openresty/lua-resty-lock)
* [lua-resty-string](https://github.com/openresty/lua-resty-string)
* [ngx_memc](http://github.com/openresty/memc-nginx-module)
* [ngx_postgres](https://github.com/FRiCKLE/ngx_postgres)
* [ngx_redis2](http://github.com/openresty/redis2-nginx-module)
* [ngx_redis](http://wiki.nginx.org/HttpRedisModule)
* [ngx_proxy](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)
* [ngx_fastcgi](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html)

Almost all the Nginx modules can be used with this ngx_lua module by means of [ngx.location.capture](#ngxlocationcapture) or [ngx.location.capture_multi](#ngxlocationcapture_multi) but it is recommended to use those `lua-resty-*` libraries instead of creating subrequests to access the Nginx upstream modules because the former is usually much more flexible and memory-efficient.

The Lua interpreter or LuaJIT instance is shared across all the requests in a single nginx worker process but request contexts are segregated using lightweight Lua coroutines.

Loaded Lua modules persist in the nginx worker process level resulting in a small memory footprint in Lua even when under heavy loads.

[回到目录](#table-of-contents)

Typical Uses
============

Just to name a few:

* Mashup'ing and processing outputs of various nginx upstream outputs (proxy, drizzle, postgres, redis, memcached, and etc) in Lua,
* doing arbitrarily complex access control and security checks in Lua before requests actually reach the upstream backends,
* manipulating response headers in an arbitrary way (by Lua)
* fetching backend information from external storage backends (like redis, memcached, mysql, postgresql) and use that information to choose which upstream backend to access on-the-fly,
* coding up arbitrarily complex web applications in a content handler using synchronous but still non-blocking access to the database backends and other storage,
* doing very complex URL dispatch in Lua at rewrite phase,
* using Lua to implement advanced caching mechanism for Nginx's subrequests and arbitrary locations.

The possibilities are unlimited as the module allows bringing together various elements within Nginx as well as exposing the power of the Lua language to the user. The module provides the full flexibility of scripting while offering performance levels comparable with native C language programs both in terms of CPU time as well as memory footprint. This is particularly the case when LuaJIT 2.x is enabled.

Other scripting language implementations typically struggle to match this performance level.

The Lua state (Lua VM instance) is shared across all the requests handled by a single nginx worker process to minimize memory use.

[回到目录](#table-of-contents)

Nginx Compatibility
===================
The latest module is compatible with the following versions of Nginx:

* 1.7.x (last tested: 1.7.10)
* 1.6.x
* 1.5.x (last tested: 1.5.12)
* 1.4.x (last tested: 1.4.4)
* 1.3.x (last tested: 1.3.11)
* 1.2.x (last tested: 1.2.9)
* 1.1.x (last tested: 1.1.5)
* 1.0.x (last tested: 1.0.15)
* 0.9.x (last tested: 0.9.4)
* 0.8.x >= 0.8.54 (last tested: 0.8.54)

[回到目录](#table-of-contents)

Installation
============

It is highly recommended to use the [ngx_openresty bundle](http://openresty.org) that bundles Nginx, ngx_lua,  LuaJIT 2.0/2.1 (or the optional standard Lua 5.1 interpreter), as well as a package of powerful companion Nginx modules. The basic installation step is a simple command: `./configure --with-luajit && make && make install`.

Alternatively, ngx_lua can be manually compiled into Nginx:

1. Install LuaJIT 2.0 or 2.1 (recommended) or Lua 5.1 (Lua 5.2 is *not* supported yet). LuaJIT can be downloaded from the [the LuaJIT project website](http://luajit.org/download.html) and Lua 5.1, from the [Lua project website](http://www.lua.org/).  Some distribution package managers also distribute LuajIT and/or Lua.
1. Download the latest version of the ngx_devel_kit (NDK) module [HERE](https://github.com/simpl/ngx_devel_kit/tags).
1. Download the latest version of ngx_lua [HERE](https://github.com/openresty/lua-nginx-module/tags).
1. Download the latest version of Nginx [HERE](http://nginx.org/) (See [Nginx Compatibility](#nginx-compatibility))

Build the source with this module:

```bash

 wget 'http://nginx.org/download/nginx-1.7.10.tar.gz'
 tar -xzvf nginx-1.7.10.tar.gz
 cd nginx-1.7.10/

 # tell nginx's build system where to find LuaJIT 2.0:
 export LUAJIT_LIB=/path/to/luajit/lib
 export LUAJIT_INC=/path/to/luajit/include/luajit-2.0

 # tell nginx's build system where to find LuaJIT 2.1:
 export LUAJIT_LIB=/path/to/luajit/lib
 export LUAJIT_INC=/path/to/luajit/include/luajit-2.1

 # or tell where to find Lua if using Lua instead:
 #export LUA_LIB=/path/to/lua/lib
 #export LUA_INC=/path/to/lua/include

 # Here we assume Nginx is to be installed under /opt/nginx/.
 ./configure --prefix=/opt/nginx \
         --with-ld-opt='-Wl,-rpath,/path/to/luajit-or-lua/lib' \
         --add-module=/path/to/ngx_devel_kit \
         --add-module=/path/to/lua-nginx-module

 make -j2
 make install
```

[回到目录](#table-of-contents)

C Macro Configurations
----------------------

While building this module either via OpenResty or with the NGINX core, you can define the following C macros via the C compiler options:

* `NGX_LUA_USE_ASSERT`
	When defined, will enable assertions in the ngx_lua C code base. Recommended for debugging or testing builds. It can introduce some (small) runtime overhead when enabled. This macro was first introduced in the `v0.9.10` release.
* `NGX_LUA_ABORT_AT_PANIC`
	When the Lua/LuaJIT VM panics, ngx_lua will instruct the current nginx worker process to quit gracefully by default. By specifying this C macro, ngx_lua will abort the current nginx worker process (which usually result in a core dump file) immediately. This option is useful for debugging VM panics. This option was first introduced in the `v0.9.8` release.
* `NGX_LUA_NO_FFI_API`
	Excludes pure C API functions for FFI-based Lua API for NGINX (as required by [lua-resty-core](https://github.com/openresty/lua-resty-core#readme), for example). Enabling this macro can make the resulting binary code size smaller.

To enable one or more of these macros, just pass extra C compiler options to the `./configure` script of either NGINX or OpenResty. For instance,


    ./configure --with-cc-opt="-DNGX_LUA_USE_ASSERT -DNGX_LUA_ABORT_AT_PANIC"


[回到目录](#table-of-contents)

Installation on Ubuntu 11.10
----------------------------

Note that it is recommended to use LuaJIT 2.0 or LuaJIT 2.1 instead of the standard Lua 5.1 interpreter wherever possible.

If the standard Lua 5.1 interpreter is required however, run the following command to install it from the Ubuntu repository:

```bash

 apt-get install -y lua5.1 liblua5.1-0 liblua5.1-0-dev
```

Everything should be installed correctly, except for one small tweak.

Library name `liblua.so` has been changed in liblua5.1 package, it only comes with `liblua5.1.so`, which needs to be symlinked to `/usr/lib` so it could be found during the configuration process.

```bash

 ln -s /usr/lib/x86_64-linux-gnu/liblua5.1.so /usr/lib/liblua.so
```

[回到目录](#table-of-contents)

Community
=========

[回到目录](#table-of-contents)

English Mailing List
--------------------

The [openresty-en](https://groups.google.com/group/openresty-en) mailing list is for English speakers.

[回到目录](#table-of-contents)

Chinese Mailing List
--------------------

The [openresty](https://groups.google.com/group/openresty) mailing list is for Chinese speakers.

[回到目录](#table-of-contents)

Code Repository
===============

The code repository of this project is hosted on github at [openresty/lua-nginx-module](https://github.com/openresty/lua-nginx-module).

[回到目录](#table-of-contents)

Bugs and Patches
================

Please submit bug reports, wishlists, or patches by

1. creating a ticket on the [GitHub Issue Tracker](https://github.com/openresty/lua-nginx-module/issues),
1. or posting to the [OpenResty community](#community).

[回到目录](#table-of-contents)

Lua/LuaJIT Bytecode Support
===========================

As from the `v0.5.0rc32` release, all `*_by_lua_file` configure directives (such as [content_by_lua_file](#content_by_lua_file)) support loading Lua 5.1 and LuaJIT 2.0/2.1 raw bytecode files directly.

Please note that the bytecode format used by LuaJIT 2.0/2.1 is not compatible with that used by the standard Lua 5.1 interpreter. So if using LuaJIT 2.0/2.1 with ngx_lua, LuaJIT compatible bytecode files must be generated as shown:

```bash

 /path/to/luajit/bin/luajit -b /path/to/input_file.lua /path/to/output_file.luac
```

The `-bg` option can be used to include debug information in the LuaJIT bytecode file:

```bash

 /path/to/luajit/bin/luajit -bg /path/to/input_file.lua /path/to/output_file.luac
```

Please refer to the official LuaJIT documentation on the `-b` option for more details:

<http://luajit.org/running.html#opt_b>

Also, the bytecode files generated by LuaJIT 2.1 is *not* compatible with LuaJIT 2.0, and vice versa. The support for LuaJIT 2.1 bytecode was first added in ngx_lua v0.9.3.

Similarly, if using the standard Lua 5.1 interpreter with ngx_lua, Lua compatible bytecode files must be generated using the `luac` commandline utility as shown:

```bash

 luac -o /path/to/output_file.luac /path/to/input_file.lua
```

Unlike as with LuaJIT, debug information is included in standard Lua 5.1 bytecode files by default. This can be striped out by specifying the `-s` option as shown:

```bash

 luac -s -o /path/to/output_file.luac /path/to/input_file.lua
```

Attempts to load standard Lua 5.1 bytecode files into ngx_lua instances linked to LuaJIT 2.0/2.1 or vice versa, will result in an error message, such as that below, being logged into the Nginx `error.log` file:


    [error] 13909#0: *1 failed to load Lua inlined code: bad byte-code header in /path/to/test_file.luac


Loading bytecode files via the Lua primitives like `require` and `dofile` should always work as expected.

[回到目录](#table-of-contents)

System Environment Variable Support
===================================

If you want to access the system environment variable, say, `foo`, in Lua via the standard Lua API [os.getenv](http://www.lua.org/manual/5.1/manual.html#pdf-os.getenv), then you should also list this environment variable name in your `nginx.conf` file via the [env directive](http://nginx.org/en/docs/ngx_core_module.html#env). For example,

```nginx

 env foo;
```

[回到目录](#table-of-contents)

HTTP 1.0 Support
================

HTTP 1.0协议不支持chunked编码输出，为了支持keep-alive特性，当响应包体不为空时，要求带有`Content-Length`头部。

所以，当发起一个HTTP 1.0协议的请求，并且开启了[lua_http10_buffering](#lua_http10_buffering)指令，ngx_lua会缓存[ngx.say](#ngxsay)和[ngx.print](#ngxprint)的输出，并且会推迟发送响应头，直到收到了所有的响应体输出。

彼时，ngx_lua便可以计算出整个包体的长度，并创建正确的`Content-Length`头部返回给HTTP 1.0客户端。如果`Content-Length`响应头在运行的Lua代码中设置，缓存功能会禁用，即使[lua_http10_buffering](#lua_http10_buffering)指令是开启状态。

对于大的流输出响应体，关闭[lua_http10_buffering](#lua_http10_buffering)指令以减少内存消耗至关重要。

注意，通常诸如`ab`和`http_load`等HTTP的基准测试工具默认发起的都是HTTP 1.0的请求。

要用`curl`发起HTTP 1.0请求，需要使用`-0`选项。
The HTTP 1.0 protocol does not support chunked output and requires an explicit `Content-Length` header when the response body is not empty in order to support the HTTP 1.0 keep-alive.
So when a HTTP 1.0 request is made and the [lua_http10_buffering](#lua_http10_buffering) directive is turned `on`, ngx_lua will buffer the
output of [ngx.say](#ngxsay) and [ngx.print](#ngxprint) calls and also postpone sending response headers until all the response body output is received.
At that time ngx_lua can calculate the total length of the body and construct a proper `Content-Length` header to return to the HTTP 1.0 client.
If the `Content-Length` response header is set in the running Lua code, however, this buffering will be disabled even if the [lua_http10_buffering](#lua_http10_buffering) directive is turned `on`.

For large streaming output responses, it is important to disable the [lua_http10_buffering](#lua_http10_buffering) directive to minimise memory usage.

Note that common HTTP benchmark tools such as `ab` and `http_load` issue HTTP 1.0 requests by default.
To force `curl` to send HTTP 1.0 requests, use the `-0` option.

[回到目录](#table-of-contents)

Statically Linking Pure Lua Modules
===================================

When LuaJIT 2.x is used, it is possible to statically link the bytecode of pure Lua modules into the Nginx executable.

Basically you use the `luajit` executable to compile `.lua` Lua module files to `.o` object files containing the exported bytecode data, and then link the `.o` files directly in your Nginx build.

Below is a trivial example to demonstrate this. Consider that we have the following `.lua` file named `foo.lua`:

```lua

 -- foo.lua
 local _M = {}

 function _M.go()
     print("Hello from foo")
 end

 return _M
```

And then we compile this `.lua` file to `foo.o` file:

    /path/to/luajit/bin/luajit -bg foo.lua foo.o

What matters here is the name of the `.lua` file, which determines how you use this module later on the Lua land. The file name `foo.o` does not matter at all except the `.o` file extension (which tells `luajit` what output format is used). If you want to strip the Lua debug information from the resulting bytecode, you can just specify the `-b` option above instead of `-bg`.

Then when building Nginx or OpenResty, pass the `--with-ld-opt="foo.o"` option to the `./configure` script:

```bash

 ./configure --with-ld-opt="/path/to/foo.o" ...
```

Finally, you can just do the following in any Lua code run by ngx_lua:

```lua

 local foo = require "foo"
 foo.go()
```

And this piece of code no longer depends on the external `foo.lua` file any more because it has already been compiled into the `nginx` executable.

If you want to use dot in the Lua module name when calling `require`, as in

```lua

 local foo = require "resty.foo"
```

then you need to rename the `foo.lua` file to `resty_foo.lua` before compiling it down to a `.o` file with the `luajit` command-line utility.

It is important to use exactly the same version of LuaJIT when compiling `.lua` files to `.o` files as building nginx + ngx_lua. This is because the LuaJIT bytecode format may be incompatible between different LuaJIT versions. When the bytecode format is incompatible, you will see a Lua runtime error saying that the Lua module is not found.

When you have multiple `.lua` files to compile and link, then just specify their `.o` files at the same time in the value of the `--with-ld-opt` option. For instance,

```bash

 ./configure --with-ld-opt="/path/to/foo.o /path/to/bar.o" ...
```

If you have just too many `.o` files, then it might not be feasible to name them all in a single command. In this case, you can build a static library (or archive) for your `.o` files, as in

```bash

 ar rcus libmyluafiles.a *.o
```

then you can link the `myluafiles` archive as a whole to your nginx executable:

```bash

 ./configure \
     --with-ld-opt="-L/path/to/lib -Wl,--whole-archive -lmyluafiles -Wl,--no-whole-archive"
```

where `/path/to/lib` is the path of the directory containing the `libmyluafiles.a` file. It should be noted that the linker option `--whole-archive` is required here because otherwise our archive will be skipped because no symbols in our archive are mentioned in the main parts of the nginx executable.

[回到目录](#table-of-contents)

Data Sharing within an Nginx Worker
===================================

To globally share data among all the requests handled by the same nginx worker process, encapsulate the shared data into a Lua module, use the Lua `require` builtin to import the module, and then manipulate the shared data in Lua. This works because required Lua modules are loaded only once and all coroutines will share the same copy of the module (both its code and data). Note however that Lua global variables (note, not module-level variables) WILL NOT persist between requests because of the one-coroutine-per-request isolation design.

Here is a complete small example:

```lua

 -- mydata.lua
 local _M = {}

 local data = {
     dog = 3,
     cat = 4,
     pig = 5,
 }

 function _M.get_age(name)
     return data[name]
 end

 return _M
```

and then accessing it from `nginx.conf`:

```nginx

 location /lua {
     content_by_lua '
         local mydata = require "mydata"
         ngx.say(mydata.get_age("dog"))
     ';
 }
```

The `mydata` module in this example will only be loaded and run on the first request to the location `/lua`,
and all subsequent requests to the same nginx worker process will use the reloaded instance of the
module as well as the same copy of the data in it, until a `HUP` signal is sent to the Nginx master process to force a reload.
This data sharing technique is essential for high performance Lua applications based on this module.

Note that this data sharing is on a *per-worker* basis and not on a *per-server* basis. That is, when there are multiple nginx worker processes under an Nginx master, data sharing cannot cross the process boundary between these workers.

It is usually recommended to share read-only data this way. You can also share changeable data among all the concurrent requests of each nginx worker process as
long as there is *no* nonblocking I/O operations (including [ngx.sleep](#ngxsleep))
in the middle of your calculations. As long as you do not give the
control back to the nginx event loop and ngx_lua's light thread
scheduler (even implicitly), there can never be any race conditions in
between. For this reason, always be very careful when you want to share changeable data on the
worker level. Buggy optimizations can easily lead to hard-to-debug
race conditions under load.

If server-wide data sharing is required, then use one or more of the following approaches:

1. Use the [ngx.shared.DICT](#ngxshareddict) API provided by this module.
1. Use only a single nginx worker and a single server (this is however not recommended when there is a multi core CPU or multiple CPUs in a single machine).
1. Use data storage mechanisms such as `memcached`, `redis`, `MySQL` or `PostgreSQL`. [The ngx_openresty bundle](http://openresty.org) associated with this module comes with a set of companion Nginx modules and Lua libraries that provide interfaces with these data storage mechanisms.

[回到目录](#table-of-contents)

Known Issues
============

[回到目录](#table-of-contents)

TCP socket connect operation issues
-----------------------------------
The [tcpsock:connect](#tcpsockconnect) method may indicate `success` despite connection failures such as with `Connection Refused` errors. 

However, later attempts to manipulate the cosocket object will fail and return the actual error status message generated by the failed connect operation. 

This issue is due to limitations in the Nginx event model and only appears to affect Mac OS X.

[回到目录](#table-of-contents)

Lua Coroutine Yielding/Resuming
-------------------------------
* Because Lua's `dofile` and `require` builtins are currently implemented as C functions in both Lua 5.1 and LuaJIT 2.0/2.1, if the Lua file being loaded by `dofile` or `require` invokes [ngx.location.capture*](#ngxlocationcapture), [ngx.exec](#ngxexec), [ngx.exit](#ngxexit), or other API functions requiring yielding in the *top-level* scope of the Lua file, then the Lua error "attempt to yield across C-call boundary" will be raised. To avoid this, put these calls requiring yielding into your own Lua functions in the Lua file instead of the top-level scope of the file.
* As the standard Lua 5.1 interpreter's VM is not fully resumable, the methods [ngx.location.capture](#ngxlocationcapture), [ngx.location.capture_multi](#ngxlocationcapture_multi), [ngx.redirect](#ngxredirect), [ngx.exec](#ngxexec), and [ngx.exit](#ngxexit) cannot be used within the context of a Lua [pcall()](http://www.lua.org/manual/5.1/manual.html#pdf-pcall) or [xpcall()](http://www.lua.org/manual/5.1/manual.html#pdf-xpcall) or even the first line of the `for ... in ...` statement when the standard Lua 5.1 interpreter is used and the `attempt to yield across metamethod/C-call boundary` error will be produced. Please use LuaJIT 2.x, which supports a fully resumable VM, to avoid this.

[回到目录](#table-of-contents)

Lua Variable Scope
------------------
注意，在导入模块时，应该使用以下形式：

```lua

 local xxx = require('xxx')
```

而不是这种形式：

```lua

 require('xxx')
```

原因如下：在设计上，全局变量的生命周期与请求的处理函数相同。每个请求处理函数拥有自身的Lua全局变量，这就是所谓的请求独立思想。通常，Lua模块会被首个Nginx请求处理函数加载，并通过`package.loaded`的内建指令`require()`缓存，用于后续使用。一些Lua模块使用的`module()`内建指令有设置全局变量给加载的模块table的副作用。但是该全局变量会在请求处理结束时被清空，每个请求的处理函数都拥有各自（干净）的全局环境。所以访问`nil`值会抛出Lua异常。

通常来说，在ngx_lua的上下文使用Lua全局变量是很不好的做法，因为：

1. 对于并发请求，当仅可以使用局部变量而误用了全局变量时，会引入严重的副作用；
2. Lua全局变量会在全局环境中查找Lua table，这通常代价较大；
3. 一些全局变量引用通常只是typos，这很难调试。

强烈建议尽可能仅适用“local”变量。

想知道你的Lua代码中使用的所有全局变量，可以在你的lua代码中运行[lua-releng tool](https://github.com/openresty/nginx-devel-utils/blob/master/lua-releng)：

    $ lua-releng
    Checking use of Lua global variables in file lib/foo/bar.lua ...
            1       [1489]  SETGLOBAL       7 -1    ; contains
            55      [1506]  GETGLOBAL       7 -3    ; setvar
            3       [1545]  GETGLOBAL       3 -4    ; varexpand

输出显示文件`lib/foo/bar.lua`的第1489行对全局变量`contains`执行了写入操作，第1506行对全局变量`setvar`执行了读取操作，第1545行对全局变量`varexpand`执行了读取操作。

该工具可以保证在Lua模块函数中的局部变量都是用`local`定义，否则会抛出运行时异常。在访问该变量时，它会避免不必要的静态条件。原因见[Data Sharing within an Nginx Worker](#data-sharing-within-an-nginx-worker)。

[回到目录](#table-of-contents)

Locations Configured by Subrequest Directives of Other Modules
--------------------------------------------------------------
[ngx.location.capture](#ngxlocationcapture)和[ngx.location.capture_multi](#ngxlocationcapture_multi)指令不能捕获包含以下指令的location：[add_before_body](http://nginx.org/en/docs/http/ngx_http_addition_module.html#add_before_body), [add_after_body](http://nginx.org/en/docs/http/ngx_http_addition_module.html#add_after_body), [auth_request](http://nginx.org/en/docs/http/ngx_http_auth_request_module.html#auth_request), [echo_location](http://github.com/openresty/echo-nginx-module#echo_location), [echo_location_async](http://github.com/openresty/echo-nginx-module#echo_location_async), [echo_subrequest](http://github.com/openresty/echo-nginx-module#echo_subrequest)和[echo_subrequest_async](http://github.com/openresty/echo-nginx-module#echo_subrequest_async)。例如，

```nginx

 location /foo {
     content_by_lua '
         res = ngx.location.capture("/bar")
     ';
 }
 location /bar {
     echo_location /blah;
 }
 location /blah {
     echo "Success!";
 }
```

```nginx

 $ curl -i http://example.com/foo
```

以上例子不能如期工作。

[回到目录](#table-of-contents)

Cosockets Not Available Everywhere
----------------------------------

由于Nginx内核的限制，在以下这些上下文中，cosocket API是禁用的：[set_by_lua*](#set_by_lua), [log_by_lua*](#log_by_lua), [header_filter_by_lua*](#header_filter_by_lua)和[body_filter_by_lua](#body_filter_by_lua)。

目前，cosocket在[init_by_lua*](#init_by_lua)和[init_worker_by_lua*](#init_worker_by_lua)指令上下文也是禁用的，但以后我们可能支持该上下文，因为在Nginx内核对此没有限制（或者可以绕过限制）。

尽管如此，如果原始上下文不需要等待cosocket的返回结果，那么存在一种绕道而行的办法。使用ngx.timer.at](#ngxtimerat)创建一个0延时计时器，在计时器回调中异步运行计算cosckit结果。

[回到目录](#table-of-contents)

Special Escaping Sequences
--------------------------
你需要对诸如`\d`，`\s`，`\w`的PCRE序列特别关注，因为反斜杠`\`在被Lua语言解析器和Nginx配置文件解析器处理之前都会被剔除掉。所以，以下代码片段不能如期工作：

```nginx

 # nginx.conf
 ? location /test {
 ?     content_by_lua '
 ?         local regex = "\d+"  -- 这是错的!!
 ?         local m = ngx.re.match("hello, 1234", regex)
 ?         if m then ngx.say(m[0]) else ngx.say("not matched!") end
 ?     ';
 ? }
 # 结果输出"not matched!"
```

解决办法是，两次转义反斜杠：

```nginx

 # nginx.conf
 location /test {
     content_by_lua '
         local regex = "\\\\d+"
         local m = ngx.re.match("hello, 1234", regex)
         if m then ngx.say(m[0]) else ngx.say("not matched!") end
     ';
 }
 # 结果输出"1234"
```

这里，`\\\\d+`被Nginx配置文件解析器解析成`\\d+`，然后被Lua语言解析器进一步解析成`\d+`，之后运行。

另外，正则模式串可以表示成long-bracketed Lua字符串的形式（`[[...]]`），这种情况下，只需要为Nginx配置文件parser一次转义。

```nginx

 # nginx.conf
 location /test {
     content_by_lua '
         local regex = [[\\d+]]
         local m = ngx.re.match("hello, 1234", regex)
         if m then ngx.say(m[0]) else ngx.say("not matched!") end
     ';
 }
 # 结果输出"1234"
```

此时，`[[\\d+]]`被Nginx配置文件解析器解析成`[[\d+]]`。

注意，如果正则匹配串中包含`[...]`序列，那么要求使用更长的括号形式`[=[...]=]`。如果需要，`[=[...]=]`可以用作默认形式。

```nginx

 # nginx.conf
 location /test {
     content_by_lua '
         local regex = [=[[0-9]+]=]
         local m = ngx.re.match("hello, 1234", regex)
         if m then ngx.say(m[0]) else ngx.say("not matched!") end
     ';
 }
 # 结果输出"1234"
```

还有一种转义PCRE序列的方法是将Lua代码放在外部脚本文件中，并使用`*_by_lua_file`指令。该方法反斜杠只需要为Lua语言解析器转义一次。

```lua

 -- test.lua
 local regex = "\\d+"
 local m = ngx.re.match("hello, 1234", regex)
 if m then ngx.say(m[0]) else ngx.say("not matched!") end
 -- 结果输出"1234"
```

在外部脚本文件中，long-bracketed形式的PCRE序列不需要改动。
 
```lua

 -- test.lua
 local regex = [[\d+]]
 local m = ngx.re.match("hello, 1234", regex)
 if m then ngx.say(m[0]) else ngx.say("not matched!") end
 -- 结果输出"1234"
```

[回到目录](#table-of-contents)

Mixing with SSI Not Supported
-----------------------------

Mixing SSI with ngx_lua in the same Nginx request is not supported at all. Just use ngx_lua exclusively. Everything you can do with SSI can be done atop ngx_lua anyway and it can be more efficient when using ngx_lua.

[回到目录](#table-of-contents)

SPDY Mode Not Fully Supported
-----------------------------

Certain Lua APIs provided by ngx_lua do not work in Nginx's SPDY mode yet: [ngx.location.capture](#ngxlocationcapture), [ngx.location.capture_multi](#ngxlocationcapture_multi), and [ngx.req.socket](#ngxreqsocket).

[回到目录](#table-of-contents)

Missing data on short circuited requests
----------------------------------------

以下状态码的情况，Nginx会提前结束请求：

* 400 (Bad Request)
* 405 (Not Allowed)
* 408 (Request Timeout)
* 414 (Request URI Too Large)
* 494 (Request Headers Too Large)
* 499 (Client Closed Request)
* 500 (Internal Server Error)
* 501 (Not Implemented)

这意味着，正常运行的阶段会略过，如rewrite或access阶段。也意味着，后续的阶段如[log_by_lua](#log_by_lua)不会访问到之前阶段正常设置的信息。

[回到目录](#table-of-contents)

TODO
====

* add `*_by_lua_block` directives for existing `*_by_lua` directives so that we put literal Lua code directly in curly braces instead of an nginx literal string. For example,
```nginx

 content_by_lua_block {
     ngx.say("hello, world\r\n")
 }
```
	which is equivalent to
```nginx

 content_by_lua '
     ngx.say("hello, world\\r\\n")
 ';
```
	but the former is much cleaner and nicer.
* cosocket: implement LuaSocket's unconnected UDP API.
* add support for implementing general TCP servers instead of HTTP servers in Lua. For example,
```lua

 tcp {
     server {
         listen 11212;
         handler_by_lua '
             -- custom Lua code implementing the special TCP server...
         ';
     }
 }
```
* add support for implementing general UDP servers instead of HTTP servers in Lua. For example,
```lua

 udp {
     server {
         listen 1953;
         handler_by_lua '
             -- custom Lua code implementing the special UDP server...
         ';
     }
 }
```
* ssl: implement directives `ssl_certificate_by_lua` and `ssl_certificate_by_lua_file` to allow using Lua to dynamically serve SSL certificates and keys for downstream SSL handshake. (already done in CloudFlare's private branch and powering CloudFlare's SSL gateway of its global network. expected to be opensourced in March 2015.)
* shm: implement a "shared queue API" to complement the existing [shared dict](#lua_shared_dict) API.
* cosocket: add support in the context of [init_by_lua*](#init_by_lua).
* cosocket: implement the `bind()` method for stream-typed cosockets.
* cosocket: pool-based backend concurrency level control: implement automatic `connect` queueing when the backend concurrency exceeds its connection pool limit.
* cosocket: review and merge aviramc's [patch](https://github.com/openresty/lua-nginx-module/pull/290) for adding the `bsdrecv` method.
* add new API function `ngx.resp.add_header` to emulate the standard `add_header` config directive.
* [ngx.re](#ngxrematch) API: use `false` instead of `nil` in the resulting match table to indicate non-existent submatch captures, such that we can avoid "holes" in the array table.
* review and apply Jader H. Silva's patch for `ngx.re.split()`.
* review and apply vadim-pavlov's patch for [ngx.location.capture](#ngxlocationcapture)'s `extra_headers` option
* use `ngx_hash_t` to optimize the built-in header look-up process for [ngx.req.set_header](#ngxreqset_header), [ngx.header.HEADER](#ngxheaderheader), and etc.
* add configure options for different strategies of handling the cosocket connection exceeding in the pools.
* add directives to run Lua codes when nginx stops.
* add `ignore_resp_headers`, `ignore_resp_body`, and `ignore_resp` options to [ngx.location.capture](#ngxlocationcapture) and [ngx.location.capture_multi](#ngxlocationcapture_multi) methods, to allow micro performance tuning on the user side.
* add automatic Lua code time slicing support by yielding and resuming the Lua VM actively via Lua's debug hooks.
* add `stat` mode similar to [mod_lua](https://httpd.apache.org/docs/trunk/mod/mod_lua.html).

[回到目录](#table-of-contents)

Changes
=======

The changes of every release of this module can be obtained from the ngx_openresty bundle's change logs:

<http://openresty.org/#Changes>

[回到目录](#table-of-contents)

Test Suite
==========

The following dependencies are required to run the test suite:

* Nginx version >= 1.4.2

* Perl modules:
	* Test::Nginx: <https://github.com/openresty/test-nginx>

* Nginx modules:
	* [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit)
	* [ngx_set_misc](https://github.com/openresty/set-misc-nginx-module)
	* [ngx_auth_request](http://mdounin.ru/files/ngx_http_auth_request_module-0.2.tar.gz) (this is not needed if you're using Nginx 1.5.4+.
	* [ngx_echo](https://github.com/openresty/echo-nginx-module)
	* [ngx_memc](https://github.com/openresty/memc-nginx-module)
	* [ngx_srcache](https://github.com/openresty/srcache-nginx-module)
	* ngx_lua (i.e., this module)
	* [ngx_lua_upstream](https://github.com/openresty/lua-upstream-nginx-module)
	* [ngx_headers_more](https://github.com/openresty/headers-more-nginx-module)
	* [ngx_drizzle](https://github.com/openresty/drizzle-nginx-module)
	* [ngx_rds_json](https://github.com/openresty/rds-json-nginx-module)
	* [ngx_coolkit](https://github.com/FRiCKLE/ngx_coolkit)
	* [ngx_redis2](https://github.com/openresty/redis2-nginx-module)

The order in which these modules are added during configuration is important because the position of any filter module in the
filtering chain determines the final output, for example. The correct adding order is shown above.

* 3rd-party Lua libraries:
	* [lua-cjson](http://www.kyne.com.au/~mark/software/lua-cjson.php)

* Applications:
	* mysql: create database 'ngx_test', grant all privileges to user 'ngx_test', password is 'ngx_test'
	* memcached: listening on the default port, 11211.
	* redis: listening on the default port, 6379.

See also the [developer build script](https://github.com/openresty/lua-nginx-module/blob/master/util/build2.sh) for more details on setting up the testing environment.

To run the whole test suite in the default testing mode:

    cd /path/to/lua-nginx-module
    export PATH=/path/to/your/nginx/sbin:$PATH
    prove -I/path/to/test-nginx/lib -r t


To run specific test files:

    cd /path/to/lua-nginx-module
    export PATH=/path/to/your/nginx/sbin:$PATH
    prove -I/path/to/test-nginx/lib t/002-content.t t/003-errors.t


To run a specific test block in a particular test file, add the line `--- ONLY` to the test block you want to run, and then use the `prove` utility to run that `.t` file.

There are also various testing modes based on mockeagain, valgrind, and etc. Refer to the [Test::Nginx documentation](http://search.cpan.org/perldoc?Test::Nginx) for more details for various advanced testing modes. See also the test reports for the Nginx test cluster running on Amazon EC2: <http://qa.openresty.org.>

[回到目录](#table-of-contents)

Copyright and License
=====================

This module is licensed under the BSD license.

Copyright (C) 2009-2015, by Xiaozhe Wang (chaoslawful) <chaoslawful@gmail.com>.

Copyright (C) 2009-2015, by Yichun "agentzh" Zhang (章亦春) <agentzh@gmail.com>, CloudFlare Inc.

All rights reserved.

Redistribution and use in source and binary forms, with or without modification, are permitted provided that the following conditions are met:

* Redistributions of source code must retain the above copyright notice, this list of conditions and the following disclaimer.

* Redistributions in binary form must reproduce the above copyright notice, this list of conditions and the following disclaimer in the documentation and/or other materials provided with the distribution.

THIS SOFTWARE IS PROVIDED BY THE COPYRIGHT HOLDERS AND CONTRIBUTORS "AS IS" AND ANY EXPRESS OR IMPLIED WARRANTIES, INCLUDING, BUT NOT LIMITED TO, THE IMPLIED WARRANTIES OF MERCHANTABILITY AND FITNESS FOR A PARTICULAR PURPOSE ARE DISCLAIMED. IN NO EVENT SHALL THE COPYRIGHT HOLDER OR CONTRIBUTORS BE LIABLE FOR ANY DIRECT, INDIRECT, INCIDENTAL, SPECIAL, EXEMPLARY, OR CONSEQUENTIAL DAMAGES (INCLUDING, BUT NOT LIMITED TO, PROCUREMENT OF SUBSTITUTE GOODS OR SERVICES; LOSS OF USE, DATA, OR PROFITS; OR BUSINESS INTERRUPTION) HOWEVER CAUSED AND ON ANY THEORY OF LIABILITY, WHETHER IN CONTRACT, STRICT LIABILITY, OR TORT (INCLUDING NEGLIGENCE OR OTHERWISE) ARISING IN ANY WAY OUT OF THE USE OF THIS SOFTWARE, EVEN IF ADVISED OF THE POSSIBILITY OF SUCH DAMAGE.

[回到目录](#table-of-contents)

See Also
========

* [lua-resty-memcached](https://github.com/openresty/lua-resty-memcached) library based on ngx_lua cosocket.
* [lua-resty-redis](https://github.com/openresty/lua-resty-redis) library based on ngx_lua cosocket.
* [lua-resty-mysql](https://github.com/openresty/lua-resty-mysql) library based on ngx_lua cosocket.
* [lua-resty-upload](https://github.com/openresty/lua-resty-upload) library based on ngx_lua cosocket.
* [lua-resty-dns](https://github.com/openresty/lua-resty-dns) library based on ngx_lua cosocket.
* [lua-resty-websocket](https://github.com/openresty/lua-resty-websocket) library for both WebSocket server and client, based on ngx_lua cosocket.
* [lua-resty-string](https://github.com/openresty/lua-resty-string) library based on [LuaJIT FFI](http://luajit.org/ext_ffi.html).
* [lua-resty-lock](https://github.com/openresty/lua-resty-lock) library for a nonblocking simple lock API.
* [lua-resty-cookie](https://github.com/cloudflare/lua-resty-cookie) library for HTTP cookie manipulation.
* [Routing requests to different MySQL queries based on URI arguments](http://openresty.org/#RoutingMySQLQueriesBasedOnURIArgs)
* [Dynamic Routing Based on Redis and Lua](http://openresty.org/#DynamicRoutingBasedOnRedis)
* [Using LuaRocks with ngx_lua](http://openresty.org/#UsingLuaRocks)
* [Introduction to ngx_lua](https://github.com/openresty/lua-nginx-module/wiki/Introduction)
* [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit)
* [echo-nginx-module](http://github.com/openresty/echo-nginx-module)
* [drizzle-nginx-module](http://github.com/openresty/drizzle-nginx-module)
* [postgres-nginx-module](https://github.com/FRiCKLE/ngx_postgres)
* [memc-nginx-module](http://github.com/openresty/memc-nginx-module)
* [The ngx_openresty bundle](http://openresty.org)
* [Nginx Systemtap Toolkit](https://github.com/openresty/nginx-systemtap-toolkit)

[回到目录](#table-of-contents)

Directives
==========

* [lua_use_default_type](#lua_use_default_type)
* [lua_code_cache](#lua_code_cache)
* [lua_regex_cache_max_entries](#lua_regex_cache_max_entries)
* [lua_regex_match_limit](#lua_regex_match_limit)
* [lua_package_path](#lua_package_path)
* [lua_package_cpath](#lua_package_cpath)
* [init_by_lua](#init_by_lua)
* [init_by_lua_file](#init_by_lua_file)
* [init_worker_by_lua](#init_worker_by_lua)
* [init_worker_by_lua_file](#init_worker_by_lua_file)
* [set_by_lua](#set_by_lua)
* [set_by_lua_file](#set_by_lua_file)
* [content_by_lua](#content_by_lua)
* [content_by_lua_file](#content_by_lua_file)
* [rewrite_by_lua](#rewrite_by_lua)
* [rewrite_by_lua_file](#rewrite_by_lua_file)
* [access_by_lua](#access_by_lua)
* [access_by_lua_file](#access_by_lua_file)
* [header_filter_by_lua](#header_filter_by_lua)
* [header_filter_by_lua_file](#header_filter_by_lua_file)
* [body_filter_by_lua](#body_filter_by_lua)
* [body_filter_by_lua_file](#body_filter_by_lua_file)
* [log_by_lua](#log_by_lua)
* [log_by_lua_file](#log_by_lua_file)
* [lua_need_request_body](#lua_need_request_body)
* [lua_shared_dict](#lua_shared_dict)
* [lua_socket_connect_timeout](#lua_socket_connect_timeout)
* [lua_socket_send_timeout](#lua_socket_send_timeout)
* [lua_socket_send_lowat](#lua_socket_send_lowat)
* [lua_socket_read_timeout](#lua_socket_read_timeout)
* [lua_socket_buffer_size](#lua_socket_buffer_size)
* [lua_socket_pool_size](#lua_socket_pool_size)
* [lua_socket_keepalive_timeout](#lua_socket_keepalive_timeout)
* [lua_socket_log_errors](#lua_socket_log_errors)
* [lua_ssl_ciphers](#lua_ssl_ciphers)
* [lua_ssl_crl](#lua_ssl_crl)
* [lua_ssl_protocols](#lua_ssl_protocols)
* [lua_ssl_trusted_certificate](#lua_ssl_trusted_certificate)
* [lua_ssl_verify_depth](#lua_ssl_verify_depth)
* [lua_http10_buffering](#lua_http10_buffering)
* [rewrite_by_lua_no_postpone](#rewrite_by_lua_no_postpone)
* [lua_transform_underscores_in_response_headers](#lua_transform_underscores_in_response_headers)
* [lua_check_client_abort](#lua_check_client_abort)
* [lua_max_pending_timers](#lua_max_pending_timers)
* [lua_max_running_timers](#lua_max_running_timers)


[回到目录](#table-of-contents)

lua_use_default_type
--------------------
**syntax:** *lua_use_default_type on | off*

**default:** *lua_use_default_type on*

**context:** *http, server, location, location if*

Specifies whether to use the MIME type specified by the [default_type](http://nginx.org/en/docs/http/ngx_http_core_module.html#default_type) directive for the default value of the `Content-Type` response header. If you do not want a default `Content-Type` response header for your Lua request handlers, then turn this directive off.

This directive is turned on by default.

This directive was first introduced in the `v0.9.1` release.

[回到目录](#directives)

lua_code_cache
--------------
**syntax:** *lua_code_cache on | off*

**default:** *lua_code_cache on*

**context:** *http, server, location, location if*

Enables or disables the Lua code cache for Lua code in `*_by_lua_file` directives (like [set_by_lua_file](#set_by_lua_file) and
[content_by_lua_file](#content_by_lua_file)) and Lua modules.

When turning off, every request served by ngx_lua will run in a separate Lua VM instance, starting from the `0.9.3` release. So the Lua files referenced in [set_by_lua_file](#set_by_lua_file),
[content_by_lua_file](#content_by_lua_file), [access_by_lua_file](#access_by_lua_file),
and etc will not be cached
and all Lua modules used will be loaded from scratch. With this in place, developers can adopt an edit-and-refresh approach.

Please note however, that Lua code written inlined within nginx.conf
such as those specified by [set_by_lua](#set_by_lua), [content_by_lua](#content_by_lua),
[access_by_lua](#access_by_lua), and [rewrite_by_lua](#rewrite_by_lua) will not be updated when you edit the inlined Lua code in your `nginx.conf` file because only the Nginx config file parser can correctly parse the `nginx.conf`
file and the only way is to reload the config file
by sending a `HUP` signal or just to restart Nginx.

Even when the code cache is enabled, Lua files which are loaded by `dofile` or `loadfile`
in *_by_lua_file cannot be cached (unless you cache the results yourself). Usually you can either use the [init_by_lua](#init_by_lua)
or [init_by_lua_file](#init-by_lua_file) directives to load all such files or just make these Lua files true Lua modules
and load them via `require`.

The ngx_lua module does not support the `stat` mode available with the
Apache `mod_lua` module (yet).

Disabling the Lua code cache is strongly
discouraged for production use and should only be used during 
development as it has a significant negative impact on overall performance. For example, the performance a "hello world" Lua example can drop by an order of magnitude after disabling the Lua code cache.

[回到目录](#directives)

lua_regex_cache_max_entries
---------------------------
**syntax:** *lua_regex_cache_max_entries &lt;num&gt;*

**default:** *lua_regex_cache_max_entries 1024*

**context:** *http*

Specifies the maximum number of entries allowed in the worker process level compiled regex cache.

The regular expressions used in [ngx.re.match](#ngxrematch), [ngx.re.gmatch](#ngxregmatch), [ngx.re.sub](#ngxresub), and [ngx.re.gsub](#ngxregsub) will be cached within this cache if the regex option `o` (i.e., compile-once flag) is specified.

The default number of entries allowed is 1024 and when this limit is reached, new regular expressions will not be cached (as if the `o` option was not specified) and there will be one, and only one, warning in the `error.log` file:


    2011/08/27 23:18:26 [warn] 31997#0: *1 lua exceeding regex cache max entries (1024), ...


Do not activate the `o` option for regular expressions (and/or `replace` string arguments for [ngx.re.sub](#ngxresub) and [ngx.re.gsub](#ngxregsub)) that are generated *on the fly* and give rise to infinite variations to avoid hitting the specified limit.

[回到目录](#directives)

lua_regex_match_limit
---------------------
**syntax:** *lua_regex_match_limit &lt;num&gt;*

**default:** *lua_regex_match_limit 0*

**context:** *http*

Specifies the "match limit" used by the PCRE library when executing the [ngx.re API](#ngxrematch). To quote the PCRE manpage, "the limit ... has the effect of limiting the amount of backtracking that can take place."

When the limit is hit, the error string "pcre_exec() failed: -8" will be returned by the [ngx.re API](#ngxrematch) functions on the Lua land.

When setting the limit to 0, the default "match limit" when compiling the PCRE library is used. And this is the default value of this directive.

This directive was first introduced in the `v0.8.5` release.

[回到目录](#directives)

lua_package_path
----------------

**syntax:** *lua_package_path &lt;lua-style-path-str&gt;*

**default:** *The content of LUA_PATH environ variable or Lua's compiled-in defaults.*

**context:** *http*

Sets the Lua module search path used by scripts specified by [set_by_lua](#set_by_lua),
[content_by_lua](#content_by_lua) and others. The path string is in standard Lua path form, and `;;`
can be used to stand for the original search paths.

As from the `v0.5.0rc29` release, the special notation `$prefix` or `${prefix}` can be used in the search path string to indicate the path of the `server prefix` usually determined by the `-p PATH` command-line option while starting the Nginx server.

[回到目录](#directives)

lua_package_cpath
-----------------

**syntax:** *lua_package_cpath &lt;lua-style-cpath-str&gt;*

**default:** *The content of LUA_CPATH environment variable or Lua's compiled-in defaults.*

**context:** *http*

Sets the Lua C-module search path used by scripts specified by [set_by_lua](#set_by_lua),
[content_by_lua](#content_by_lua) and others. The cpath string is in standard Lua cpath form, and `;;`
can be used to stand for the original cpath.

As from the `v0.5.0rc29` release, the special notation `$prefix` or `${prefix}` can be used in the search path string to indicate the path of the `server prefix` usually determined by the `-p PATH` command-line option while starting the Nginx server.

[回到目录](#directives)

init_by_lua
-----------

**syntax:** *init_by_lua &lt;lua-script-str&gt;*

**context:** *http*

**phase:** *loading-config*

Runs the Lua code specified by the argument `<lua-script-str>` on the global Lua VM level when the Nginx master process (if any) is loading the Nginx config file.

When Nginx receives the `HUP` signal and starts reloading the config file, the Lua VM will also be re-created and `init_by_lua` will run again on the new Lua VM. In case that the [lua_code_cache](#lua_code_cache) directive is turned off (default on), the `init_by_lua` handler will run upon every request because in this special mode a standalone Lua VM is always created for each request.

Usually you can register (true) Lua global variables or pre-load Lua modules at server start-up by means of this hook. Here is an example for pre-loading Lua modules:

```nginx

 init_by_lua 'cjson = require "cjson"';

 server {
     location = /api {
         content_by_lua '
             ngx.say(cjson.encode({dog = 5, cat = 6}))
         ';
     }
 }
```

You can also initialize the [lua_shared_dict](#lua_shared_dict) shm storage at this phase. Here is an example for this:

```nginx

 lua_shared_dict dogs 1m;

 init_by_lua '
     local dogs = ngx.shared.dogs;
     dogs:set("Tom", 56)
 ';

 server {
     location = /api {
         content_by_lua '
             local dogs = ngx.shared.dogs;
             ngx.say(dogs:get("Tom"))
         ';
     }
 }
```

But note that, the [lua_shared_dict](#lua_shared_dict)'s shm storage will not be cleared through a config reload (via the `HUP` signal, for example). So if you do *not* want to re-initialize the shm storage in your `init_by_lua` code in this case, then you just need to set a custom flag in the shm storage and always check the flag in your `init_by_lua` code.

Because the Lua code in this context runs before Nginx forks its worker processes (if any), data or code loaded here will enjoy the [Copy-on-write (COW)](http://en.wikipedia.org/wiki/Copy-on-write) feature provided by many operating systems among all the worker processes, thus saving a lot of memory.

Do *not* initialize your own Lua global variables in this context because use of Lua global variables have performance penalties and can lead to global namespace pollution (see the [Lua Variable Scope](#lua-variable-scope) section for more details). The recommended way is to use proper [Lua module](http://www.lua.org/manual/5.1/manual.html#5.3) files (but do not use the standard Lua function [module()](http://www.lua.org/manual/5.1/manual.html#pdf-module) to define Lua modules because it pollutes the global namespace as well) and call [require()](http://www.lua.org/manual/5.1/manual.html#pdf-require) to load your own module files in `init_by_lua` or other contexts ([require()](http://www.lua.org/manual/5.1/manual.html#pdf-require) does cache the loaded Lua modules in the global `package.loaded` table in the Lua registry so your modules will only loaded once for the whole Lua VM instance).

Only a small set of the [Nginx API for Lua](#nginx-api-for-lua) is supported in this context:

* Logging APIs: [ngx.log](#ngxlog) and [print](#print),
* Shared Dictionary API: [ngx.shared.DICT](#ngxshareddict).

More Nginx APIs for Lua may be supported in this context upon future user requests.

Basically you can safely use Lua libraries that do blocking I/O in this very context because blocking the master process during server start-up is completely okay. Even the Nginx core does blocking I/O (at least on resolving upstream's host names) at the configure-loading phase.

You should be very careful about potential security vulnerabilities in your Lua code registered in this context because the Nginx master process is often run under the `root` account.

This directive was first introduced in the `v0.5.5` release.

[回到目录](#directives)

init_by_lua_file
----------------

**syntax:** *init_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *http*

**phase:** *loading-config*

Equivalent to [init_by_lua](#init_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code or [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

This directive was first introduced in the `v0.5.5` release.

[回到目录](#directives)

init_worker_by_lua
------------------

**syntax:** *init_worker_by_lua &lt;lua-script-str&gt;*

**context:** *http*

**phase:** *starting-worker*

Runs the specified Lua code upon every Nginx worker process's startup when the master process is enabled. When the master process is disabled, this hook will just run after [init_by_lua*](#init_by_lua).

This hook is often used to create per-worker reoccurring timers (via the [ngx.timer.at](#ngxtimerat) Lua API), either for backend healthcheck or other timed routine work. Below is an example,

```nginx

 init_worker_by_lua '
     local delay = 3  -- in seconds
     local new_timer = ngx.timer.at
     local log = ngx.log
     local ERR = ngx.ERR
     local check

     check = function(premature)
         if not premature then
             -- do the health check or other routine work
             local ok, err = new_timer(delay, check)
             if not ok then
                 log(ERR, "failed to create timer: ", err)
                 return
             end
         end
     end

     local ok, err = new_timer(delay, check)
     if not ok then
         log(ERR, "failed to create timer: ", err)
         return
     end
 ';
```

This directive was first introduced in the `v0.9.5` release.

[回到目录](#directives)

init_worker_by_lua_file
-----------------------

**syntax:** *init_worker_by_lua_file &lt;lua-file-path&gt;*

**context:** *http*

**phase:** *starting-worker*

Similar to [init_worker_by_lua](#init_worker_by_lua), but accepts the file path to a Lua source file or Lua bytecode file.

This directive was first introduced in the `v0.9.5` release.

[回到目录](#directives)

set_by_lua
----------

**syntax:** *set_by_lua $res &lt;lua-script-str&gt; [$arg1 $arg2 ...]*

**context:** *server, server if, location, location if*

**phase:** *rewrite*

Executes code specified in `<lua-script-str>` with optional input arguments `$arg1 $arg2 ...`, and returns string output to `$res`. 
The code in `<lua-script-str>` can make [API calls](#nginx-api-for-lua) and can retrieve input arguments from the `ngx.arg` table (index starts from `1` and increases sequentially).

This directive is designed to execute short, fast running code blocks as the Nginx event loop is blocked during code execution. Time consuming code sequences should therefore be avoided.

This directive is implemented by injecting custom commands into the standard [ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)'s command list. Because [ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html) does not support nonblocking I/O in its commands, Lua APIs requiring yielding the current Lua "light thread" cannot work in this directive.

At least the following API functions are currently disabled within the context of `set_by_lua`:

* Output API functions (e.g., [ngx.say](#ngxsay) and [ngx.send_headers](#ngxsend_headers))
* Control API functions (e.g., [ngx.exit](#ngxexit)) 
* Subrequest API functions (e.g., [ngx.location.capture](#ngxlocationcapture) and [ngx.location.capture_multi](#ngxlocationcapture_multi))
* Cosocket API functions (e.g., [ngx.socket.tcp](#ngxsockettcp) and [ngx.req.socket](#ngxreqsocket)).
* Sleeping API function [ngx.sleep](#ngxsleep).

In addition, note that this directive can only write out a value to a single Nginx variable at
a time. However, a workaround is possible using the [ngx.var.VARIABLE](#ngxvarvariable) interface.

```nginx

 location /foo {
     set $diff ''; # we have to predefine the $diff variable here

     set_by_lua $sum '
         local a = 32
         local b = 56

         ngx.var.diff = a - b;  -- write to $diff directly
         return a + b;          -- return the $sum value normally
     ';

     echo "sum = $sum, diff = $diff";
 }
```

This directive can be freely mixed with all directives of the [ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html), [set-misc-nginx-module](http://github.com/openresty/set-misc-nginx-module), and [array-var-nginx-module](http://github.com/openresty/array-var-nginx-module) modules. All of these directives will run in the same order as they appear in the config file.

```nginx

 set $foo 32;
 set_by_lua $bar 'tonumber(ngx.var.foo) + 1';
 set $baz "bar: $bar";  # $baz == "bar: 33"
```

As from the `v0.5.0rc29` release, Nginx variable interpolation is disabled in the `<lua-script-str>` argument of this directive and therefore, the dollar sign character (`$`) can be used directly.

This directive requires the [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit) module.

[回到目录](#directives)

set_by_lua_file
---------------
**syntax:** *set_by_lua_file $res &lt;path-to-lua-script-file&gt; [$arg1 $arg2 ...]*

**context:** *server, server if, location, location if*

**phase:** *rewrite*

Equivalent to [set_by_lua](#set_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or, as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed. 

Nginx variable interpolation is supported in the `<path-to-lua-script-file>` argument string of this directive. But special care must be taken for injection attacks.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

When the Lua code cache is turned on (by default), the user code is loaded once at the first request and cached 
and the Nginx config must be reloaded each time the Lua source file is modified.
The Lua code cache can be temporarily disabled during development by 
switching [lua_code_cache](#lua_code_cache) `off` in `nginx.conf` to avoid reloading Nginx.

This directive requires the [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit) module.

[回到目录](#directives)

content_by_lua
--------------

**syntax:** *content_by_lua &lt;lua-script-str&gt;*

**context:** *location, location if*

**phase:** *content*

Acts as a "content handler" and executes Lua code string specified in `<lua-script-str>` for every request. 
The Lua code may make [API calls](#nginx-api-for-lua) and is executed as a new spawned coroutine in an independent global environment (i.e. a sandbox).

Do not use this directive and other content handler directives in the same location. For example, this directive and the [proxy_pass](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass) directive should not be used in the same location.

[回到目录](#directives)

content_by_lua_file
-------------------

**syntax:** *content_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *location, location if*

**phase:** *content*

Equivalent to [content_by_lua](#content_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or, as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

Nginx variables can be used in the `<path-to-lua-script-file>` string to provide flexibility. This however carries some risks and is not ordinarily recommended.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

When the Lua code cache is turned on (by default), the user code is loaded once at the first request and cached 
and the Nginx config must be reloaded each time the Lua source file is modified.
The Lua code cache can be temporarily disabled during development by 
switching [lua_code_cache](#lua_code_cache) `off` in `nginx.conf` to avoid reloading Nginx.

Nginx variables are supported in the file path for dynamic dispatch, for example:

```nginx

 # WARNING: contents in nginx var must be carefully filtered,
 # otherwise there'll be great security risk!
 location ~ ^/app/([-_a-zA-Z0-9/]+) {
     set $path $1;
     content_by_lua_file /path/to/lua/app/root/$path.lua;
 }
```

But be very careful about malicious user inputs and always carefully validate or filter out the user-supplied path components.

[回到目录](#directives)

rewrite_by_lua
--------------

**syntax:** *rewrite_by_lua &lt;lua-script-str&gt;*

**context:** *http, server, location, location if*

**phase:** *rewrite tail*

Acts as a rewrite phase handler and executes Lua code string specified in `<lua-script-str>` for every request.
The Lua code may make [API calls](#nginx-api-for-lua) and is executed as a new spawned coroutine in an independent global environment (i.e. a sandbox).

Note that this handler always runs *after* the standard [ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html). So the following will work as expected:

```nginx

 location /foo {
     set $a 12; # create and initialize $a
     set $b ""; # create and initialize $b
     rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
     echo "res = $b";
 }
```

because `set $a 12` and `set $b ""` run *before* [rewrite_by_lua](#rewrite_by_lua).

On the other hand, the following will not work as expected:

```nginx

 ?  location /foo {
 ?      set $a 12; # create and initialize $a
 ?      set $b ''; # create and initialize $b
 ?      rewrite_by_lua 'ngx.var.b = tonumber(ngx.var.a) + 1';
 ?      if ($b = '13') {
 ?         rewrite ^ /bar redirect;
 ?         break;
 ?      }
 ?
 ?      echo "res = $b";
 ?  }
```

because `if` runs *before* [rewrite_by_lua](#rewrite_by_lua) even if it is placed after [rewrite_by_lua](#rewrite_by_lua) in the config.

The right way of doing this is as follows:

```nginx

 location /foo {
     set $a 12; # create and initialize $a
     set $b ''; # create and initialize $b
     rewrite_by_lua '
         ngx.var.b = tonumber(ngx.var.a) + 1
         if tonumber(ngx.var.b) == 13 then
             return ngx.redirect("/bar");
         end
     ';

     echo "res = $b";
 }
```

Note that the [ngx_eval](http://www.grid.net.ru/nginx/eval.en.html) module can be approximated by using [rewrite_by_lua](#rewrite_by_lua). For example,

```nginx

 location / {
     eval $res {
         proxy_pass http://foo.com/check-spam;
     }

     if ($res = 'spam') {
         rewrite ^ /terms-of-use.html redirect;
     }

     fastcgi_pass ...;
 }
```

can be implemented in ngx_lua as:

```nginx

 location = /check-spam {
     internal;
     proxy_pass http://foo.com/check-spam;
 }

 location / {
     rewrite_by_lua '
         local res = ngx.location.capture("/check-spam")
         if res.body == "spam" then
             return ngx.redirect("/terms-of-use.html")
         end
     ';

     fastcgi_pass ...;
 }
```

Just as any other rewrite phase handlers, [rewrite_by_lua](#rewrite_by_lua) also runs in subrequests.

Note that when calling `ngx.exit(ngx.OK)` within a [rewrite_by_lua](#rewrite_by_lua) handler, the nginx request processing control flow will still continue to the content handler. To terminate the current request from within a [rewrite_by_lua](#rewrite_by_lua) handler, calling [ngx.exit](#ngxexit) with status >= 200 (`ngx.HTTP_OK`) and status < 300 (`ngx.HTTP_SPECIAL_RESPONSE`) for successful quits and `ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)` (or its friends) for failures.

If the [ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)'s [rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite) directive is used to change the URI and initiate location re-lookups (internal redirections), then any [rewrite_by_lua](#rewrite_by_lua) or [rewrite_by_lua_file](#rewrite_by_lua_file) code sequences within the current location will not be executed. For example,

```nginx

 location /foo {
     rewrite ^ /bar;
     rewrite_by_lua 'ngx.exit(503)';
 }
 location /bar {
     ...
 }
```

Here the Lua code `ngx.exit(503)` will never run. This will be the case if `rewrite ^ /bar last` is used as this will similarly initiate an internal redirection. If the `break` modifier is used instead, there will be no internal redirection and the `rewrite_by_lua` code will be executed.

The `rewrite_by_lua` code will always run at the end of the `rewrite` request-processing phase unless [rewrite_by_lua_no_postpone](#rewrite_by_lua_no_postpone) is turned on.

[回到目录](#directives)

rewrite_by_lua_file
-------------------

**syntax:** *rewrite_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *http, server, location, location if*

**phase:** *rewrite tail*

Equivalent to [rewrite_by_lua](#rewrite_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or, as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

Nginx variables can be used in the `<path-to-lua-script-file>` string to provide flexibility. This however carries some risks and is not ordinarily recommended.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

When the Lua code cache is turned on (by default), the user code is loaded once at the first request and cached and the Nginx config must be reloaded each time the Lua source file is modified. The Lua code cache can be temporarily disabled during development by switching [lua_code_cache](#lua_code_cache) `off` in `nginx.conf` to avoid reloading Nginx.

The `rewrite_by_lua_file` code will always run at the end of the `rewrite` request-processing phase unless [rewrite_by_lua_no_postpone](#rewrite_by_lua_no_postpone) is turned on.

Nginx variables are supported in the file path for dynamic dispatch just as in [content_by_lua_file](#content_by_lua_file).

[回到目录](#directives)

access_by_lua
-------------

**syntax:** *access_by_lua &lt;lua-script-str&gt;*

**context:** *http, server, location, location if*

**phase:** *access tail*

Acts as an access phase handler and executes Lua code string specified in `<lua-script-str>` for every request.
The Lua code may make [API calls](#nginx-api-for-lua) and is executed as a new spawned coroutine in an independent global environment (i.e. a sandbox).

Note that this handler always runs *after* the standard [ngx_http_access_module](http://nginx.org/en/docs/http/ngx_http_access_module.html). So the following will work as expected:

```nginx

 location / {
     deny    192.168.1.1;
     allow   192.168.1.0/24;
     allow   10.1.1.0/16;
     deny    all;

     access_by_lua '
         local res = ngx.location.capture("/mysql", { ... })
         ...
     ';

     # proxy_pass/fastcgi_pass/...
 }
```

That is, if a client IP address is in the blacklist, it will be denied before the MySQL query for more complex authentication is executed by [access_by_lua](#access_by_lua).

Note that the [ngx_auth_request](http://mdounin.ru/hg/ngx_http_auth_request_module/) module can be approximated by using [access_by_lua](#access_by_lua):

```nginx

 location / {
     auth_request /auth;

     # proxy_pass/fastcgi_pass/postgres_pass/...
 }
```

can be implemented in ngx_lua as:

```nginx

 location / {
     access_by_lua '
         local res = ngx.location.capture("/auth")

         if res.status == ngx.HTTP_OK then
             return
         end

         if res.status == ngx.HTTP_FORBIDDEN then
             ngx.exit(res.status)
         end

         ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)
     ';

     # proxy_pass/fastcgi_pass/postgres_pass/...
 }
```

As with other access phase handlers, [access_by_lua](#access_by_lua) will *not* run in subrequests.

Note that when calling `ngx.exit(ngx.OK)` within a [access_by_lua](#access_by_lua) handler, the nginx request processing control flow will still continue to the content handler. To terminate the current request from within a [access_by_lua](#access_by_lua) handler, calling [ngx.exit](#ngxexit) with status >= 200 (`ngx.HTTP_OK`) and status < 300 (`ngx.HTTP_SPECIAL_RESPONSE`) for successful quits and `ngx.exit(ngx.HTTP_INTERNAL_SERVER_ERROR)` (or its friends) for failures.

[回到目录](#directives)

access_by_lua_file
------------------

**syntax:** *access_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *http, server, location, location if*

**phase:** *access tail*

Equivalent to [access_by_lua](#access_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or, as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

Nginx variables can be used in the `<path-to-lua-script-file>` string to provide flexibility. This however carries some risks and is not ordinarily recommended.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

When the Lua code cache is turned on (by default), the user code is loaded once at the first request and cached 
and the Nginx config must be reloaded each time the Lua source file is modified.
The Lua code cache can be temporarily disabled during development by switching [lua_code_cache](#lua_code_cache) `off` in `nginx.conf` to avoid repeatedly reloading Nginx.

Nginx variables are supported in the file path for dynamic dispatch just as in [content_by_lua_file](#content_by_lua_file).

[回到目录](#directives)

header_filter_by_lua
--------------------

**syntax:** *header_filter_by_lua &lt;lua-script-str&gt;*

**context:** *http, server, location, location if*

**phase:** *output-header-filter*

Uses Lua code specified in `<lua-script-str>` to define an output header filter.

Note that the following API functions are currently disabled within this context:

* Output API functions (e.g., [ngx.say](#ngxsay) and [ngx.send_headers](#ngxsend_headers))
* Control API functions (e.g., [ngx.exit](#ngxexit) and [ngx.exec](#ngxexec))
* Subrequest API functions (e.g., [ngx.location.capture](#ngxlocationcapture) and [ngx.location.capture_multi](#ngxlocationcapture_multi))
* Cosocket API functions (e.g., [ngx.socket.tcp](#ngxsockettcp) and [ngx.req.socket](#ngxreqsocket)).

Here is an example of overriding a response header (or adding one if absent) in our Lua header filter:

```nginx

 location / {
     proxy_pass http://mybackend;
     header_filter_by_lua 'ngx.header.Foo = "blah"';
 }
```

This directive was first introduced in the `v0.2.1rc20` release.

[回到目录](#directives)

header_filter_by_lua_file
-------------------------

**syntax:** *header_filter_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *http, server, location, location if*

**phase:** *output-header-filter*

Equivalent to [header_filter_by_lua](#header_filter_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

This directive was first introduced in the `v0.2.1rc20` release.

[回到目录](#directives)

body_filter_by_lua
------------------

**syntax:** *body_filter_by_lua &lt;lua-script-str&gt;*

**context:** *http, server, location, location if*

**phase:** *output-body-filter*

Uses Lua code specified in `<lua-script-str>` to define an output body filter.

The input data chunk is passed via [ngx.arg](#ngxarg)\[1\] (as a Lua string value) and the "eof" flag indicating the end of the response body data stream is passed via [ngx.arg](#ngxarg)\[2\] (as a Lua boolean value).

Behind the scene, the "eof" flag is just the `last_buf` (for main requests) or `last_in_chain` (for subrequests) flag of the Nginx chain link buffers. (Before the `v0.7.14` release, the "eof" flag does not work at all in subrequests.)

The output data stream can be aborted immediately by running the following Lua statement:

```lua

 return ngx.ERROR
```

This will truncate the response body and usually result in incomplete and also invalid responses.

The Lua code can pass its own modified version of the input data chunk to the downstream Nginx output body filters by overriding [ngx.arg](#ngxarg)\[1\] with a Lua string or a Lua table of strings. For example, to transform all the lowercase letters in the response body, we can just write:

```nginx

 location / {
     proxy_pass http://mybackend;
     body_filter_by_lua 'ngx.arg[1] = string.upper(ngx.arg[1])';
 }
```

When setting `nil` or an empty Lua string value to `ngx.arg[1]`, no data chunk will be passed to the downstream Nginx output filters at all.

Likewise, new "eof" flag can also be specified by setting a boolean value to [ngx.arg](#ngxarg)\[2\]. For example,

```nginx

 location /t {
     echo hello world;
     echo hiya globe;

     body_filter_by_lua '
         local chunk = ngx.arg[1]
         if string.match(chunk, "hello") then
             ngx.arg[2] = true  -- new eof
             return
         end

         -- just throw away any remaining chunk data
         ngx.arg[1] = nil
     ';
 }
```

Then `GET /t` will just return the output


    hello world


That is, when the body filter sees a chunk containing the word "hello", then it will set the "eof" flag to true immediately, resulting in truncated but still valid responses.

When the Lua code may change the length of the response body, then it is required to always clear out the `Content-Length` response header (if any) in a header filter to enforce streaming output, as in

```nginx

 location /foo {
     # fastcgi_pass/proxy_pass/...

     header_filter_by_lua 'ngx.header.content_length = nil';
     body_filter_by_lua 'ngx.arg[1] = string.len(ngx.arg[1]) .. "\\n"';
 }
```

Note that the following API functions are currently disabled within this context due to the limitations in NGINX output filter's current implementation:

* Output API functions (e.g., [ngx.say](#ngxsay) and [ngx.send_headers](#ngxsend_headers))
* Control API functions (e.g., [ngx.exit](#ngxexit) and [ngx.exec](#ngxexec))
* Subrequest API functions (e.g., [ngx.location.capture](#ngxlocationcapture) and [ngx.location.capture_multi](#ngxlocationcapture_multi))
* Cosocket API functions (e.g., [ngx.socket.tcp](#ngxsockettcp) and [ngx.req.socket](#ngxreqsocket)).

Nginx output filters may be called multiple times for a single request because response body may be delivered in chunks. Thus, the Lua code specified by in this directive may also run multiple times in the lifetime of a single HTTP request.

This directive was first introduced in the `v0.5.0rc32` release.

[回到目录](#directives)

body_filter_by_lua_file
-----------------------

**syntax:** *body_filter_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *http, server, location, location if*

**phase:** *output-body-filter*

Equivalent to [body_filter_by_lua](#body_filter_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or, as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

This directive was first introduced in the `v0.5.0rc32` release.

[回到目录](#directives)

log_by_lua
----------

**syntax:** *log_by_lua &lt;lua-script-str&gt;*

**context:** *http, server, location, location if*

**phase:** *log*

Run the Lua source code inlined as the `<lua-script-str>` at the `log` request processing phase. This does not replace the current access logs, but runs after.

Note that the following API functions are currently disabled within this context:

* Output API functions (e.g., [ngx.say](#ngxsay) and [ngx.send_headers](#ngxsend_headers))
* Control API functions (e.g., [ngx.exit](#ngxexit)) 
* Subrequest API functions (e.g., [ngx.location.capture](#ngxlocationcapture) and [ngx.location.capture_multi](#ngxlocationcapture_multi))
* Cosocket API functions (e.g., [ngx.socket.tcp](#ngxsockettcp) and [ngx.req.socket](#ngxreqsocket)).

Here is an example of gathering average data for [$upstream_response_time](http://nginx.org/en/docs/http/ngx_http_upstream_module.html#var_upstream_response_time):

```nginx

 lua_shared_dict log_dict 5M;

 server {
     location / {
         proxy_pass http://mybackend;

         log_by_lua '
             local log_dict = ngx.shared.log_dict
             local upstream_time = tonumber(ngx.var.upstream_response_time)

             local sum = log_dict:get("upstream_time-sum") or 0
             sum = sum + upstream_time
             log_dict:set("upstream_time-sum", sum)

             local newval, err = log_dict:incr("upstream_time-nb", 1)
             if not newval and err == "not found" then
                 log_dict:add("upstream_time-nb", 0)
                 log_dict:incr("upstream_time-nb", 1)
             end
         ';
     }

     location = /status {
         content_by_lua '
             local log_dict = ngx.shared.log_dict
             local sum = log_dict:get("upstream_time-sum")
             local nb = log_dict:get("upstream_time-nb")

             if nb and sum then
                 ngx.say("average upstream response time: ", sum / nb,
                         " (", nb, " reqs)")
             else
                 ngx.say("no data yet")
             end
         ';
     }
 }
```

This directive was first introduced in the `v0.5.0rc31` release.

[回到目录](#directives)

log_by_lua_file
---------------

**syntax:** *log_by_lua_file &lt;path-to-lua-script-file&gt;*

**context:** *http, server, location, location if*

**phase:** *log*

Equivalent to [log_by_lua](#log_by_lua), except that the file specified by `<path-to-lua-script-file>` contains the Lua code, or, as from the `v0.5.0rc32` release, the [Lua/LuaJIT bytecode](#lualuajit-bytecode-support) to be executed.

When a relative path like `foo/bar.lua` is given, they will be turned into the absolute path relative to the `server prefix` path determined by the `-p PATH` command-line option while starting the Nginx server.

This directive was first introduced in the `v0.5.0rc31` release.

[回到目录](#directives)

lua_need_request_body
---------------------

**syntax:** *lua_need_request_body &lt;on|off&gt;*

**default:** *off*

**context:** *http, server, location, location if*

**phase:** *depends on usage*

Determines whether to force the request body data to be read before running rewrite/access/access_by_lua* or not. The Nginx core does not read the client request body by default and if request body data is required, then this directive should be turned `on` or the [ngx.req.read_body](#ngxreqread_body) function should be called within the Lua code.

To read the request body data within the [$request_body](http://nginx.org/en/docs/http/ngx_http_core_module.html#var_request_body) variable, 
[client_body_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) must have the same value as [client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size). Because when the content length exceeds [client_body_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) but less than [client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size), Nginx will buffer the data into a temporary file on the disk, which will lead to empty value in the [$request_body](http://nginx.org/en/docs/http/ngx_http_core_module.html#var_request_body) variable.

If the current location includes [rewrite_by_lua](#rewrite_by_lua) or [rewrite_by_lua_file](#rewrite_by_lua_file) directives,
then the request body will be read just before the [rewrite_by_lua](#rewrite_by_lua) or [rewrite_by_lua_file](#rewrite_by_lua_file) code is run (and also at the
`rewrite` phase). Similarly, if only [content_by_lua](#content_by_lua) is specified,
the request body will not be read until the content handler's Lua code is
about to run (i.e., the request body will be read during the content phase).

It is recommended however, to use the [ngx.req.read_body](#ngxreqread_body) and [ngx.req.discard_body](#ngxreqdiscard_body) functions for finer control over the request body reading process instead.

This also applies to [access_by_lua](#access_by_lua) and [access_by_lua_file](#access_by_lua_file).

[回到目录](#directives)

lua_shared_dict
---------------

**syntax:** *lua_shared_dict &lt;name&gt; &lt;size&gt;*

**default:** *no*

**context:** *http*

**phase:** *depends on usage*

Declares a shared memory zone, `<name>`, to serve as storage for the shm based Lua dictionary `ngx.shared.<name>`.

Shared memory zones are always shared by all the nginx worker processes in the current nginx server instance.

The `<size>` argument accepts size units such as `k` and `m`:

```nginx

 http {
     lua_shared_dict dogs 10m;
     ...
 }
```

See [ngx.shared.DICT](#ngxshareddict) for details.

This directive was first introduced in the `v0.3.1rc22` release.

[回到目录](#directives)

lua_socket_connect_timeout
--------------------------

**syntax:** *lua_socket_connect_timeout &lt;time&gt;*

**default:** *lua_socket_connect_timeout 60s*

**context:** *http, server, location*

This directive controls the default timeout value used in TCP/unix-domain socket object's [connect](#tcpsockconnect) method and can be overridden by the [settimeout](#tcpsocksettimeout) method.

The `<time>` argument can be an integer, with an optional time unit, like `s` (second), `ms` (millisecond), `m` (minute). The default time unit is `s`, i.e., "second". The default setting is `60s`.

This directive was first introduced in the `v0.5.0rc1` release.

[回到目录](#directives)

lua_socket_send_timeout
-----------------------

**syntax:** *lua_socket_send_timeout &lt;time&gt;*

**default:** *lua_socket_send_timeout 60s*

**context:** *http, server, location*

Controls the default timeout value used in TCP/unix-domain socket object's [send](#tcpsocksend) method and can be overridden by the [settimeout](#tcpsocksettimeout) method.

The `<time>` argument can be an integer, with an optional time unit, like `s` (second), `ms` (millisecond), `m` (minute). The default time unit is `s`, i.e., "second". The default setting is `60s`.

This directive was first introduced in the `v0.5.0rc1` release.

[回到目录](#directives)

lua_socket_send_lowat
---------------------

**syntax:** *lua_socket_send_lowat &lt;size&gt;*

**default:** *lua_socket_send_lowat 0*

**context:** *http, server, location*

Controls the `lowat` (low water) value for the cosocket send buffer.

[回到目录](#directives)

lua_socket_read_timeout
-----------------------

**syntax:** *lua_socket_read_timeout &lt;time&gt;*

**default:** *lua_socket_read_timeout 60s*

**context:** *http, server, location*

**phase:** *depends on usage*

This directive controls the default timeout value used in TCP/unix-domain socket object's [receive](#tcpsockreceive) method and iterator functions returned by the [receiveuntil](#tcpsockreceiveuntil) method. This setting can be overridden by the [settimeout](#tcpsocksettimeout) method.

The `<time>` argument can be an integer, with an optional time unit, like `s` (second), `ms` (millisecond), `m` (minute). The default time unit is `s`, i.e., "second". The default setting is `60s`.

This directive was first introduced in the `v0.5.0rc1` release.

[回到目录](#directives)

lua_socket_buffer_size
----------------------

**syntax:** *lua_socket_buffer_size &lt;size&gt;*

**default:** *lua_socket_buffer_size 4k/8k*

**context:** *http, server, location*

Specifies the buffer size used by cosocket reading operations.

This buffer does not have to be that big to hold everything at the same time because cosocket supports 100% non-buffered reading and parsing. So even `1` byte buffer size should still work everywhere but the performance could be terrible.

This directive was first introduced in the `v0.5.0rc1` release.

[回到目录](#directives)

lua_socket_pool_size
--------------------

**syntax:** *lua_socket_pool_size &lt;size&gt;*

**default:** *lua_socket_pool_size 30*

**context:** *http, server, location*

Specifies the size limit (in terms of connection count) for every cosocket connection pool associated with every remote server (i.e., identified by either the host-port pair or the unix domain socket file path).

Default to 30 connections for every pool.

When the connection pool exceeds the available size limit, the least recently used (idle) connection already in the pool will be closed to make room for the current connection.

Note that the cosocket connection pool is per nginx worker process rather than per nginx server instance, so size limit specified here also applies to every single nginx worker process.

This directive was first introduced in the `v0.5.0rc1` release.

[回到目录](#directives)

lua_socket_keepalive_timeout
----------------------------

**syntax:** *lua_socket_keepalive_timeout &lt;time&gt;*

**default:** *lua_socket_keepalive_timeout 60s*

**context:** *http, server, location*

This directive controls the default maximal idle time of the connections in the cosocket built-in connection pool. When this timeout reaches, idle connections will be closed and removed from the pool. This setting can be overridden by cosocket objects' [setkeepalive](#tcpsocksetkeepalive) method.

The `<time>` argument can be an integer, with an optional time unit, like `s` (second), `ms` (millisecond), `m` (minute). The default time unit is `s`, i.e., "second". The default setting is `60s`.

This directive was first introduced in the `v0.5.0rc1` release.

[回到目录](#directives)

lua_socket_log_errors
---------------------

**syntax:** *lua_socket_log_errors on|off*

**default:** *lua_socket_log_errors on*

**context:** *http, server, location*

This directive can be used to toggle error logging when a failure occurs for the TCP or UDP cosockets. If you are already doing proper error handling and logging in your Lua code, then it is recommended to turn this directive off to prevent data flushing in your nginx error log files (which is usually rather expensive).

This directive was first introduced in the `v0.5.13` release.

[回到目录](#directives)

lua_ssl_ciphers
---------------

**syntax:** *lua_ssl_ciphers &lt;ciphers&gt;*

**default:** *lua_ssl_ciphers DEFAULT*

**context:** *http, server, location*

Specifies the enabled ciphers for requests to a SSL/TLS server in the [tcpsock:sslhandshake](#tcpsocksslhandshake) method. The ciphers are specified in the format understood by the OpenSSL library.

The full list can be viewed using the “openssl ciphers” command.

This directive was first introduced in the `v0.9.11` release.

[回到目录](#directives)

lua_ssl_crl
-----------

**syntax:** *lua_ssl_crl &lt;file&gt;*

**default:** *no*

**context:** *http, server, location*

Specifies a file with revoked certificates (CRL) in the PEM format used to verify the certificate of the SSL/TLS server in the [tcpsock:sslhandshake](#tcpsocksslhandshake) method.

This directive was first introduced in the `v0.9.11` release.

[回到目录](#directives)

lua_ssl_protocols
-----------------

**syntax:** *lua_ssl_protocols \[SSLv2\] \[SSLv3\] \[TLSv1\] [TLSv1.1] [TLSv1.2]*

**default:** *lua_ssl_protocols SSLv3 TLSv1 TLSv1.1 TLSv1.2*

**context:** *http, server, location*

Enables the specified protocols for requests to a SSL/TLS server in the [tcpsock:sslhandshake](#tcpsocksslhandshake) method.

This directive was first introduced in the `v0.9.11` release.

[回到目录](#directives)

lua_ssl_trusted_certificate
---------------------------

**syntax:** *lua_ssl_trusted_certificate &lt;file&gt;*

**default:** *no*

**context:** *http, server, location*

Specifies a file path with trusted CA certificates in the PEM format used to verify the certificate of the SSL/TLS server in the [tcpsock:sslhandshake](#tcpsocksslhandshake) method.

This directive was first introduced in the `v0.9.11` release.

See also [lua_ssl_verify_depth](#lua_ssl_verify_depth).

[回到目录](#directives)

lua_ssl_verify_depth
--------------------

**syntax:** *lua_ssl_verify_depth &lt;number&gt;*

**default:** *lua_ssl_verify_depth 1*

**context:** *http, server, location*

Sets the verification depth in the server certificates chain.

This directive was first introduced in the `v0.9.11` release.

See also [lua_ssl_trusted_certificate](#lua_ssl_trusted_certificate).

[回到目录](#directives)

lua_http10_buffering
--------------------

**syntax:** *lua_http10_buffering on|off*

**default:** *lua_http10_buffering on*

**context:** *http, server, location, location-if*

Enables or disables automatic response buffering for HTTP 1.0 (or older) requests. This buffering mechanism is mainly used for HTTP 1.0 keep-alive which replies on a proper `Content-Length` response header.

If the Lua code explicitly sets a `Content-Length` response header before sending the headers (either explicitly via [ngx.send_headers](#ngxsend_headers) or implicitly via the first [ngx.say](#ngxsay) or [ngx.print](#ngxprint) call), then the HTTP 1.0 response buffering will be disabled even when this directive is turned on.

To output very large response data in a streaming fashion (via the [ngx.flush](#ngxflush) call, for example), this directive MUST be turned off to minimize memory usage.

This directive is turned `on` by default.

This directive was first introduced in the `v0.5.0rc19` release.

[回到目录](#directives)

rewrite_by_lua_no_postpone
--------------------------

**syntax:** *rewrite_by_lua_no_postpone on|off*

**default:** *rewrite_by_lua_no_postpone off*

**context:** *http*

Controls whether or not to disable postponing [rewrite_by_lua](#rewrite_by_lua) and [rewrite_by_lua_file](#rewrite_by_lua_file) directives to run at the end of the `rewrite` request-processing phase. By default, this directive is turned off and the Lua code is postponed to run at the end of the `rewrite` phase.

This directive was first introduced in the `v0.5.0rc29` release.

[回到目录](#directives)

lua_transform_underscores_in_response_headers
---------------------------------------------

**syntax:** *lua_transform_underscores_in_response_headers on|off*

**default:** *lua_transform_underscores_in_response_headers on*

**context:** *http, server, location, location-if*

Controls whether to transform underscores (`_`) in the response header names specified in the [ngx.header.HEADER](#ngxheaderheader) API to hypens (`-`).

This directive was first introduced in the `v0.5.0rc32` release.

[回到目录](#directives)

lua_check_client_abort
----------------------

**syntax:** *lua_check_client_abort on|off*

**default:** *lua_check_client_abort off*

**context:** *http, server, location, location-if*

This directive controls whether to check for premature client connection abortion.

When this directive is turned on, the ngx_lua module will monitor the premature connection close event on the downstream connections. And when there is such an event, it will call the user Lua function callback (registered by [ngx.on_abort](#ngxon_abort)) or just stop and clean up all the Lua "light threads" running in the current request's request handler when there is no user callback function registered.

According to the current implementation, however, if the client closes the connection before the Lua code finishes reading the request body data via [ngx.req.socket](#ngxreqsocket), then ngx_lua will neither stop all the running "light threads" nor call the user callback (if [ngx.on_abort](#ngxon_abort) has been called). Instead, the reading operation on [ngx.req.socket](#ngxreqsocket) will just return the error message "client aborted" as the second return value (the first return value is surely `nil`).

When TCP keepalive is disabled, it is relying on the client side to close the socket gracefully (by sending a `FIN` packet or something like that). For (soft) real-time web applications, it is highly recommended to configure the [TCP keepalive](http://tldp.org/HOWTO/TCP-Keepalive-HOWTO/overview.html) support in your system's TCP stack implementation in order to detect "half-open" TCP connections in time.

For example, on Linux, you can configure the standard [listen](http://nginx.org/en/docs/http/ngx_http_core_module.html#listen) directive in your `nginx.conf` file like this:

```nginx

 listen 80 so_keepalive=2s:2s:8;
```

On FreeBSD, you can only tune the system-wide configuration for TCP keepalive, for example:

    # sysctl net.inet.tcp.keepintvl=2000
    # sysctl net.inet.tcp.keepidle=2000

This directive was first introduced in the `v0.7.4` release.

See also [ngx.on_abort](#ngxon_abort).

[回到目录](#directives)

lua_max_pending_timers
----------------------

**syntax:** *lua_max_pending_timers &lt;count&gt;*

**default:** *lua_max_pending_timers 1024*

**context:** *http*

Controls the maximum number of pending timers allowed.

Pending timers are those timers that have not expired yet.

When exceeding this limit, the [ngx.timer.at](#ngxtimerat) call will immediately return `nil` and the error string "too many pending timers".

This directive was first introduced in the `v0.8.0` release.

[回到目录](#directives)

lua_max_running_timers
----------------------

**syntax:** *lua_max_running_timers &lt;count&gt;*

**default:** *lua_max_running_timers 256*

**context:** *http*

Controls the maximum number of "running timers" allowed.

Running timers are those timers whose user callback functions are still running.

When exceeding this limit, Nginx will stop running the callbacks of newly expired timers and log an error message "N lua_max_running_timers are not enough" where "N" is the current value of this directive.

This directive was first introduced in the `v0.8.0` release.

[回到目录](#directives)

Nginx API for Lua
=================

* [Introduction](#introduction)
* [ngx.arg](#ngxarg)
* [ngx.var.VARIABLE](#ngxvarvariable)
* [Core constants](#core-constants)
* [HTTP method constants](#http-method-constants)
* [HTTP status constants](#http-status-constants)
* [Nginx log level constants](#nginx-log-level-constants)
* [print](#print)
* [ngx.ctx](#ngxctx)
* [ngx.location.capture](#ngxlocationcapture)
* [ngx.location.capture_multi](#ngxlocationcapture_multi)
* [ngx.status](#ngxstatus)
* [ngx.header.HEADER](#ngxheaderheader)
* [ngx.resp.get_headers](#ngxrespget_headers)
* [ngx.req.start_time](#ngxreqstart_time)
* [ngx.req.http_version](#ngxreqhttp_version)
* [ngx.req.raw_header](#ngxreqraw_header)
* [ngx.req.get_method](#ngxreqget_method)
* [ngx.req.set_method](#ngxreqset_method)
* [ngx.req.set_uri](#ngxreqset_uri)
* [ngx.req.set_uri_args](#ngxreqset_uri_args)
* [ngx.req.get_uri_args](#ngxreqget_uri_args)
* [ngx.req.get_post_args](#ngxreqget_post_args)
* [ngx.req.get_headers](#ngxreqget_headers)
* [ngx.req.set_header](#ngxreqset_header)
* [ngx.req.clear_header](#ngxreqclear_header)
* [ngx.req.read_body](#ngxreqread_body)
* [ngx.req.discard_body](#ngxreqdiscard_body)
* [ngx.req.get_body_data](#ngxreqget_body_data)
* [ngx.req.get_body_file](#ngxreqget_body_file)
* [ngx.req.set_body_data](#ngxreqset_body_data)
* [ngx.req.set_body_file](#ngxreqset_body_file)
* [ngx.req.init_body](#ngxreqinit_body)
* [ngx.req.append_body](#ngxreqappend_body)
* [ngx.req.finish_body](#ngxreqfinish_body)
* [ngx.req.socket](#ngxreqsocket)
* [ngx.exec](#ngxexec)
* [ngx.redirect](#ngxredirect)
* [ngx.send_headers](#ngxsend_headers)
* [ngx.headers_sent](#ngxheaders_sent)
* [ngx.print](#ngxprint)
* [ngx.say](#ngxsay)
* [ngx.log](#ngxlog)
* [ngx.flush](#ngxflush)
* [ngx.exit](#ngxexit)
* [ngx.eof](#ngxeof)
* [ngx.sleep](#ngxsleep)
* [ngx.escape_uri](#ngxescape_uri)
* [ngx.unescape_uri](#ngxunescape_uri)
* [ngx.encode_args](#ngxencode_args)
* [ngx.decode_args](#ngxdecode_args)
* [ngx.encode_base64](#ngxencode_base64)
* [ngx.decode_base64](#ngxdecode_base64)
* [ngx.crc32_short](#ngxcrc32_short)
* [ngx.crc32_long](#ngxcrc32_long)
* [ngx.hmac_sha1](#ngxhmac_sha1)
* [ngx.md5](#ngxmd5)
* [ngx.md5_bin](#ngxmd5_bin)
* [ngx.sha1_bin](#ngxsha1_bin)
* [ngx.quote_sql_str](#ngxquote_sql_str)
* [ngx.today](#ngxtoday)
* [ngx.time](#ngxtime)
* [ngx.now](#ngxnow)
* [ngx.update_time](#ngxupdate_time)
* [ngx.localtime](#ngxlocaltime)
* [ngx.utctime](#ngxutctime)
* [ngx.cookie_time](#ngxcookie_time)
* [ngx.http_time](#ngxhttp_time)
* [ngx.parse_http_time](#ngxparse_http_time)
* [ngx.is_subrequest](#ngxis_subrequest)
* [ngx.re.match](#ngxrematch)
* [ngx.re.find](#ngxrefind)
* [ngx.re.gmatch](#ngxregmatch)
* [ngx.re.sub](#ngxresub)
* [ngx.re.gsub](#ngxregsub)
* [ngx.shared.DICT](#ngxshareddict)
* [ngx.shared.DICT.get](#ngxshareddictget)
* [ngx.shared.DICT.get_stale](#ngxshareddictget_stale)
* [ngx.shared.DICT.set](#ngxshareddictset)
* [ngx.shared.DICT.safe_set](#ngxshareddictsafe_set)
* [ngx.shared.DICT.add](#ngxshareddictadd)
* [ngx.shared.DICT.safe_add](#ngxshareddictsafe_add)
* [ngx.shared.DICT.replace](#ngxshareddictreplace)
* [ngx.shared.DICT.delete](#ngxshareddictdelete)
* [ngx.shared.DICT.incr](#ngxshareddictincr)
* [ngx.shared.DICT.flush_all](#ngxshareddictflush_all)
* [ngx.shared.DICT.flush_expired](#ngxshareddictflush_expired)
* [ngx.shared.DICT.get_keys](#ngxshareddictget_keys)
* [ngx.socket.udp](#ngxsocketudp)
* [udpsock:setpeername](#udpsocksetpeername)
* [udpsock:send](#udpsocksend)
* [udpsock:receive](#udpsockreceive)
* [udpsock:close](#udpsockclose)
* [udpsock:settimeout](#udpsocksettimeout)
* [ngx.socket.tcp](#ngxsockettcp)
* [tcpsock:connect](#tcpsockconnect)
* [tcpsock:sslhandshake](#tcpsocksslhandshake)
* [tcpsock:send](#tcpsocksend)
* [tcpsock:receive](#tcpsockreceive)
* [tcpsock:receiveuntil](#tcpsockreceiveuntil)
* [tcpsock:close](#tcpsockclose)
* [tcpsock:settimeout](#tcpsocksettimeout)
* [tcpsock:setoption](#tcpsocksetoption)
* [tcpsock:setkeepalive](#tcpsocksetkeepalive)
* [tcpsock:getreusedtimes](#tcpsockgetreusedtimes)
* [ngx.socket.connect](#ngxsocketconnect)
* [ngx.get_phase](#ngxget_phase)
* [ngx.thread.spawn](#ngxthreadspawn)
* [ngx.thread.wait](#ngxthreadwait)
* [ngx.thread.kill](#ngxthreadkill)
* [ngx.on_abort](#ngxon_abort)
* [ngx.timer.at](#ngxtimerat)
* [ngx.config.debug](#ngxconfigdebug)
* [ngx.config.prefix](#ngxconfigprefix)
* [ngx.config.nginx_version](#ngxconfignginx_version)
* [ngx.config.nginx_configure](#ngxconfignginx_configure)
* [ngx.config.ngx_lua_version](#ngxconfigngx_lua_version)
* [ngx.worker.exiting](#ngxworkerexiting)
* [ngx.worker.pid](#ngxworkerpid)
* [ndk.set_var.DIRECTIVE](#ndkset_vardirective)
* [coroutine.create](#coroutinecreate)
* [coroutine.resume](#coroutineresume)
* [coroutine.yield](#coroutineyield)
* [coroutine.wrap](#coroutinewrap)
* [coroutine.running](#coroutinerunning)
* [coroutine.status](#coroutinestatus)


[回到目录](#table-of-contents)

Introduction
------------
The various `*_by_lua` and `*_by_lua_file` configuration directives serve as gateways to the Lua API within the `nginx.conf` file. The Nginx Lua API described below can only be called within the user Lua code run in the context of these configuration directives.

The API is exposed to Lua in the form of two standard packages `ngx` and `ndk`. These packages are in the default global scope within ngx_lua and are always available within ngx_lua directives.

The packages can be introduced into external Lua modules like this:

```lua

 local say = ngx.say

 local _M = {}

 function _M.foo(a)
     say(a)
 end

 return _M
```

Use of the [package.seeall](http://www.lua.org/manual/5.1/manual.html#pdf-package.seeall) flag is strongly discouraged due to its various bad side-effects.

It is also possible to directly require the packages in external Lua modules:

```lua

 local ngx = require "ngx"
 local ndk = require "ndk"
```

The ability to require these packages was introduced in the `v0.2.1rc19` release.

Network I/O operations in user code should only be done through the Nginx Lua API calls as the Nginx event loop may be blocked and performance drop off dramatically otherwise. Disk operations with relatively small amount of data can be done using the standard Lua `io` library but huge file reading and writing should be avoided wherever possible as they may block the Nginx process significantly. Delegating all network and disk I/O operations to Nginx's subrequests (via the [ngx.location.capture](#ngxlocationcapture) method and similar) is strongly recommended for maximum performance.

[回到目录](#nginx-api-for-lua)

ngx.arg
-------
**语法:** *val = ngx.arg\[index\]*

**上下文:** *set_by_lua*, body_filter_by_lua**

当用于[set_by_lua](#set_by_lua)或[set_by_lua_file](#set_by_lua_file)指令上下文时，该table是只读的，包含输入参数：

```lua

 value = ngx.arg[n]
```

下面是一个例子：

```nginx

 location /foo {
     set $a 32;
     set $b 56;

     set_by_lua $sum
         'return tonumber(ngx.arg[1]) + tonumber(ngx.arg[2])'
         $a $b;

     echo $sum;
 }
```

输出`88`，`32`和`56`的和。

当用于[body_filter_by_lua](#body_filter_by_lua)或[body_filter_by_lua_file](#body_filter_by_lua_file)上下文时，第一个元素为传给输出过滤代码的输入数据块，第二个元素是标示整个输出数据流结束的布尔标志。

传给Nginx下游输出过滤器的数据块和”eof“标志可以通过直接赋值给对应table元素来重写。当设定`nil`或空字符串给`ngx.arg[1]`时，不会有数据块传给下游Nginx输出过滤器。

[回到目录](#nginx-api-for-lua)

ngx.var.VARIABLE
----------------
**语法:** *ngx.var.VAR_NAME*

**上下文:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua**

读写Nginx变量的值。

```nginx

 value = ngx.var.some_nginx_variable_name
 ngx.var.some_nginx_variable_name = value
```

注意，只有已经定义的nginx变量才能被写入。例如：

```nginx

 location /foo {
     set $my_var ''; # 这一行是必须的，用于在配置时生成$my_var变量
     content_by_lua '
         ngx.var.my_var = 123;
         ...
     ';
 }
```

也就是说，nginx变量不能凭空创建，需要set指令。

像`$args`和`$limit_rate`这样的特殊nginx变量可以被赋值，其他像`$query_string`，`$arg_PARAMETER`和`$http_NAME`这些变量不能修改。

Nginx正则捕获组变量`$1`，`$2`，`$3`等也可以通过此方法被读取，写法为`ngx.var[1]`，`ngx.var[2]`，`ngx.var[3]`等。

将`ngx.var.Foo`置为`nil`会把Nginx变量`$Foo`的值重置。

```lua

 ngx.var.args = nil
```

**警告** 当读取Nginx变量时，Nginx会在每个请求的内存池中分配内存，分配的内存在请求结束时才会释放内存。因此，如果在你的Lua代码中要频繁的读取一个Nginx变量，推荐的做法是在Lua局部变量中环城该Nginx变量。例如：

```lua

 local val = ngx.var.some_var
 --- 之后可以频繁使用val了
```

在当前请求的生命周期内，防止（临时）内存泄漏的另一种做法是使用[ngx.ctx](#ngxctx)缓存结果。

该API需要相对昂贵的元方法调用，建议避免在热代码中使用它。

[回到目录](#nginx-api-for-lua)

Core constants
--------------
**上下文:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, *log_by_lua*, ngx.timer.**

```lua

   ngx.OK (0)
   ngx.ERROR (-1)
   ngx.AGAIN (-2)
   ngx.DONE (-4)
   ngx.DECLINED (-5)
```

注意，以上只有三个常量被[Nginx API for Lua](#nginx-api-for-lua)使用（如，[ngx.exit](#ngxexit)接受`NGX_OK`，`NGX_ERROR`和`NGX_DECLINED`作为输入）。

```lua

   ngx.null
```

`ngx.null`常量是一个通常用于表示Lua表中nil值的`NULL`用户数据，类似于[lua-cjson](http://www.kyne.com.au/~mark/software/lua-cjson.php)库中的`cjson.null`常量。该常量首次出现在`v0.5.0rc5`版本中。

`ngx.DECLINED`常量首次出现在`v0.5.0rc19`版本中。

[回到目录](#nginx-api-for-lua)

HTTP method constants
---------------------
**上下文:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**


      ngx.HTTP_GET
      ngx.HTTP_HEAD
      ngx.HTTP_PUT
      ngx.HTTP_POST
      ngx.HTTP_DELETE
      ngx.HTTP_OPTIONS   (added in the v0.5.0rc24 release)
      ngx.HTTP_MKCOL     (added in the v0.8.2 release)
      ngx.HTTP_COPY      (added in the v0.8.2 release)
      ngx.HTTP_MOVE      (added in the v0.8.2 release)
      ngx.HTTP_PROPFIND  (added in the v0.8.2 release)
      ngx.HTTP_PROPPATCH (added in the v0.8.2 release)
      ngx.HTTP_LOCK      (added in the v0.8.2 release)
      ngx.HTTP_UNLOCK    (added in the v0.8.2 release)
      ngx.HTTP_PATCH     (added in the v0.8.2 release)
      ngx.HTTP_TRACE     (added in the v0.8.2 release)


这些常量通常用于[ngx.location.capture](#ngxlocationcapture)和[ngx.location.capture_multi](#ngxlocationcapture_multi)方法调用。

[回到目录](#nginx-api-for-lua)

HTTP status constants
---------------------
**上下文:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**

```nginx

   value = ngx.HTTP_OK (200)
   value = ngx.HTTP_CREATED (201)
   value = ngx.HTTP_SPECIAL_RESPONSE (300)
   value = ngx.HTTP_MOVED_PERMANENTLY (301)
   value = ngx.HTTP_MOVED_TEMPORARILY (302)
   value = ngx.HTTP_SEE_OTHER (303)
   value = ngx.HTTP_NOT_MODIFIED (304)
   value = ngx.HTTP_BAD_REQUEST (400)
   value = ngx.HTTP_UNAUTHORIZED (401)
   value = ngx.HTTP_FORBIDDEN (403)
   value = ngx.HTTP_NOT_FOUND (404)
   value = ngx.HTTP_NOT_ALLOWED (405)
   value = ngx.HTTP_GONE (410)
   value = ngx.HTTP_INTERNAL_SERVER_ERROR (500)
   value = ngx.HTTP_METHOD_NOT_IMPLEMENTED (501)
   value = ngx.HTTP_SERVICE_UNAVAILABLE (503)
   value = ngx.HTTP_GATEWAY_TIMEOUT (504) (first added in the v0.3.1rc38 release)
```

[回到目录](#nginx-api-for-lua)

Nginx log level constants
-------------------------
**上下文:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**

```lua

   ngx.STDERR
   ngx.EMERG
   ngx.ALERT
   ngx.CRIT
   ngx.ERR
   ngx.WARN
   ngx.NOTICE
   ngx.INFO
   ngx.DEBUG
```

这些常量通常用于[ngx.log](#ngxlog)方法。

[回到目录](#nginx-api-for-lua)

print
-----
**语法:** *print(...)*

**上下文:** *init_by_lua*, init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**

使用`ngx.NOTICE`日志级别，将参数值写入nginx的`error.log`文件。

这等价于：

```lua

 ngx.log(ngx.NOTICE, ...)
```

`nil`参数也是可接受的，会输出`"nil"`字符串，布尔变量会输出`"true"`或者`"false"`字符串。`ngx.null`常量会输出`"null"`字符串。

在Nginx内核中，有错误信息长度`2048`字节的硬编码限制。该限制包括末尾的换行符和时间戳前缀。如果信息长度超过该限制，Nginx会相应地截断输出。该限制可以通过修改`NGX_MAX_ERROR_STR`宏定义来手动改变该限制，该宏定义在Nginx源文件`src/core/ngx_log.h`中。

[回到目录](#nginx-api-for-lua)

ngx.ctx
-------
**上下文:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua*, ngx.timer.**

该table可以用于存储每请求的Lua上下文数据，与当前请求有相同的生命周期（和Nginx变量一样）。

考虑下面的例子：

```nginx

 location /test {
     rewrite_by_lua '
         ngx.ctx.foo = 76
     ';
     access_by_lua '
         ngx.ctx.foo = ngx.ctx.foo + 3
     ';
     content_by_lua '
         ngx.say(ngx.ctx.foo)
     ';
 }
```

`GET /test`会输出：

```bash

 79
```

也就是说，`ngx.ctx.foo`的值是可以跨请求的rewrite、access和content阶段的。

包括子请求在内的所有请求都有一份自己的table拷贝。例如：

```nginx

 location /sub {
     content_by_lua '
         ngx.say("sub pre: ", ngx.ctx.blah)
         ngx.ctx.blah = 32
         ngx.say("sub post: ", ngx.ctx.blah)
     ';
 }

 location /main {
     content_by_lua '
         ngx.ctx.blah = 73
         ngx.say("main pre: ", ngx.ctx.blah)
         local res = ngx.location.capture("/sub")
         ngx.print(res.body)
         ngx.say("main post: ", ngx.ctx.blah)
     ';
 }
```

`GET /main`将会输出：

```bash

 main pre: 73
 sub pre: nil
 sub post: 32
 main post: 73
```

这里，在子请求中对`ngx.ctx.blah`的修改不会影响父请求的变量。这是因为它们拥有两个不同版本的`ngx.ctx.blah`。

内部重定向会破坏原始请求的`ngx.ctx`数据，新请求会有一个空的`ngx.ctx`表。例如：

```nginx

 location /new {
     content_by_lua '
         ngx.say(ngx.ctx.foo)
     ';
 }

 location /orig {
     content_by_lua '
         ngx.ctx.foo = "hello"
         ngx.exec("/new")
     ';
 }
```

`GET /orig`会输出：

```bash

 nil
```

而不是输出原始的`"hello"`。

任意类型的数据，包括Lua closures和嵌套tables，都可以插入到`ngx.ctx`表。它也允许注册自定义的元方法。

用一个新表覆盖`ngx.ctx`也是支持的，例如：

```lua

 ngx.ctx = { foo = 32, bar = 54 }
```

当该表用于[init_worker_by_lua*](#init_worker_by_lua)上下文时，该表的生命周期与当前的Lua处理函数相同。

`ngx.ctx`表的索引相对昂贵的元方法调用，它比通过函数参数显示传送请求数据要慢得多。所以，不要为了节省函数参数而滥用该API，因为这通常带来性能损失。也由于元方法魔法，不要在你的函数作用域外部使用”local“局部化`ngx.ctx`。

[回到目录](#nginx-api-for-lua)

ngx.location.capture
--------------------
**语法:** *res = ngx.location.capture(uri, options?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

对`uri`发起同步非阻塞的*Nginx子请求*。

Nginx子请求用于发起非阻塞内部子请求到磁盘文件目录对应的location或任何其他Nginx C模块，如`ngx_proxy`，`ngx_fastcgi`，`ngx_memc`，`ngx_postgres`，`ngx_drizzle`或者ngx_lua自身。

但注意，子请求指示模仿HTTP接口，但并没有涉及额外的HTTP/TCP或IPC流量。一切都在内部以C级别高效运作。

子请求与HTTP的301/302跳转和内部跳转完全不同。

在初始化一个子请求之前，你应该始终读取请求体（或者通过调用[ngx.req.read_body](#ngxreqread_body)，或者通过配置[lua_need_request_body](#lua_need_request_body)）。

下面是一个简单的例子：

```lua

 res = ngx.location.capture(uri)
```

返回含有四个元素的table（`res.status`，`res.header`，`res.body`，`res.truncated`）。

`res.status`为子请求响应码。

`res.header`为子请求的所有响应头，它是一个Lua table。对于多值的响应头，值为以出现次序排列的Lua数组table。例如，如果子请求响应头中包含以下几行：

```bash

 Set-Cookie: a=3
 Set-Cookie: foo=bar
 Set-Cookie: baz=blah
```

`res.header["Set-Cookie"]`的值为table值`{"a=3", "foo=bar", "baz=blah"}`。

`res.body`为子请求响应体数据，可能是被截断过的。你总是需要检查布尔标志`res.truncated`来确定是否`res.body`为截断后数据。数据截断只可能发生在子请求遇到不可恢复错误时（如，在接收响应过程中远端提前退出连接，或子请求从远端接收响应时发生读取超时等）。

URI查询字符串可能会被连接成URI本身，例如：

```lua

 res = ngx.location.capture('/foo/bar?a=3&b=4')
```

由于Nginx内核的限制，不允许使用像`@foo`这样的具名location。对于内部使用的location请与`internal`指令一起使用。

第二个参数是可选的参数表，支持以下选项：

* `method`
    指定子请求的请求方法，值接受像`ngx.HTTP_POST`这样的常量。
* `body`
    指定子请求的请求体（只允许字符串类型）。
* `args`
    指定子请求的URI查询参数（接受字符串类型和table类型）。
* `ctx`
    为子请求指定一个table作为[ngx.ctx](#ngxctx)。这可以是当前请求的[ngx.ctx](#ngxctx)表，这样可以有效地使父请求和子请求共享相同的上下文。该选项首次出现在`v0.3.1rc25`版本中。
* `vars` 
    保存用于在子请求中设定特定Nginx变量值的table。该选项最早出现在`v0.3.1rc31`中。
* `copy_all_vars`
    指定是否拷贝当前请求的Nginx变量值到子请求中。子请求中Nginx变量的修改不会影响当前请求。该选项最早出现在`v0.3.1rc31`中。
* `share_all_vars`
    指定是否在当前请求和子请求之间共享Nginx变量。子请求中变量的修改将会影响当前请求。由于副作用，开启该选项可能导致难以调试的问题出现。请只在你完全了解你的操作时开启该选项。
* `always_forward_body`
    当该选项设置为true，如果选项`body`没有指定，当前请求的请求体总是会转发给子请求。通过[ngx.req.read_body()](#ngxreqread_body)或[lua_need_request_body on](#lua_need_request_body)读取的请求体将会直接转发给子请求，在创建子请求时不会拷贝整个请求体（无论请求体在内存中还是在临时文件中）。默认情况下，该选项为`false`，当选项`body`没有指定时，当前请求的请求体只会在子请求为`PUT`或`POST`方法时才会被转发。

例如，发起一个POST子请求可以这样做：

```lua

 res = ngx.location.capture(
     '/foo/bar',
     { method = ngx.HTTP_POST, body = 'hello, world' }
 )
```

默认情况下，`method`选项为`ngx.HTTP_GET`。

选项`args`可以指定额外的URI参数，例如：

```lua

 ngx.location.capture('/foo?a=1',
     { args = { b = 3, c = ':' } }
 )
```

等价于：

```lua

 ngx.location.capture('/foo?a=1&b=3&c=%3a')
```

也就是说，该方法会根据URI规则转义参数的键值，并将他们拼接成完整的请求字符串。传递作为`args`参数的table的格式与[ngx.encode_args](#ngxencode_args)中的格式一模一样。

选项`args`也可以是纯文本查询字符串：

```lua

 ngx.location.capture('/foo?a=1',
     { args = 'b=3&c=%3a' } }
 )
```

上面的例子在功能上与前一个示例完全一样。

选项`share_all_vars`控制是否在当前请求和子请求之间共享Nginx变量。如果该选项设定为`true`，当前请求及其对应的子请求会共享相同的Nginx变量。因此，在子请求中对Nginx变量的修改会影响到当前请求。

需要万分小心的是，变量的作用域共享会带来意想不到的副作用。更推荐使用的选项是`args`，`vars`和`copy_all_vars`。

该选项默认为`false`。

```nginx

 location /other {
     set $dog "$dog world";
     echo "$uri dog: $dog";
 }

 location /lua {
     set $dog 'hello';
     content_by_lua '
         res = ngx.location.capture("/other",
             { share_all_vars = true });

         ngx.print(res.body)
         ngx.say(ngx.var.uri, ": ", ngx.var.dog)
     ';
 }
```

访问location `/lua`会输出：

    /other dog: hello world
    /lua: hello world

选项`copy_all_vars`在父请求发起子请求时，为子请求提供了一份父请求Nginx变量的拷贝。在子请求中修改这些变量不会影响父请求或任何其他共享父请求变量的子请求。

```nginx

 location /other {
     set $dog "$dog world";
     echo "$uri dog: $dog";
 }

 location /lua {
     set $dog 'hello';
     content_by_lua '
         res = ngx.location.capture("/other",
             { copy_all_vars = true });

         ngx.print(res.body)
         ngx.say(ngx.var.uri, ": ", ngx.var.dog)
     ';
 }
```

请求`GET /lua`会输出：

    /other dog: hello world
    /lua: hello

注意，如果同时设置了`share_all_vars`和`copy_all_vars`两个选项为true，那么`share_all_vars`选项优先。

除了以上两个设置选项，也可以使用`vars`选项在子请求中为变量赋值。这些变量的设置在变量共享或复制操作之后，这提供了一个给子请求传值的更有效的方法（编码为URL参数并在Nginx配置文件中转义）。

```nginx

 location /other {
     content_by_lua '
         ngx.say("dog = ", ngx.var.dog)
         ngx.say("cat = ", ngx.var.cat)
     ';
 }

 location /lua {
     set $dog '';
     set $cat '';
     content_by_lua '
         res = ngx.location.capture("/other",
             { vars = { dog = "hello", cat = 32 }});

         ngx.print(res.body)
     ';
 }
```

访问`/lua`输出：


    dog = hello
    cat = 32


`ctx`选项用于为子请求指定一个自定义的table作为[ngx.ctx](#ngxctx)。

```nginx

 location /sub {
     content_by_lua '
         ngx.ctx.foo = "bar";
     ';
 }
 location /lua {
     content_by_lua '
         local ctx = {}
         res = ngx.location.capture("/sub", { ctx = ctx })

         ngx.say(ctx.foo);
         ngx.say(ngx.ctx.foo);
     ';
 }
```

请求`GET /lua`输出：


    bar
    nil

也可以通过`ctx`选项在父子请求之间共享相同的[ngx.ctx](#ngxctx)：

```nginx

 location /sub {
     content_by_lua '
         ngx.ctx.foo = "bar";
     ';
 }
 location /lua {
     content_by_lua '
         res = ngx.location.capture("/sub", { ctx = ngx.ctx })
         ngx.say(ngx.ctx.foo);
     ';
 }
```

请求`GET /lua`输出：


    bar


注意，通过[ngx.location.capture](#ngxlocationcapture)发起的子请求默认继承当前请求所有的请求头，这可能在某些子请求响应中产生非预期的副作用。例如，当使用标准的`ngx_proxy`模块响应子请求时，在主请求中的"Accept-Encoding: gzip"头可能导致gzip编码的响应，这无法在Lua代码中正确处理。原始请求头应该在子请求的location中通过将[proxy_pass_request_headers](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_pass_request_headers)设置为`off`忽略。

当`body`选项没有指定，并且`always_forward_body`为false（默认值）时，`POST`和`PUT`方法子请求会继承父请求的请求体（如果有的话）。

每个请求可以发起的并发子请求上限是有硬编码上限的。在老版本的Nginx中，该上限为`50`，在更新的版本，Nginx `1.1.x`中，该上限增加到`200`。当超过该上限时，下列错误会被添加到`error.log`文件中：


    [error] 13983#0: *1 subrequests cycle while processing "/uri"

如果需要，该上限可以通过修改源文件`nginx/src/http/ngx_http_request.h`中的宏定义`NGX_HTTP_MAX_SUBREQUESTS`手动改变。

也请注意通过[subrequest directives of other modules](#locations-configured-by-subrequest-directives-of-other-modules)配置的捕获location的限制。

[回到目录](#nginx-api-for-lua)

ngx.location.capture_multi
--------------------------
**语法:** *res1, res2, ... = ngx.location.capture_multi({ {uri, options?}, {uri, options?}, ... })*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

同[ngx.location.capture](#ngxlocationcapture)类似，但支持并发执行若干子请求。

该方法通过指定输入table并行发起多个子请求，并以table指定的顺序返回结果。例如：

```lua

 res1, res2, res3 = ngx.location.capture_multi{
     { "/foo", { args = "a=3&b=4" } },
     { "/bar" },
     { "/baz", { method = ngx.HTTP_POST, body = "hello" } },
 }

 if res1.status == ngx.HTTP_OK then
     ...
 end

 if res2.body == "BLAH" then
     ...
 end
```

该方法直到所有子请求都结束后才会返回。请求的总时延为所有子请求中的最长时延而不是它们的时延之和。

如果发起的子请求数目不能提前知道，Lua table可以同时用于请求和响应：

```lua

 -- construct the requests table
 local reqs = {}
 table.insert(reqs, { "/mysql" })
 table.insert(reqs, { "/postgres" })
 table.insert(reqs, { "/redis" })
 table.insert(reqs, { "/memcached" })

 -- issue all the requests at once and wait until they all return
 local resps = { ngx.location.capture_multi(reqs) }

 -- loop over the responses table
 for i, resp in ipairs(resps) do
     -- process the response table "resp"
 end
```

函数[ngx.location.capture](#ngxlocationcapture)为该函数的特殊形式。逻辑上，[ngx.location.capture](#ngxlocationcapture)的实现类似于这样：

```lua

 ngx.location.capture =
     function (uri, args)
         return ngx.location.capture_multi({ {uri, args} })
     end
```

也请注意通过[subrequest directives of other modules](#locations-configured-by-subrequest-directives-of-other-modules)配置的捕获location的限制。

[回到目录](#nginx-api-for-lua)

ngx.status
----------
**上下文:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

读写当前请求的响应状态。该函数应该在响应头被发送出去之前调用。

```lua

 ngx.status = ngx.HTTP_CREATED
 status = ngx.status
```

在响应头发送出去之后设置`ngx.status`将不起作用，但会在nginx的错误日志中留下错误信息：


    attempt to set ngx.status after sending out response headers


[回到目录](#nginx-api-for-lua)

ngx.header.HEADER
-----------------
**语法:** *ngx.header.HEADER = VALUE*

**语法:** *value = ngx.header.HEADER*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

设置、增加或者清除当前请求的头部字段。

默认情况下，字段名中的下划线(`_`)会被短横线(`-`)替换。该转换可以通过[lua_transform_unserscores_in_response_headers](#lua_transform_underscores_in_response_headers)指令关闭。

头部字段名字是`大小写敏感`的。

```lua

 -- 等同于ngx.header["Content-Type"] = 'text/plain'
 ngx.header.content_type = 'text/plain';

 ngx.header["X-My-Header"] = 'blah blah';
```

多值头部字段这样设置：

```lua

 ngx.header['Set-Cookie'] = {'a=32; path=/', 'b=4; path=/'}
```

响应头输出如下：

```bash

 Set-Cookie: a=32; path=/
 Set-Cookie: b=4; path=/
```

仅接受Lua table类型（像`Content-Type`这种单值头部，仅table的最后一个值生效）。

```lua

 ngx.header.content_type = {'a', 'b'}
```

等同于：

```lua

 ngx.header.content_type = 'b'
```

将一个槽设置为`nil`等同于从响应头中删除它：

```lua

 ngx.header["X-My-Header"] = nil;
```

同样的效果可以通过将其设置为一个空table达到:

```lua

 ngx.header["X-My-Header"] = {};
```

在发送响应头之后（或者显式地调用[ngx.send_headers](#ngxsend_headers)或者隐式地调用[ngx.print](#ngxprint)或类似的函数）设置`ngx.header.HEADER`会抛出Lua异常。

读取`ngx.header.HEADER`会返回名字为`HEADER`的响应头值。

该指令在上下文[header_filter_by_lua](#header_filter_by_lua)和[header_filter_by_lua_file](#header_filter_by_lua_file)下尤其有效，例如：

```nginx

 location /test {
     set $footer '';

     proxy_pass http://some-backend;

     header_filter_by_lua '
         if ngx.header["X-My-Header"] == "blah" then
             ngx.var.footer = "some value"
         end
     ';

     echo_after_body $footer;
 }
```

对于多值的头部，头部的所有值会按照顺序被收集，并以Lua table的形式返回。例如，响应头


    Foo: bar
    Foo: baz

通过读取`ngx.header.Foo`会返回

```lua

 {"bar", "baz"}
```

注意，`ngx.header`并不是一个正规的Lua table类型，因此你不能用`ipaires`函数来迭代遍历它。

要读取*请求*头，请使用[ngx.req.get_headers](#ngxreqget_headers)函数。

[回到目录](#nginx-api-for-lua)

ngx.resp.get_headers
--------------------
**语法:** *headers = ngx.resp.get_headers(max_headers?, raw?)*

**上下文:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

对当前请求返回包含所有当前响应头的Lua table。

```lua

 local h = ngx.resp.get_headers()
 for k, v in pairs(h) do
     ...
 end
```

除了获取请求头部和响应头部的区别，该函数与[ngx.req.get_headers](#ngxreqget_headers)并不差别。

该API最早出现在`v0.9.5`版本。

[回到目录](#nginx-api-for-lua)

ngx.req.start_time
------------------
**syntax:** *secs = ngx.req.start_time()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua**

Returns a floating-point number representing the timestamp (including milliseconds as the decimal part) when the current request was created.

The following example emulates the `$request_time` variable value (provided by [ngx_http_log_module](http://nginx.org/en/docs/http/ngx_http_log_module.html)) in pure Lua:

```lua

 local request_time = ngx.now() - ngx.req.start_time()
```

This function was first introduced in the `v0.7.7` release.

See also [ngx.now](#ngxnow) and [ngx.update_time](#ngxupdate_time).

[回到目录](#nginx-api-for-lua)

ngx.req.http_version
--------------------
**syntax:** *num = ngx.req.http_version()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

Returns the HTTP version number for the current request as a Lua number.

Current possible values are 1.0, 1.1, and 0.9. Returns `nil` for unrecognized values.

This method was first introduced in the `v0.7.17` release.

[回到目录](#nginx-api-for-lua)

ngx.req.raw_header
------------------
**syntax:** *str = ngx.req.raw_header(no_request_line?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

Returns the original raw HTTP protocol header received by the Nginx server.

By default, the request line and trailing `CR LF` terminator will also be included. For example,

```lua

 ngx.print(ngx.req.raw_header())
```

gives something like this:


    GET /t HTTP/1.1
    Host: localhost
    Connection: close
    Foo: bar



You can specify the optional
`no_request_line` argument as a `true` value to exclude the request line from the result. For example,

```lua

 ngx.print(ngx.req.raw_header(true))
```

outputs something like this:


    Host: localhost
    Connection: close
    Foo: bar



This method was first introduced in the `v0.7.17` release.

[回到目录](#nginx-api-for-lua)

ngx.req.get_method
------------------
**syntax:** *method_name = ngx.req.get_method()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

Retrieves the current request's request method name. Strings like `"GET"` and `"POST"` are returned instead of numerical [method constants](#http-method-constants).

If the current request is an Nginx subrequest, then the subrequest's method name will be returned.

This method was first introduced in the `v0.5.6` release.

See also [ngx.req.set_method](#ngxreqset_method).

[回到目录](#nginx-api-for-lua)

ngx.req.set_method
------------------
**syntax:** *ngx.req.set_method(method_id)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua**

Overrides the current request's request method with the `request_id` argument. Currently only numerical [method constants](#http-method-constants) are supported, like `ngx.HTTP_POST` and `ngx.HTTP_GET`.

If the current request is an Nginx subrequest, then the subrequest's method will be overridden.

This method was first introduced in the `v0.5.6` release.

See also [ngx.req.get_method](#ngxreqget_method).

[回到目录](#nginx-api-for-lua)

ngx.req.set_uri
---------------
**syntax:** *ngx.req.set_uri(uri, jump?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua**

Rewrite the current request's (parsed) URI by the `uri` argument. The `uri` argument must be a Lua string and cannot be of zero length, or a Lua exception will be thrown.

The optional boolean `jump` argument can trigger location rematch (or location jump) as [ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html)'s [rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite) directive, that is, when `jump` is `true` (default to `false`), this function will never return and it will tell Nginx to try re-searching locations with the new URI value at the later `post-rewrite` phase and jumping to the new location.

Location jump will not be triggered otherwise, and only the current request's URI will be modified, which is also the default behavior. This function will return but with no returned values when the `jump` argument is `false` or absent altogether.

For example, the following nginx config snippet

```nginx

 rewrite ^ /foo last;
```

can be coded in Lua like this:

```lua

 ngx.req.set_uri("/foo", true)
```

Similarly, Nginx config

```nginx

 rewrite ^ /foo break;
```

can be coded in Lua as

```lua

 ngx.req.set_uri("/foo", false)
```

or equivalently,

```lua

 ngx.req.set_uri("/foo")
```

The `jump` can only be set to `true` in [rewrite_by_lua](#rewrite_by_lua) and [rewrite_by_lua_file](#rewrite_by_lua_file). Use of jump in other contexts is prohibited and will throw out a Lua exception.

A more sophisticated example involving regex substitutions is as follows

```nginx

 location /test {
     rewrite_by_lua '
         local uri = ngx.re.sub(ngx.var.uri, "^/test/(.*)", "$1", "o")
         ngx.req.set_uri(uri)
     ';
     proxy_pass http://my_backend;
 }
```

which is functionally equivalent to

```nginx

 location /test {
     rewrite ^/test/(.*) /$1 break;
     proxy_pass http://my_backend;
 }
```

Note that it is not possible to use this interface to rewrite URI arguments and that [ngx.req.set_uri_args](#ngxreqset_uri_args) should be used for this instead. For instance, Nginx config

```nginx

 rewrite ^ /foo?a=3? last;
```

can be coded as

```nginx

 ngx.req.set_uri_args("a=3")
 ngx.req.set_uri("/foo", true)
```

or

```nginx

 ngx.req.set_uri_args({a = 3})
 ngx.req.set_uri("/foo", true)
```

This interface was first introduced in the `v0.3.1rc14` release.

[回到目录](#nginx-api-for-lua)

ngx.req.set_uri_args
--------------------
**syntax:** *ngx.req.set_uri_args(args)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua**

Rewrite the current request's URI query arguments by the `args` argument. The `args` argument can be either a Lua string, as in

```lua

 ngx.req.set_uri_args("a=3&b=hello%20world")
```

or a Lua table holding the query arguments' key-value pairs, as in

```lua

 ngx.req.set_uri_args({ a = 3, b = "hello world" })
```

where in the latter case, this method will escape argument keys and values according to the URI escaping rule.

Multi-value arguments are also supported:

```lua

 ngx.req.set_uri_args({ a = 3, b = {5, 6} })
```

which will result in a query string like `a=3&b=5&b=6`.

This interface was first introduced in the `v0.3.1rc13` release.

See also [ngx.req.set_uri](#ngxreqset_uri).

[回到目录](#nginx-api-for-lua)

ngx.req.get_uri_args
--------------------
**syntax:** *args = ngx.req.get_uri_args(max_args?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

Returns a Lua table holding all the current request URL query arguments.

```nginx

 location = /test {
     content_by_lua '
         local args = ngx.req.get_uri_args()
         for key, val in pairs(args) do
             if type(val) == "table" then
                 ngx.say(key, ": ", table.concat(val, ", "))
             else
                 ngx.say(key, ": ", val)
             end
         end
     ';
 }
```

Then `GET /test?foo=bar&bar=baz&bar=blah` will yield the response body

```bash

 foo: bar
 bar: baz, blah
```

Multiple occurrences of an argument key will result in a table value holding all the values for that key in order.

Keys and values are unescaped according to URI escaping rules. In the settings above, `GET /test?a%20b=1%61+2` will yield:

```bash

 a b: 1a 2
```

Arguments without the `=<value>` parts are treated as boolean arguments. `GET /test?foo&bar` will yield:

```bash

 foo: true
 bar: true
```

That is, they will take Lua boolean values `true`. However, they are different from arguments taking empty string values. `GET /test?foo=&bar=` will give something like

```bash

 foo:
 bar:
```

Empty key arguments are discarded. `GET /test?=hello&=world` will yield an empty output for instance.

Updating query arguments via the nginx variable `$args` (or `ngx.var.args` in Lua) at runtime is also supported:

```lua

 ngx.var.args = "a=3&b=42"
 local args = ngx.req.get_uri_args()
```

Here the `args` table will always look like

```lua

 {a = 3, b = 42}
```

regardless of the actual request query string.

Note that a maximum of 100 request arguments are parsed by default (including those with the same name) and that additional request arguments are silently discarded to guard against potential denial of service attacks.

However, the optional `max_args` function argument can be used to override this limit:

```lua

 local args = ngx.req.get_uri_args(10)
```

This argument can be set to zero to remove the limit and to process all request arguments received:

```lua

 local args = ngx.req.get_uri_args(0)
```

Removing the `max_args` cap is strongly discouraged.

[回到目录](#nginx-api-for-lua)

ngx.req.get_post_args
---------------------
**syntax:** *args, err = ngx.req.get_post_args(max_args?)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

Returns a Lua table holding all the current request POST query arguments (of the MIME type `application/x-www-form-urlencoded`). Call [ngx.req.read_body](#ngxreqread_body) to read the request body first or turn on the [lua_need_request_body](#lua_need_request_body) directive to avoid errors.

```nginx

 location = /test {
     content_by_lua '
         ngx.req.read_body()
         local args, err = ngx.req.get_post_args()
         if not args then
             ngx.say("failed to get post args: ", err)
             return
         end
         for key, val in pairs(args) do
             if type(val) == "table" then
                 ngx.say(key, ": ", table.concat(val, ", "))
             else
                 ngx.say(key, ": ", val)
             end
         end
     ';
 }
```

Then

```bash

 # Post request with the body 'foo=bar&bar=baz&bar=blah'
 $ curl --data 'foo=bar&bar=baz&bar=blah' localhost/test
```

will yield the response body like

```bash

 foo: bar
 bar: baz, blah
```

Multiple occurrences of an argument key will result in a table value holding all of the values for that key in order.

Keys and values will be unescaped according to URI escaping rules.

With the settings above,

```bash

 # POST request with body 'a%20b=1%61+2'
 $ curl -d 'a%20b=1%61+2' localhost/test
```

will yield:

```bash

 a b: 1a 2
```

Arguments without the `=<value>` parts are treated as boolean arguments. `GET /test?foo&bar` will yield:

```bash

 foo: true
 bar: true
```

That is, they will take Lua boolean values `true`. However, they are different from arguments taking empty string values. `POST /test` with request body `foo=&bar=` will return something like

```bash

 foo:
 bar:
```

Empty key arguments are discarded. `POST /test` with body `=hello&=world` will yield empty outputs for instance.

Note that a maximum of 100 request arguments are parsed by default (including those with the same name) and that additional request arguments are silently discarded to guard against potential denial of service attacks.  

However, the optional `max_args` function argument can be used to override this limit:

```lua

 local args = ngx.req.get_post_args(10)
```

This argument can be set to zero to remove the limit and to process all request arguments received:

```lua

 local args = ngx.req.get_post_args(0)
```

Removing the `max_args` cap is strongly discouraged.

[回到目录](#nginx-api-for-lua)

ngx.req.get_headers
-------------------
**syntax:** *headers = ngx.req.get_headers(max_headers?, raw?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua, log_by_lua**

Returns a Lua table holding all the current request headers.

```lua

 local h = ngx.req.get_headers()
 for k, v in pairs(h) do
     ...
 end
```

To read an individual header:

```lua

 ngx.say("Host: ", ngx.req.get_headers()["Host"])
```

Note that the [ngx.var.HEADER](#ngxvarvariable) API call, which uses core [$http_HEADER](http://nginx.org/en/docs/http/ngx_http_core_module.html#var_http_) variables, may be more preferable for reading individual request headers.

For multiple instances of request headers such as:

```bash

 Foo: foo
 Foo: bar
 Foo: baz
```

the value of `ngx.req.get_headers()["Foo"]` will be a Lua (array) table such as:

```lua

 {"foo", "bar", "baz"}
```

Note that a maximum of 100 request headers are parsed by default (including those with the same name) and that additional request headers are silently discarded to guard against potential denial of service attacks.  

However, the optional `max_headers` function argument can be used to override this limit:

```lua

 local args = ngx.req.get_headers(10)
```

This argument can be set to zero to remove the limit and to process all request headers received:

```lua

 local args = ngx.req.get_headers(0)
```

Removing the `max_headers` cap is strongly discouraged.

Since the `0.6.9` release, all the header names in the Lua table returned are converted to the pure lower-case form by default, unless the `raw` argument is set to `true` (default to `false`).

Also, by default, an `__index` metamethod is added to the resulting Lua table and will normalize the keys to a pure lowercase form with all underscores converted to dashes in case of a lookup miss. For example, if a request header `My-Foo-Header` is present, then the following invocations will all pick up the value of this header correctly:

```lua

 ngx.say(headers.my_foo_header)
 ngx.say(headers["My-Foo-Header"])
 ngx.say(headers["my-foo-header"])
```

The `__index` metamethod will not be added when the `raw` argument is set to `true`.

[回到目录](#nginx-api-for-lua)

ngx.req.set_header
------------------
**syntax:** *ngx.req.set_header(header_name, header_value)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*

Set the current request's request header named `header_name` to value `header_value`, overriding any existing ones.

By default, all the subrequests subsequently initiated by [ngx.location.capture](#ngxlocationcapture) and [ngx.location.capture_multi](#ngxlocationcapture_multi) will inherit the new header.

Here is an example of setting the `Content-Type` header:

```lua

 ngx.req.set_header("Content-Type", "text/css")
```

The `header_value` can take an array list of values,
for example,

```lua

 ngx.req.set_header("Foo", {"a", "abc"})
```

will produce two new request headers:

```bash

 Foo: a
 Foo: abc
```

and old `Foo` headers will be overridden if there is any.

When the `header_value` argument is `nil`, the request header will be removed. So

```lua

 ngx.req.set_header("X-Foo", nil)
```

is equivalent to

```lua

 ngx.req.clear_header("X-Foo")
```

[回到目录](#nginx-api-for-lua)

ngx.req.clear_header
--------------------
**syntax:** *ngx.req.clear_header(header_name)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua**

Clears the current request's request header named `header_name`. None of the current request's existing subrequests will be affected but subsequently initiated subrequests will inherit the change by default.

[回到目录](#nginx-api-for-lua)

ngx.req.read_body
-----------------
**syntax:** *ngx.req.read_body()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Reads the client request body synchronously without blocking the Nginx event loop.

```lua

 ngx.req.read_body()
 local args = ngx.req.get_post_args()
```

If the request body is already read previously by turning on [lua_need_request_body](#lua_need_request_body) or by using other modules, then this function does not run and returns immediately.

If the request body has already been explicitly discarded, either by the [ngx.req.discard_body](#ngxreqdiscard_body) function or other modules, this function does not run and returns immediately.

In case of errors, such as connection errors while reading the data, this method will throw out a Lua exception *or* terminate the current request with a 500 status code immediately.

The request body data read using this function can be retrieved later via [ngx.req.get_body_data](#ngxreqget_body_data) or, alternatively, the temporary file name for the body data cached to disk using [ngx.req.get_body_file](#ngxreqget_body_file). This depends on

1. whether the current request body is already larger than the [client_body_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size),
1. and whether [client_body_in_file_only](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_in_file_only) has been switched on.

In cases where current request may have a request body and the request body data is not required, The [ngx.req.discard_body](#ngxreqdiscard_body) function must be used to explicitly discard the request body to avoid breaking things under HTTP 1.1 keepalive or HTTP 1.1 pipelining.

This function was first introduced in the `v0.3.1rc17` release.

[回到目录](#nginx-api-for-lua)

ngx.req.discard_body
--------------------
**syntax:** *ngx.req.discard_body()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Explicitly discard the request body, i.e., read the data on the connection and throw it away immediately. Please note that ignoring request body is not the right way to discard it, and that this function must be called to avoid breaking things under HTTP 1.1 keepalive or HTTP 1.1 pipelining.

This function is an asynchronous call and returns immediately.

If the request body has already been read, this function does nothing and returns immediately.

This function was first introduced in the `v0.3.1rc17` release.

See also [ngx.req.read_body](#ngxreqread_body).

[回到目录](#nginx-api-for-lua)

ngx.req.get_body_data
---------------------
**syntax:** *data = ngx.req.get_body_data()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Retrieves in-memory request body data. It returns a Lua string rather than a Lua table holding all the parsed query arguments. Use the [ngx.req.get_post_args](#ngxreqget_post_args) function instead if a Lua table is required.

This function returns `nil` if

1. the request body has not been read,
1. the request body has been read into disk temporary files,
1. or the request body has zero size.

If the request body has not been read yet, call [ngx.req.read_body](#ngxreqread_body) first (or turned on [lua_need_request_body](#lua_need_request_body) to force this module to read the request body. This is not recommended however).

If the request body has been read into disk files, try calling the [ngx.req.get_body_file](#ngxreqget_body_file) function instead.

To force in-memory request bodies, try setting [client_body_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) to the same size value in [client_max_body_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_max_body_size).

Note that calling this function instead of using `ngx.var.request_body` or `ngx.var.echo_request_body` is more efficient because it can save one dynamic memory allocation and one data copy.

This function was first introduced in the `v0.3.1rc17` release.

See also [ngx.req.get_body_file](#ngxreqget_body_file).

[回到目录](#nginx-api-for-lua)

ngx.req.get_body_file
---------------------
**syntax:** *file_name = ngx.req.get_body_file()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Retrieves the file name for the in-file request body data. Returns `nil` if the request body has not been read or has been read into memory.

The returned file is read only and is usually cleaned up by Nginx's memory pool. It should not be manually modified, renamed, or removed in Lua code.

If the request body has not been read yet, call [ngx.req.read_body](#ngxreqread_body) first (or turned on [lua_need_request_body](#lua_need_request_body) to force this module to read the request body. This is not recommended however).

If the request body has been read into memory, try calling the [ngx.req.get_body_data](#ngxreqget_body_data) function instead.

To force in-file request bodies, try turning on [client_body_in_file_only](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_in_file_only).

This function was first introduced in the `v0.3.1rc17` release.

See also [ngx.req.get_body_data](#ngxreqget_body_data).

[回到目录](#nginx-api-for-lua)

ngx.req.set_body_data
---------------------
**syntax:** *ngx.req.set_body_data(data)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Set the current request's request body using the in-memory data specified by the `data` argument.

If the current request's request body has not been read, then it will be properly discarded. When the current request's request body has been read into memory or buffered into a disk file, then the old request body's memory will be freed or the disk file will be cleaned up immediately, respectively.

This function was first introduced in the `v0.3.1rc18` release.

See also [ngx.req.set_body_file](#ngxreqset_body_file).

[回到目录](#nginx-api-for-lua)

ngx.req.set_body_file
---------------------
**syntax:** *ngx.req.set_body_file(file_name, auto_clean?)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Set the current request's request body using the in-file data specified by the `file_name` argument.

If the optional `auto_clean` argument is given a `true` value, then this file will be removed at request completion or the next time this function or [ngx.req.set_body_data](#ngxreqset_body_data) are called in the same request. The `auto_clean` is default to `false`.

Please ensure that the file specified by the `file_name` argument exists and is readable by an Nginx worker process by setting its permission properly to avoid Lua exception errors.

If the current request's request body has not been read, then it will be properly discarded. When the current request's request body has been read into memory or buffered into a disk file, then the old request body's memory will be freed or the disk file will be cleaned up immediately, respectively.

This function was first introduced in the `v0.3.1rc18` release.

See also [ngx.req.set_body_data](#ngxreqset_body_data).

[回到目录](#nginx-api-for-lua)

ngx.req.init_body
-----------------
**syntax:** *ngx.req.init_body(buffer_size?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

Creates a new blank request body for the current request and inializes the buffer for later request body data writing via the [ngx.req.append_body](#ngxreqappend_body) and [ngx.req.finish_body](#ngxreqfinish_body) APIs.

If the `buffer_size` argument is specified, then its value will be used for the size of the memory buffer for body writing with [ngx.req.append_body](#ngxreqappend_body). If the argument is omitted, then the value specified by the standard [client_body_buffer_size](http://nginx.org/en/docs/http/ngx_http_core_module.html#client_body_buffer_size) directive will be used instead.

When the data can no longer be hold in the memory buffer for the request body, then the data will be flushed onto a temporary file just like the standard request body reader in the Nginx core.

It is important to always call the [ngx.req.finish_body](#ngxreqfinish_body) after all the data has been appended onto the current request body. Also, when this function is used together with [ngx.req.socket](#ngxreqsocket), it is required to call [ngx.req.socket](#ngxreqsocket) *before* this function, or you will get the "request body already exists" error message.

The usage of this function is often like this:

```lua

 ngx.req.init_body(128 * 1024)  -- buffer is 128KB
 for chunk in next_data_chunk() do
     ngx.req.append_body(chunk) -- each chunk can be 4KB
 end
 ngx.req.finish_body()
```

This function can be used with [ngx.req.append_body](#ngxreqappend_body), [ngx.req.finish_body](#ngxreqfinish_body), and [ngx.req.socket](#ngxreqsocket) to implement efficient input filters in pure Lua (in the context of [rewrite_by_lua](#rewrite_by_lua)* or [access_by_lua](#access_by_lua)*), which can be used with other Nginx content handler or upstream modules like [ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) and [ngx_http_fastcgi_module](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html).

This function was first introduced in the `v0.5.11` release.

[回到目录](#nginx-api-for-lua)

ngx.req.append_body
-------------------
**syntax:** *ngx.req.append_body(data_chunk)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

Append new data chunk specified by the `data_chunk` argument onto the existing request body created by the [ngx.req.init_body](#ngxreqinit_body) call.

When the data can no longer be hold in the memory buffer for the request body, then the data will be flushed onto a temporary file just like the standard request body reader in the Nginx core.

It is important to always call the [ngx.req.finish_body](#ngxreqfinish_body) after all the data has been appended onto the current request body.

This function can be used with [ngx.req.init_body](#ngxreqinit_body), [ngx.req.finish_body](#ngxreqfinish_body), and [ngx.req.socket](#ngxreqsocket) to implement efficient input filters in pure Lua (in the context of [rewrite_by_lua](#rewrite_by_lua)* or [access_by_lua](#access_by_lua)*), which can be used with other Nginx content handler or upstream modules like [ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) and [ngx_http_fastcgi_module](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html).

This function was first introduced in the `v0.5.11` release.

See also [ngx.req.init_body](#ngxreqinit_body).

[回到目录](#nginx-api-for-lua)

ngx.req.finish_body
-------------------
**syntax:** *ngx.req.finish_body()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

Completes the construction process of the new request body created by the [ngx.req.init_body](#ngxreqinit_body) and [ngx.req.append_body](#ngxreqappend_body) calls.

This function can be used with [ngx.req.init_body](#ngxreqinit_body), [ngx.req.append_body](#ngxreqappend_body), and [ngx.req.socket](#ngxreqsocket) to implement efficient input filters in pure Lua (in the context of [rewrite_by_lua](#rewrite_by_lua)* or [access_by_lua](#access_by_lua)*), which can be used with other Nginx content handler or upstream modules like [ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html) and [ngx_http_fastcgi_module](http://nginx.org/en/docs/http/ngx_http_fastcgi_module.html).

This function was first introduced in the `v0.5.11` release.

See also [ngx.req.init_body](#ngxreqinit_body).

[回到目录](#nginx-api-for-lua)

ngx.req.socket
--------------
**syntax:** *tcpsock, err = ngx.req.socket()*

**syntax:** *tcpsock, err = ngx.req.socket(raw)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Returns a read-only cosocket object that wraps the downstream connection. Only [receive](#tcpsockreceive) and [receiveuntil](#tcpsockreceiveuntil) methods are supported on this object.

In case of error, `nil` will be returned as well as a string describing the error.

The socket object returned by this method is usually used to read the current request's body in a streaming fashion. Do not turn on the [lua_need_request_body](#lua_need_request_body) directive, and do not mix this call with [ngx.req.read_body](#ngxreqread_body) and [ngx.req.discard_body](#ngxreqdiscard_body).

If any request body data has been pre-read into the Nginx core request header buffer, the resulting cosocket object will take care of this to avoid potential data loss resulting from such pre-reading.
Chunked request bodies are not yet supported in this API.

Since the `v0.9.0` release, this function accepts an optional boolean `raw` argument. When this argument is `true`, this function returns a full-duplex cosocket object wrapping around the raw downstream connection socket, upon which you can call the [receive](#tcpsockreceive), [receiveuntil](#tcpsockreceiveuntil), and [send](#tcpsocksend) methods.

When the `raw` argument is `true`, it is required that no pending data from any previous [ngx.say](#ngxsay), [ngx.print](#ngxprint), or [ngx.send_headers](#ngxsend_headers) calls exists. So if you have these downstream output calls previously, you should call [ngx.flush(true)](#ngxflush) before calling `ngx.req.socket(true)` to ensure that there is no pending output data. If the request body has not been read yet, then this "raw socket" can also be used to read the request body.

You can use the "raw request socket" returned by `ngx.req.socket(true)` to implement fancy protocols like [WebSocket](http://en.wikipedia.org/wiki/WebSocket), or just emit your own raw HTTP response header or body data. You can refer to the [lua-resty-websocket library](https://github.com/openresty/lua-resty-websocket) for a real world example.

This function was first introduced in the `v0.5.0rc1` release.

[回到目录](#nginx-api-for-lua)

ngx.exec
--------
**syntax:** *ngx.exec(uri, args?)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Does an internal redirect to `uri` with `args` and is similar to the [echo_exec](http://github.com/openresty/echo-nginx-module#echo_exec) directive of the [echo-nginx-module](http://github.com/openresty/echo-nginx-module).

```lua

 ngx.exec('/some-location');
 ngx.exec('/some-location', 'a=3&b=5&c=6');
 ngx.exec('/some-location?a=3&b=5', 'c=6');
```

The optional second `args` can be used to specify extra URI query arguments, for example:

```lua

 ngx.exec("/foo", "a=3&b=hello%20world")
```

Alternatively, a Lua table can be passed for the `args` argument for ngx_lua to carry out URI escaping and string concatenation.

```lua

 ngx.exec("/foo", { a = 3, b = "hello world" })
```

The result is exactly the same as the previous example.

The format for the Lua table passed as the `args` argument is identical to the format used in the [ngx.encode_args](#ngxencode_args) method.

Named locations are also supported but the second `args` argument will be ignored if present and the querystring for the new target is inherited from the referring location (if any).

`GET /foo/file.php?a=hello` will return "hello" and not "goodbye" in the example below

```nginx

 location /foo {
     content_by_lua '
         ngx.exec("@bar", "a=goodbye");
     ';
 }

 location @bar {
     content_by_lua '
         local args = ngx.req.get_uri_args()
         for key, val in pairs(args) do
             if key == "a" then
                 ngx.say(val)
             end
         end
     ';
 }
```

Note that the `ngx.exec` method is different from [ngx.redirect](#ngxredirect) in that
it is purely an internal redirect and that no new external HTTP traffic is involved.

Also note that this method call terminates the processing of the current request and that it *must* be called before [ngx.send_headers](#ngxsend_headers) or explicit response body
outputs by either [ngx.print](#ngxprint) or [ngx.say](#ngxsay).

It is recommended that a coding style that combines this method call with the `return` statement, i.e., `return ngx.exec(...)` be adopted when this method call is used in contexts other than [header_filter_by_lua](#header_filter_by_lua) to reinforce the fact that the request processing is being terminated.

[回到目录](#nginx-api-for-lua)

ngx.redirect
------------
**语法:** *ngx.redirect(uri, status?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

发起`HTTP 301`或者`302`重定向到指定`uri`。

可选参数`status`指定`301`或`302`。默认情况下是`302`（`ngx.HTTP_MOVED_TEMPORARILY`）。

下面的例子假设当前服务器名字为`localhost`，监听端口1984：

```lua

 return ngx.redirect("/foo")
```

等价于

```lua

 return ngx.redirect("/foo", ngx.HTTP_MOVED_TEMPORARILY)
```

也可以重定向到任意外部URL，例如：

```lua

 return ngx.redirect("http://www.google.com")
```

第二个参数`status`也可以直接用数字代替：

```lua

 return ngx.redirect("/foo", 301)
```

该方法与带有`redirect`修饰符的[rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite)指令十分类似，详情看[ngx_http_rewrite_module](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html).例如，下面nginx.conf的配置：

```nginx

 rewrite ^ /foo? redirect;  # nginx config
```

等价于下面的Lua代码：

```lua

 return ngx.redirect('/foo');  -- Lua code
```

同样，

```nginx

 rewrite ^ /foo? permanent;  # nginx config
```

等价于

```lua

 return ngx.redirect('/foo', ngx.HTTP_MOVED_PERMANENTLY)  -- Lua code
```

也可以指定URI参数，例如：

```lua

 return ngx.redirect('/foo?a=3&b=4')
```

注意，该方法调用会终止当前请求的执行，所以你必须在调用[ngx.send_headers](#ngxsend_headers)或显式发送响应体的函数[ngx.print](#ngxprint)和[ngx.say](#ngxsay)之前被调用。

建议的编码风格是该方法调用总是和`return`语句一起使用，例如，`return ngx.redirect(...)`，用以加强请求处理终止的事实。一个例外是，不要在[header_filter_by_lua](#header_filter_by_lua)上下文中这么做。

[回到目录](#nginx-api-for-lua)

ngx.send_headers
----------------
**syntax:** *ok, err = ngx.send_headers()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

Explicitly send out the response headers.

Since `v0.8.3` this function returns `1` on success, or returns `nil` and a string describing the error otherwise.

Note that there is normally no need to manually send out response headers as ngx_lua will automatically send headers out
before content is output with [ngx.say](#ngxsay) or [ngx.print](#ngxprint) or when [content_by_lua](#content_by_lua) exits normally.

[回到目录](#nginx-api-for-lua)

ngx.headers_sent
----------------
**syntax:** *value = ngx.headers_sent*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua**

Returns `true` if the response headers have been sent (by ngx_lua), and `false` otherwise.

This API was first introduced in ngx_lua v0.3.1rc6.

[回到目录](#nginx-api-for-lua)

ngx.print
---------
**语法:** *ok, err = ngx.print(...)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

拼接参数发送给HTTP客户端（作为包体）。如果响应头尚且没有发送，该函数会先发送头部，然后发送包体数据。

从`v0.8.3`版本开始，该函数成功返回`1`，否则返回`nil`和一个错误描述字符串。

Lua的`nil`值会输出`"nil"`字符串，Lua布尔值会输出`"true"`和`"false"`字符串。

嵌套字符串数组也是允许的，数组的元素会逐个发送：

```lua

 local table = {
     "hello, ",
     {"world: ", true, " or ", false,
         {": ", nil}}
 }
 ngx.print(table)
```

输出：

```bash

 hello, world: true or false: nil
```

非数组table参数会抛出Lua异常。

`ngx.null`常量输出`"null"`字符串。

这是一个异步调用，不会等待所有数据都写入系统的发送缓冲区，而是立即返回。要以同步的方式允许，可以在`ngx.print`之后调用`ngx.flush(true)`。这对于流式输出非常有用。更多细节请看[ngx.flush](#ngxflush)。

注意，`ngx.print`和[ngx.say](#ngxsay)总是会唤醒整个Nginx输出包体过滤链，而这是代价昂贵的操作。所以，如果要在循环中调用这两个函数需要格外小心，自个儿在Lua中缓存数据，节省调用。

[回到目录](#nginx-api-for-lua)

ngx.say
-------
**语法:** *ok, err = ngx.say(...)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

与[ngx.print](#ngxprint)类似，但也发送换行符。

[回到目录](#nginx-api-for-lua)

ngx.log
-------
**syntax:** *ngx.log(log_level, ...)*

**context:** *init_by_lua*, init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Log arguments concatenated to error.log with the given logging level.

Lua `nil` arguments are accepted and result in literal `"nil"` string while Lua booleans result in literal `"true"` or `"false"` string outputs. And the `ngx.null` constant will yield the `"null"` string output.

The `log_level` argument can take constants like `ngx.ERR` and `ngx.WARN`. Check out [Nginx log level constants](#nginx-log-level-constants) for details.

There is a hard coded `2048` byte limitation on error message lengths in the Nginx core. This limit includes trailing newlines and leading time stamps. If the message size exceeds this limit, Nginx will truncate the message text accordingly. This limit can be manually modified by editing the `NGX_MAX_ERROR_STR` macro definition in the `src/core/ngx_log.h` file in the Nginx source tree.

[回到目录](#nginx-api-for-lua)

ngx.flush
---------
**语法:** *ok, err = ngx.flush(wait?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

强制发送响应给客户端。

`ngx.flush`接受一个可选布尔参数`wait`（默认为false）。当带有默认参数调用该函数时，将发起一个异步调用（不等待数据写入系统发送缓存，立即返回）。当带有参数`wait`设为true调用该函数时，将转为同步模式。 

在同步模式下，除非所有的数据都已经写入系统发送缓冲区或者[send_timeout](http://nginx.org/en/docs/http/ngx_http_core_module.html#send_timeout)超时到达，该函数一直不会返回。注意，使用lua的协程机制意味着在同步模式下该函数不会阻塞nginx事件循环。

当`ngx.flush(true)`在[ngx.print](#ngxprint)或者[ngx.say](#ngxsay)立即调用时，会使之后的函数运行在同步模式。这在流输出时非常有用。

但值得注意的是，`ngx.flush`在HTTP 1.0的output buffering模式下不起作用，详情见[HTTP 1.0支持](#http-10-support)。

从`v0.8.3`起，该函数成功返回`1`，否则返回`nil`和一个描述错误的字符串。

[回到目录](#nginx-api-for-lua)

ngx.exit
--------
**语法:** *ngx.exit(status)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, ngx.timer.**

当`status >= 200`（即`ngx.HTTP_OK`或更大）时，将中断当前请求的执行，并向nginx返回状态码。

当`status == 0`（即`ngx.OK`）时，只是退出当前阶段的回调函数，会继续执行当前请求的下一阶段的回调函数。

参数`status`可以取值`ngx.OK`，`ngx.ERROR`，`ngx.HTTP_NOT_FOUND`，`ngx.HTTP_MOVED_TEMPORARILY`，或者其他[HTTP状态常量](#http-status-constants)。

要返回带有自定义内容的错误页，可以像下面这样书写代码：

```lua

 ngx.status = ngx.HTTP_GONE
 ngx.say("This is our own content")
 -- 退出整个请求的执行
 ngx.exit(ngx.HTTP_OK)
```

执行效果：

```bash

 $ curl -i http://localhost/test
 HTTP/1.1 410 Gone
 Server: nginx/1.0.6
 Date: Thu, 15 Sep 2011 00:51:48 GMT
 Content-Type: text/plain
 Transfer-Encoding: chunked
 Connection: keep-alive

 This is our own content
```

数字也可以直接用于状态参数，例如

```lua

 ngx.exit(501)
```

注意，虽然这个方法接受所有的[HTTP状态常量](#http-status-constants)作为输入，但它只接受`NGX_OK`和`NGX_ERROR`两个[核心常量](#core-constants)。

还有，因为该函数可以终止当前请求的处理，所以建议的编程风格是该函数与`return`语句连同使用，如`return ngx.exit(...)`用来强化当前请求结束的事实。

当该函数用在[header_filter_by_lua](#header_filter_by_lua)上下文时，`ngx.exit()`为异步操作，会立即返回。这种行为在未来可能会改变，因此建议用户总是将其与`return`一起使用。

[回到目录](#nginx-api-for-lua)

ngx.eof
-------
**语法:** *ok, err = ngx.eof()*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

显示标示响应流的结束。在HTTP 1.1协议中的chunked编码输出时，该函数会触发Nginx内核发送最后一个chunk。

如果你禁用了下游连接HTTP 1.1协议的keep-alive功能，你可以用此方法使下游客户主动关闭连接。这种技术可以用于后台任务，而不必使HTTP客户在连接上等待，就像下面这样：

```nginx

 location = /async {
     keepalive_timeout 0;
     content_by_lua '
         ngx.say("got the task!")
         ngx.eof()  -- 此处HTTP客户端会关闭连接
         -- 在此处访问MySQL, PostgreSQL, Redis, Memcached等
     ';
 }
```

但是，如果你创建子请求访问Nginx上游模块配置的其他location，你需要配置这些上游模块忽略客户端连接退出。例如，默认情况下，客户端关闭连接，标准[ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)会终止子请求和主请求的处理。所以，请在你的[ngx_http_proxy_module](http://nginx.org/en/docs/http/ngx_http_proxy_module.html)配置的location中打开[proxy_ignore_client_abort](http://nginx.org/en/docs/http/ngx_http_proxy_module.html#proxy_ignore_client_abort)指令：

```nginx

 proxy_ignore_client_abort on;
```

更好的一种执行后台任务的做法是使用[ngx.timer.at](#ngxtimerat) API。

从`v0.8.3`版本后，该函数成功返回`1`，否则返回`nil`和一个描述错误的字符串。

[回到目录](#nginx-api-for-lua)

ngx.sleep
---------
**语法:** *ngx.sleep(seconds)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

非阻塞地睡眠指定秒数。可以指定的时间精度为0.001秒。

该方法底层实现使用了Nginx定时器。

从`0.7.20`版本开始，时间参数也可以指定为`0`。

该方法最早出现在`0.5.0rc30`版本。

[回到目录](#nginx-api-for-lua)

ngx.escape_uri
--------------
**syntax:** *newstr = ngx.escape_uri(str)*

**context:** *init_by_lua*, init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Escape `str` as a URI component.

[回到目录](#nginx-api-for-lua)

ngx.unescape_uri
----------------
**syntax:** *newstr = ngx.unescape_uri(str)*

**context:** *init_by_lua*, init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Unescape `str` as an escaped URI component.

For example,

```lua

 ngx.say(ngx.unescape_uri("b%20r56+7"))
```

gives the output


    b r56 7


[回到目录](#nginx-api-for-lua)

ngx.encode_args
---------------
**syntax:** *str = ngx.encode_args(table)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Encode the Lua table to a query args string according to the URI encoded rules.

For example,

```lua

 ngx.encode_args({foo = 3, ["b r"] = "hello world"})
```

yields


    foo=3&b%20r=hello%20world


The table keys must be Lua strings.

Multi-value query args are also supported. Just use a Lua table for the argument's value, for example:

```lua

 ngx.encode_args({baz = {32, "hello"}})
```

gives


    baz=32&baz=hello


If the value table is empty and the effect is equivalent to the `nil` value.

Boolean argument values are also supported, for instance,

```lua

 ngx.encode_args({a = true, b = 1})
```

yields


    a&b=1


If the argument value is `false`, then the effect is equivalent to the `nil` value.

This method was first introduced in the `v0.3.1rc27` release.

[回到目录](#nginx-api-for-lua)

ngx.decode_args
---------------
**syntax:** *table = ngx.decode_args(str, max_args?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Decodes a URI encoded query-string into a Lua table. This is the inverse function of [ngx.encode_args](#ngxencode_args).

The optional `max_args` argument can be used to specify the maximum number of arguments parsed from the `str` argument. By default, a maximum of 100 request arguments are parsed (including those with the same name) and that additional URI arguments are silently discarded to guard against potential denial of service attacks.

This argument can be set to zero to remove the limit and to process all request arguments received:

```lua

 local args = ngx.decode_args(str, 0)
```

Removing the `max_args` cap is strongly discouraged.

This method was introduced in the `v0.5.0rc29`.

[回到目录](#nginx-api-for-lua)

ngx.encode_base64
-----------------
**syntax:** *newstr = ngx.encode_base64(str, no_padding?)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Encodes `str` to a base64 digest.

Since the `0.9.16` release, an optional boolean-typed `no_padding` argument can be specified to control whether the base64 padding should be appended to the resulting digest (default to `false`, i.e., with padding enabled). This enables streaming base64 digest calculation by (data chunks) though it would be the caller's responsibility to append an appropriate padding at the end of data stream.

[回到目录](#nginx-api-for-lua)

ngx.decode_base64
-----------------
**syntax:** *newstr = ngx.decode_base64(str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Decodes the `str` argument as a base64 digest to the raw form. Returns `nil` if `str` is not well formed.

[回到目录](#nginx-api-for-lua)

ngx.crc32_short
---------------
**syntax:** *intval = ngx.crc32_short(str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Calculates the CRC-32 (Cyclic Redundancy Code) digest for the `str` argument.

This method performs better on relatively short `str` inputs (i.e., less than 30 ~ 60 bytes), as compared to [ngx.crc32_long](#ngxcrc32_long). The result is exactly the same as [ngx.crc32_long](#ngxcrc32_long).

Behind the scene, it is just a thin wrapper around the `ngx_crc32_short` function defined in the Nginx core.

This API was first introduced in the `v0.3.1rc8` release.

[回到目录](#nginx-api-for-lua)

ngx.crc32_long
--------------
**syntax:** *intval = ngx.crc32_long(str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Calculates the CRC-32 (Cyclic Redundancy Code) digest for the `str` argument.

This method performs better on relatively long `str` inputs (i.e., longer than 30 ~ 60 bytes), as compared to [ngx.crc32_short](#ngxcrc32_short).  The result is exactly the same as [ngx.crc32_short](#ngxcrc32_short).

Behind the scene, it is just a thin wrapper around the `ngx_crc32_long` function defined in the Nginx core.

This API was first introduced in the `v0.3.1rc8` release.

[回到目录](#nginx-api-for-lua)

ngx.hmac_sha1
-------------
**syntax:** *digest = ngx.hmac_sha1(secret_key, str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Computes the [HMAC-SHA1](http://en.wikipedia.org/wiki/HMAC) digest of the argument `str` and turns the result using the secret key `<secret_key>`.

The raw binary form of the `HMAC-SHA1` digest will be generated, use [ngx.encode_base64](#ngxencode_base64), for example, to encode the result to a textual representation if desired.

For example,

```lua

 local key = "thisisverysecretstuff"
 local src = "some string we want to sign"
 local digest = ngx.hmac_sha1(key, src)
 ngx.say(ngx.encode_base64(digest))
```

yields the output


    R/pvxzHC4NLtj7S+kXFg/NePTmk=


This API requires the OpenSSL library enabled in the Nginx build (usually by passing the `--with-http_ssl_module` option to the `./configure` script).

This function was first introduced in the `v0.3.1rc29` release.

[回到目录](#nginx-api-for-lua)

ngx.md5
-------
**syntax:** *digest = ngx.md5(str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns the hexadecimal representation of the MD5 digest of the `str` argument.

For example,

```nginx

 location = /md5 {
     content_by_lua 'ngx.say(ngx.md5("hello"))';
 }
```

yields the output


    5d41402abc4b2a76b9719d911017c592


See [ngx.md5_bin](#ngxmd5_bin) if the raw binary MD5 digest is required.

[回到目录](#nginx-api-for-lua)

ngx.md5_bin
-----------
**syntax:** *digest = ngx.md5_bin(str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns the binary form of the MD5 digest of the `str` argument.

See [ngx.md5](#ngxmd5) if the hexadecimal form of the MD5 digest is required.

[回到目录](#nginx-api-for-lua)

ngx.sha1_bin
------------
**syntax:** *digest = ngx.sha1_bin(str)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns the binary form of the SHA-1 digest of the `str` argument.

This function requires SHA-1 support in the Nginx build. (This usually just means OpenSSL should be installed while building Nginx).

This function was first introduced in the `v0.5.0rc6`.

[回到目录](#nginx-api-for-lua)

ngx.quote_sql_str
-----------------
**syntax:** *quoted_value = ngx.quote_sql_str(raw_value)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns a quoted SQL string literal according to the MySQL quoting rules.

[回到目录](#nginx-api-for-lua)

ngx.today
---------
**syntax:** *str = ngx.today()*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns current date (in the format `yyyy-mm-dd`) from the nginx cached time (no syscall involved unlike Lua's date library).

This is the local time.

[回到目录](#nginx-api-for-lua)

ngx.time
--------
**syntax:** *secs = ngx.time()*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns the elapsed seconds from the epoch for the current time stamp from the nginx cached time (no syscall involved unlike Lua's date library).

Updates of the Nginx time cache an be forced by calling [ngx.update_time](#ngxupdate_time) first.

[回到目录](#nginx-api-for-lua)

ngx.now
-------
**syntax:** *secs = ngx.now()*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns a floating-point number for the elapsed time in seconds (including milliseconds as the decimal part) from the epoch for the current time stamp from the nginx cached time (no syscall involved unlike Lua's date library).

You can forcibly update the Nginx time cache by calling [ngx.update_time](#ngxupdate_time) first.

This API was first introduced in `v0.3.1rc32`.

[回到目录](#nginx-api-for-lua)

ngx.update_time
---------------
**syntax:** *ngx.update_time()*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Forcibly updates the Nginx current time cache. This call involves a syscall and thus has some overhead, so do not abuse it.

This API was first introduced in `v0.3.1rc32`.

[回到目录](#nginx-api-for-lua)

ngx.localtime
-------------
**syntax:** *str = ngx.localtime()*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns the current time stamp (in the format `yyyy-mm-dd hh:mm:ss`) of the nginx cached time (no syscall involved unlike Lua's [os.date](http://www.lua.org/manual/5.1/manual.html#pdf-os.date) function).

This is the local time.

[回到目录](#nginx-api-for-lua)

ngx.utctime
-----------
**syntax:** *str = ngx.utctime()*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns the current time stamp (in the format `yyyy-mm-dd hh:mm:ss`) of the nginx cached time (no syscall involved unlike Lua's [os.date](http://www.lua.org/manual/5.1/manual.html#pdf-os.date) function).

This is the UTC time.

[回到目录](#nginx-api-for-lua)

ngx.cookie_time
---------------
**syntax:** *str = ngx.cookie_time(sec)*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns a formatted string can be used as the cookie expiration time. The parameter `sec` is the time stamp in seconds (like those returned from [ngx.time](#ngxtime)).

```nginx

 ngx.say(ngx.cookie_time(1290079655))
     -- yields "Thu, 18-Nov-10 11:27:35 GMT"
```

[回到目录](#nginx-api-for-lua)

ngx.http_time
-------------
**syntax:** *str = ngx.http_time(sec)*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Returns a formated string can be used as the http header time (for example, being used in `Last-Modified` header). The parameter `sec` is the time stamp in seconds (like those returned from [ngx.time](#ngxtime)).

```nginx

 ngx.say(ngx.http_time(1290079655))
     -- yields "Thu, 18 Nov 2010 11:27:35 GMT"
```

[回到目录](#nginx-api-for-lua)

ngx.parse_http_time
-------------------
**syntax:** *sec = ngx.parse_http_time(str)*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Parse the http time string (as returned by [ngx.http_time](#ngxhttp_time)) into seconds. Returns the seconds or `nil` if the input string is in bad forms.

```nginx

 local time = ngx.parse_http_time("Thu, 18 Nov 2010 11:27:35 GMT")
 if time == nil then
     ...
 end
```

[回到目录](#nginx-api-for-lua)

ngx.is_subrequest
-----------------
**syntax:** *value = ngx.is_subrequest*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua**

Returns `true` if the current request is an nginx subrequest, or `false` otherwise.

[回到目录](#nginx-api-for-lua)

ngx.re.match
------------
**语法:** *captures, err = ngx.re.match(subject, regex, options?, ctx?, res_table?)*

**上下文:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

根据`options`，使用PCRE `regex`正则匹配目标字符串`subject`。

只有返回第一个被匹配的串，或者没有匹配串返回`nil`。如果遇到错误，比如给出错误的正则表达式或超出PCRE栈限制等，返回`nil`和错误描述字符串。

当有匹配找到，返回Lua table类型`captures`，其中，`capture[0]`包含整个匹配的子串，`capture[1]`包含第一个括号括起来的匹配捕获串，`capture[2]`第二个，依次类推。

```lua

 local m, err = ngx.re.match("hello, 1234", "[0-9]+")
 if m then
     -- 匹配
     -- m[0] == "1234"

 else
     if err then
         -- 遇到错误
         ngx.log(ngx.ERR, "error: ", err)
         return
     end

     -- 没有匹配
     ngx.say("match not found")
 end
```

```lua

 local m, err = ngx.re.match("hello, 1234", "([0-9])[0-9]+")
 -- m[0] == "1234"
 -- m[1] == "1"
```

从`v0.7.14`版本开始支持具名捕获，同样以key-value的形式返回相同的Lua table。

```lua

 local m, err = ngx.re.match("hello, 1234", "([0-9])(?<remaining>[0-9]+)")
 -- m[0] == "1234"
 -- m[1] == "1"
 -- m[2] == "234"
 -- m["remaining"] == "234"
```

未匹配的子模式会在对应的capture表中返回`nil`。

```lua

 local m, err = ngx.re.match("hello, world", "(world)|(hello)|(?<named>howdy)")
 -- m[0] == "hello"
 -- m[1] == nil
 -- m[2] == "hello"
 -- m[3] == nil
 -- m["named"] == nil
```

`options`用于控制如何执行匹配操作。支持以下选项：


    a             锚定模式（仅从头匹配）

    d             开启DFA模式（或最长token匹配语义），
                  这要求PCRE6.0+，否则抛出Lua异常。首次出现在v0.3.1rc30

    D             开启可重复具名模式。这允许具名模式可以重复使用，
                  在Lua table中返回捕获。例如：
                    local m = ngx.re.match("hello, world",
                                           "(?<named>\w+), (?<named>\w+)",
                                           "D")
                    -- m["named"] == {"hello", "world"}
                  该选项首次出现在v0.7.14版本。该选项要求PCRE在8.12以上

    i             大小写不敏感模式（类似于Perl中的/i修饰符）

    j             开启PCRE JIT编译，该选项要求PCRE在8.21以上，并且使用
                  --enable-jit创建。为了性能考虑，该选项应该总是和'o'选项一起使用。该选项最早出现在ngx_lua v0.3.1rc30中

    J             开启PCRE js兼容模式。该选项最早出现在v0.7.14版本，
                  并且要求PCRE在8.12以上

    m             多行模式（类似于Perl中的/m修饰符）

    o             一次编译模式（类似于Perl中的/o修饰符），
                  开启工作进程级别的正则编译缓存

    s             单行模式（类似于Perl中的/s修饰符）

    u             UTF-8模式。该选项要求PCRE库使用--enable-utf8选项创建，
                  否则会抛出Lua异常

    U             类似于‘u'选项，但禁用目标字符串PCRE的UTF-8有效性检查。
                  该选项最早出现在ngx_lua v0.8.1版本

    x             扩展模式（类似于Perl中的/x修饰符）


这些选项可以联合使用：

```nginx

 local m, err = ngx.re.match("hello, world", "HEL LO", "ix")
 -- m[0] == "hello"
```

```nginx

 local m, err = ngx.re.match("hello, 美好生活", "HELLO, (.{2})", "iu")
 -- m[0] == "hello, 美好"
 -- m[1] == "美好"
```

`o`选项对于性能调优非常有用，因为正则匹配模式只会被编译一次，并缓存在工作进程中，并在该工作进程中的所有请求共享。可以通过[lua_regex_cache_max_entries](#lua_regex_cache_max_entries)指令对正则缓存的上限进行调整。

第四个可选参数`ctx`可以是包含可选域`pos`的Lua table。如果`ctx`表中指定了`pos`域，`ngx.re.match`会从指定的offset（起点为1）开始匹配。如果没有指定`pos`，`ngx.re.match`总是在匹配成功时在匹配子串之后设定`pos`位置。如果匹配失败，`ctx`表保持不变。

```lua

 local ctx = {}
 local m, err = ngx.re.match("1234, hello", "[0-9]+", "", ctx)
      -- m[0] = "1234"
      -- ctx.pos == 5
```

```lua

 local ctx = { pos = 2 }
 local m, err = ngx.re.match("1234, hello", "[0-9]+", "", ctx)
      -- m[0] = "34"
      -- ctx.pos == 5
```

`ctx`表参数和`a`正则修饰符一起使用可以用于创建一个Lexer。

注意，当给定`ctx`参数时，`options`参数不再是可选的，并且如果不想给定有意义的选项参数，需要指定一个空字符串（`""`）作为占位符。

该方法要求Nginx开启PCRE库。([Known Issue With Special Escaping Sequences](#special-escaping-sequences)).

为了确认开启PCRE JIT，在`./configure`脚本中添加`--with-debug`选项来开启Nginx debug日志。然后，在`error_log`指令中配置”debug“级别错误日志。如果PCRE JIT开启，错误日志中会有以下信息输出：


    pcre JIT compiling result: 1


从`0.9.4`版本开始，该函数也接受第五个参数`res_table`，用于给调用者提供一个存储所有捕获结果的Lua table。从`0.9.6`版本开始，调用者负责保证该表是空的。这对于重用Lua表和避免GC和表分配负担过重非常有用。

该功能最早出现在`v0.2.1rc11`版本中。

[回到目录](#nginx-api-for-lua)

ngx.re.find
-----------
**语法:** *from, to, err = ngx.re.find(subject, regex, options?, ctx?, nth?)*

**上下文:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

该函数与[ngx.re.match](#ngxrematch)类似，但只返回匹配子串的起始索引(`from`)和结束索引(`end`)。返回的索引从1开始，可以直接用于[string.sub](http://www.lua.org/manual/5.1/manual.html#pdf-string.sub)来获取匹配的子字符串。

如果发生错误（无效正则表达式或PCRE运行时错误），该函数返回两个`nil`和一个错误描述字符串。

如果没有发现匹配，该函数仅返回一个`nil`。

例如下面的一个例子：

```lua

 local s = "hello, 1234"
 local from, to, err = ngx.re.find(s, "([0-9]+)", "jo")
 if from then
     ngx.say("from: ", from)
     ngx.say("to: ", to)
     ngx.say("matched: ", string.sub(s, from, to))
 else
     if err then
         ngx.say("error: ", err)
         return
     end
     ngx.say("not matched!")
 end
```

产生的输出为：

    from: 8
    to: 11
    matched: 1234

由于该函数不创建新的Lua字符串和新的Lua table，因此该函数比[ngx.re.match](#ngxrematch)更高效。任何时候都应该尽量使用该函数。

从`0.9.3`版本开始，第五个可选参数`nth`可以用于指定返回第几个捕获的索引。当`nth`为0（默认值）时，返回整个匹配子串的索引；当`nth`为2时，返回第二个匹配子串的索引，以此类推。当指定的捕获没有对应匹配，返回两个`nil`。看下面的这个例子：

```lua

 local str = "hello, 1234"
 local from, to = ngx.re.find(str, "([0-9])([0-9]+)", "jo", nil, 2)
 if from then
     ngx.say("matched 2nd submatch: ", string.sub(str, from, to))  -- 输出”234“
 end
```

该函数最早出现在`v0.9.2`版本中。

[回到目录](#nginx-api-for-lua)

ngx.re.gmatch
-------------
**语法:** *iterator, err = ngx.re.gmatch(subject, regex, options?)*

**上下文:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

该函数与[ngx.re.match](#ngxrematch)类似，不同的是会返回一个Lua迭代器，以便用户可以遍历`<subject>`中匹配正则表达式`regex`的所有匹配。

如果发生错误（如错误正则表达式），会返回`nil`和一个错误描述字符串。

下面是一个描述该函数基本用法的例子：

```lua

 local iterator, err = ngx.re.gmatch("hello, world!", "([a-z]+)", "i")
 if not iterator then
     ngx.log(ngx.ERR, "error: ", err)
     return
 end

 local m
 m, err = iterator()    -- m[0] == m[1] == "hello"
 if err then
     ngx.log(ngx.ERR, "error: ", err)
     return
 end

 m, err = iterator()    -- m[0] == m[1] == "world"
 if err then
     ngx.log(ngx.ERR, "error: ", err)
     return
 end

 m, err = iterator()    -- m == nil
 if err then
     ngx.log(ngx.ERR, "error: ", err)
     return
 end
```

更常见的做法是，我们会使用循环来迭代：

```lua

 local it, err = ngx.re.gmatch("hello, world!", "([a-z]+)", "i")
 if not it then
     ngx.log(ngx.ERR, "error: ", err)
     return
 end

 while true do
     local m, err = it()
     if err then
         ngx.log(ngx.ERR, "error: ", err)
         return
     end

     if not m then
         -- 不再有匹配
         break
     end

     -- 匹配
     ngx.say(m[0])
     ngx.say(m[1])
 end
```

可选参数`options`的语义与在[ngx.re.match](#ngxrematch)中的相同。

当前的实现要求返回的迭代器仅能用于单个请求当中。也就是说，绝对不能将它赋值给一个拥有连续命名空间的变量（如Lua包）。

该方法需要Nginx使用PCRE库。([Known Issue With Special Escaping Sequences](#special-escaping-sequences))。

该功能最早出现在`v0.2.1rc12`版本中。

[回到目录](#nginx-api-for-lua)

ngx.re.sub
----------
**语法:** *newstr, n, err = ngx.re.sub(subject, regex, replace, options?)*

**上下文:** *init_worker_by_lua, set_by_lua, rewrite_by_lua, access_by_lua, content_by_lua, header_filter_by_lua, body_filter_by_lua, log_by_lua, ngx.timer.*

对参数`subject`字符串使用字符串或函数参数`replace`替换第一个匹配`regex`的子串。可选参数`options`的语义与[ngx.re.match](#ngxrematch)中的相同。

该方法返回新的结果字符串和成功替换的数量。如果发生错误，返回`nil`和一个错误描述字符串。

如果`replace`是一个字符串，它被看作是用于字符串替换的特殊模板。例如：

```lua

 local newstr, n, err = ngx.re.sub("hello, 1234", "([0-9])[0-9]", "[$0][$1]")
 if newstr then
     -- newstr == "hello, [12][1]34"
     -- n == 1
 else
     ngx.log(ngx.ERR, "error: ", err)
     return
 end
```

`$0`是整个模式匹配的子串，`$1`是第一个括号匹配捕获的子串。

花括号也用于消除歧义：

```lua

 local newstr, n, err = ngx.re.sub("hello, 1234", "[0-9]", "${0}00")
     -- newstr == "hello, 100234"
     -- n == 1
```

Literal dollar sign characters (`$`) in the `replace` string argument can be escaped by another dollar sign, for instance,

```lua

 local newstr, n, err = ngx.re.sub("hello, 1234", "[0-9]", "$$")
     -- newstr == "hello, $234"
     -- n == 1
```

不要使用反斜杠转移`$`，会有问题哦。

如果`replace`参数是函数类型，该函数会以匹配列表作为参数，产生用于替换的替换字符串。`replace`函数的输入参数”match table"与[ngx.re.match](#ngxrematch)的返回值相同。看下面的例子：

```lua

 local func = function (m)
     return "[" .. m[0] .. "][" .. m[1] .. "]"
 end
 local newstr, n, err = ngx.re.sub("hello, 1234", "( [0-9] ) [0-9]", func, "x")
     -- newstr == "hello, [12][1]34"
     -- n == 1
```

`replace`函数参数的返回值中的`$`字符没有特殊含义。

该方法要求Nginx使用PCRE库。([Known Issue With Special Escaping Sequences](#special-escaping-sequences))。

该功能最早出现在`v0.2.1rc13`版本中。

[回到目录](#nginx-api-for-lua)

ngx.re.gsub
-----------
**语法:** *newstr, n, err = ngx.re.gsub(subject, regex, replace, options?)*

**上下文:** *init_worker_by_lua, set_by_lua, rewrite_by_lua, access_by_lua, content_by_lua, header_filter_by_lua, body_filter_by_lua, log_by_lua, ngx.timer.*

与[ngx.re.sub](#ngxresub)类似，但是做全局替换。

这是一些例子：

```lua

 local newstr, n, err = ngx.re.gsub("hello, world", "([a-z])[a-z]+", "[$0,$1]", "i")
 if newstr then
     -- newstr == "[hello,h], [world,w]"
     -- n == 2
 else
     ngx.log(ngx.ERR, "error: ", err)
     return
 end
```

```lua

 local func = function (m)
     return "[" .. m[0] .. "," .. m[1] .. "]"
 end
 local newstr, n, err = ngx.re.gsub("hello, world", "([a-z])[a-z]+", func, "i")
     -- newstr == "[hello,h], [world,w]"
     -- n == 2
```

该方法要求Nginx使用PCRE库。([Known Issue With Special Escaping Sequences](#special-escaping-sequences))。

该功能最早出现在`v0.2.1rc15`版本中。

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT
---------------
**syntax:** *dict = ngx.shared.DICT*

**syntax:** *dict = ngx.shared\[name_var\]*

**context:** *init_by_lua*, init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Fetching the shm-based Lua dictionary object for the shared memory zone named `DICT` defined by the [lua_shared_dict](#lua_shared_dict) directive.

Shared memory zones are always shared by all the nginx worker processes in the current nginx server instance.

The resulting object `dict` has the following methods:

* [get](#ngxshareddictget)
* [get_stale](#ngxshareddictget_stale)
* [set](#ngxshareddictset)
* [safe_set](#ngxshareddictsafe_set)
* [add](#ngxshareddictadd)
* [safe_add](#ngxshareddictsafe_add)
* [replace](#ngxshareddictreplace)
* [delete](#ngxshareddictdelete)
* [incr](#ngxshareddictincr)
* [flush_all](#ngxshareddictflush_all)
* [flush_expired](#ngxshareddictflush_expired)
* [get_keys](#ngxshareddictget_keys)

Here is an example:

```nginx

 http {
     lua_shared_dict dogs 10m;
     server {
         location /set {
             content_by_lua '
                 local dogs = ngx.shared.dogs
                 dogs:set("Jim", 8)
                 ngx.say("STORED")
             ';
         }
         location /get {
             content_by_lua '
                 local dogs = ngx.shared.dogs
                 ngx.say(dogs:get("Jim"))
             ';
         }
     }
 }
```

Let us test it:

```bash

 $ curl localhost/set
 STORED

 $ curl localhost/get
 8

 $ curl localhost/get
 8
```

The number `8` will be consistently output when accessing `/get` regardless of how many Nginx workers there are because the `dogs` dictionary resides in the shared memory and visible to *all* of the worker processes.

The shared dictionary will retain its contents through a server config reload (either by sending the `HUP` signal to the Nginx process or by using the `-s reload` command-line option).

The contents in the dictionary storage will be lost, however, when the Nginx server quits.

This feature was first introduced in the `v0.3.1rc22` release.

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.get
-------------------
**syntax:** *value, flags = ngx.shared.DICT:get(key)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Retrieving the value in the dictionary [ngx.shared.DICT](#ngxshareddict) for the key `key`. If the key does not exist or has been expired, then `nil` will be returned.

In case of errors, `nil` and a string describing the error will be returned.

The value returned will have the original data type when they were inserted into the dictionary, for example, Lua booleans, numbers, or strings.

The first argument to this method must be the dictionary object itself, for example,

```lua

 local cats = ngx.shared.cats
 local value, flags = cats.get(cats, "Marry")
```

or use Lua's syntactic sugar for method calls:

```lua

 local cats = ngx.shared.cats
 local value, flags = cats:get("Marry")
```

These two forms are fundamentally equivalent.

If the user flags is `0` (the default), then no flags value will be returned.

This feature was first introduced in the `v0.3.1rc22` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.get_stale
-------------------------
**syntax:** *value, flags, stale = ngx.shared.DICT:get_stale(key)*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Similar to the [get](#ngxshareddictget) method but returns the value even if the key has already expired.

Returns a 3rd value, `stale`, indicating whether the key has expired or not.

Note that the value of an expired key is not guaranteed to be available so one should never rely on the availability of expired items.

This method was first introduced in the `0.8.6` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.set
-------------------
**syntax:** *success, err, forcible = ngx.shared.DICT:set(key, value, exptime?, flags?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Unconditionally sets a key-value pair into the shm-based dictionary [ngx.shared.DICT](#ngxshareddict). Returns three values:

* `success`: boolean value to indicate whether the key-value pair is stored or not.
* `err`: textual error message, can be `"no memory"`.
* `forcible`: a boolean value to indicate whether other valid items have been removed forcibly when out of storage in the shared memory zone.

The `value` argument inserted can be Lua booleans, numbers, strings, or `nil`. Their value type will also be stored into the dictionary and the same data type can be retrieved later via the [get](#ngxshareddictget) method.

The optional `exptime` argument specifies expiration time (in seconds) for the inserted key-value pair. The time resolution is `0.001` seconds. If the `exptime` takes the value `0` (which is the default), then the item will never be expired.

The optional `flags` argument specifies a user flags value associated with the entry to be stored. It can also be retrieved later with the value. The user flags is stored as an unsigned 32-bit integer internally. Defaults to `0`. The user flags argument was first introduced in the `v0.5.0rc2` release.

When it fails to allocate memory for the current key-value item, then `set` will try removing existing items in the storage according to the Least-Recently Used (LRU) algorithm. Note that, LRU takes priority over expiration time here. If up to tens of existing items have been removed and the storage left is still insufficient (either due to the total capacity limit specified by [lua_shared_dict](#lua_shared_dict) or memory segmentation), then the `err` return value will be `no memory` and `success` will be `false`.

If this method succeeds in storing the current item by forcibly removing other not-yet-expired items in the dictionary via LRU, the `forcible` return value will be `true`. If it stores the item without forcibly removing other valid items, then the return value `forcible` will be `false`.

The first argument to this method must be the dictionary object itself, for example,

```lua

 local cats = ngx.shared.cats
 local succ, err, forcible = cats.set(cats, "Marry", "it is a nice cat!")
```

or use Lua's syntactic sugar for method calls:

```lua

 local cats = ngx.shared.cats
 local succ, err, forcible = cats:set("Marry", "it is a nice cat!")
```

These two forms are fundamentally equivalent.

This feature was first introduced in the `v0.3.1rc22` release.

Please note that while internally the key-value pair is set atomically, the atomicity does not go across the method call boundary.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.safe_set
------------------------
**syntax:** *ok, err = ngx.shared.DICT:safe_set(key, value, exptime?, flags?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Similar to the [set](#ngxshareddictset) method, but never overrides the (least recently used) unexpired items in the store when running out of storage in the shared memory zone. In this case, it will immediately return `nil` and the string "no memory".

This feature was first introduced in the `v0.7.18` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.add
-------------------
**syntax:** *success, err, forcible = ngx.shared.DICT:add(key, value, exptime?, flags?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Just like the [set](#ngxshareddictset) method, but only stores the key-value pair into the dictionary [ngx.shared.DICT](#ngxshareddict) if the key does *not* exist.

If the `key` argument already exists in the dictionary (and not expired for sure), the `success` return value will be `false` and the `err` return value will be `"exists"`.

This feature was first introduced in the `v0.3.1rc22` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.safe_add
------------------------
**syntax:** *ok, err = ngx.shared.DICT:safe_add(key, value, exptime?, flags?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Similar to the [add](#ngxshareddictadd) method, but never overrides the (least recently used) unexpired items in the store when running out of storage in the shared memory zone. In this case, it will immediately return `nil` and the string "no memory".

This feature was first introduced in the `v0.7.18` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.replace
-----------------------
**syntax:** *success, err, forcible = ngx.shared.DICT:replace(key, value, exptime?, flags?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Just like the [set](#ngxshareddictset) method, but only stores the key-value pair into the dictionary [ngx.shared.DICT](#ngxshareddict) if the key *does* exist.

If the `key` argument does *not* exist in the dictionary (or expired already), the `success` return value will be `false` and the `err` return value will be `"not found"`.

This feature was first introduced in the `v0.3.1rc22` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.delete
----------------------
**syntax:** *ngx.shared.DICT:delete(key)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Unconditionally removes the key-value pair from the shm-based dictionary [ngx.shared.DICT](#ngxshareddict).

It is equivalent to `ngx.shared.DICT:set(key, nil)`.

This feature was first introduced in the `v0.3.1rc22` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.incr
--------------------
**syntax:** *newval, err = ngx.shared.DICT:incr(key, value)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Increments the (numerical) value for `key` in the shm-based dictionary [ngx.shared.DICT](#ngxshareddict) by the step value `value`. Returns the new resulting number if the operation is successfully completed or `nil` and an error message otherwise.

The key must already exist in the dictionary, otherwise it will return `nil` and `"not found"`.

If the original value is not a valid Lua number in the dictionary, it will return `nil` and `"not a number"`.

The `value` argument can be any valid Lua numbers, like negative numbers or floating-point numbers.

This feature was first introduced in the `v0.3.1rc22` release.

See also [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.flush_all
-------------------------
**syntax:** *ngx.shared.DICT:flush_all()*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Flushes out all the items in the dictionary. This method does not actuall free up all the memory blocks in the dictionary but just marks all the existing items as expired.

This feature was first introduced in the `v0.5.0rc17` release.

See also [ngx.shared.DICT.flush_expired](#ngxshareddictflush_expired) and [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.flush_expired
-----------------------------
**syntax:** *flushed = ngx.shared.DICT:flush_expired(max_count?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Flushes out the expired items in the dictionary, up to the maximal number specified by the optional `max_count` argument. When the `max_count` argument is given `0` or not given at all, then it means unlimited. Returns the number of items that have actually been flushed.

Unlike the [flush_all](#ngxshareddictflush_all) method, this method actually free up the memory used by the expired items.

This feature was first introduced in the `v0.6.3` release.

See also [ngx.shared.DICT.flush_all](#ngxshareddictflush_all) and [ngx.shared.DICT](#ngxshareddict).

[回到目录](#nginx-api-for-lua)

ngx.shared.DICT.get_keys
------------------------
**syntax:** *keys = ngx.shared.DICT:get_keys(max_count?)*

**context:** *init_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

Fetch a list of the keys from the dictionary, up to `<max_count>`.

By default, only the first 1024 keys (if any) are returned. When the `<max_count>` argument is given the value `0`, then all the keys will be returned even there is more than 1024 keys in the dictionary.

**WARNING** Be careful when calling this method on dictionaries with a really huge number of keys. This method may lock the dictionary for quite a while and block all the nginx worker processes that are trying to access the dictionary.

This feature was first introduced in the `v0.7.3` release.

[回到目录](#nginx-api-for-lua)

ngx.socket.udp
--------------
**syntax:** *udpsock = ngx.socket.udp()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

Creates and returns a UDP or datagram-oriented unix domain socket object (also known as one type of the "cosocket" objects). The following methods are supported on this object:

* [setpeername](#udpsocksetpeername)
* [send](#udpsocksend)
* [receive](#udpsockreceive)
* [close](#udpsockclose)
* [settimeout](#udpsocksettimeout)

It is intended to be compatible with the UDP API of the [LuaSocket](http://w3.impa.br/~diego/software/luasocket/udp.html) library but is 100% nonblocking out of the box.

This feature was first introduced in the `v0.5.7` release.

See also [ngx.socket.tcp](#ngxsockettcp).

[回到目录](#nginx-api-for-lua)

udpsock:setpeername
-------------------
**syntax:** *ok, err = udpsock:setpeername(host, port)*

**syntax:** *ok, err = udpsock:setpeername("unix:/path/to/unix-domain.socket")*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

Attempts to connect a UDP socket object to a remote server or to a datagram unix domain socket file. Because the datagram protocol is actually connection-less, this method does not really establish a "connection", but only just set the name of the remote peer for subsequent read/write operations.

Both IP addresses and domain names can be specified as the `host` argument. In case of domain names, this method will use Nginx core's dynamic resolver to parse the domain name without blocking and it is required to configure the [resolver](http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver) directive in the `nginx.conf` file like this:

```nginx

 resolver 8.8.8.8;  # use Google's public DNS nameserver
```

If the nameserver returns multiple IP addresses for the host name, this method will pick up one randomly.

In case of error, the method returns `nil` followed by a string describing the error. In case of success, the method returns `1`.

Here is an example for connecting to a UDP (memcached) server:

```nginx

 location /test {
     resolver 8.8.8.8;

     content_by_lua '
         local sock = ngx.socket.udp()
         local ok, err = sock:setpeername("my.memcached.server.domain", 11211)
         if not ok then
             ngx.say("failed to connect to memcached: ", err)
             return
         end
         ngx.say("successfully connected to memcached!")
         sock:close()
     ';
 }
```

Since the `v0.7.18` release, connecting to a datagram unix domain socket file is also possible on Linux:

```lua

 local sock = ngx.socket.udp()
 local ok, err = sock:setpeername("unix:/tmp/some-datagram-service.sock")
 if not ok then
     ngx.say("failed to connect to the datagram unix domain socket: ", err)
     return
 end
```

assuming the datagram service is listening on the unix domain socket file `/tmp/some-datagram-service.sock` and the client socket will use the "autobind" feature on Linux.

Calling this method on an already connected socket object will cause the original connection to be closed first.

This method was first introduced in the `v0.5.7` release.

[回到目录](#nginx-api-for-lua)

udpsock:send
------------
**syntax:** *ok, err = udpsock:send(data)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

Sends data on the current UDP or datagram unix domain socket object.

In case of success, it returns `1`. Otherwise, it returns `nil` and a string describing the error.

The input argument `data` can either be a Lua string or a (nested) Lua table holding string fragments. In case of table arguments, this method will copy all the string elements piece by piece to the underlying Nginx socket send buffers, which is usually optimal than doing string concatenation operations on the Lua land.

This feature was first introduced in the `v0.5.7` release.

[回到目录](#nginx-api-for-lua)

udpsock:receive
---------------
**syntax:** *data, err = udpsock:receive(size?)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

Receives data from the UDP or datagram unix domain socket object with an optional receive buffer size argument, `size`.

This method is a synchronous operation and is 100% nonblocking.

In case of success, it returns the data received; in case of error, it returns `nil` with a string describing the error.

If the `size` argument is specified, then this method will use this size as the receive buffer size. But when this size is greater than `8192`, then `8192` will be used instead.

If no argument is specified, then the maximal buffer size, `8192` is assumed.

Timeout for the reading operation is controlled by the [lua_socket_read_timeout](#lua_socket_read_timeout) config directive and the [settimeout](#udpsocksettimeout) method. And the latter takes priority. For example:

```lua

 sock:settimeout(1000)  -- one second timeout
 local data, err = sock:receive()
 if not data then
     ngx.say("failed to read a packet: ", data)
     return
 end
 ngx.say("successfully read a packet: ", data)
```

It is important here to call the [settimeout](#udpsocksettimeout) method *before* calling this method.

This feature was first introduced in the `v0.5.7` release.

[回到目录](#nginx-api-for-lua)

udpsock:close
-------------
**syntax:** *ok, err = udpsock:close()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

Closes the current UDP or datagram unix domain socket. It returns the `1` in case of success and returns `nil` with a string describing the error otherwise.

Socket objects that have not invoked this method (and associated connections) will be closed when the socket object is released by the Lua GC (Garbage Collector) or the current client HTTP request finishes processing.

This feature was first introduced in the `v0.5.7` release.

[回到目录](#nginx-api-for-lua)

udpsock:settimeout
------------------
**syntax:** *udpsock:settimeout(time)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

Set the timeout value in milliseconds for subsequent socket operations (like [receive](#udpsockreceive)).

Settings done by this method takes priority over those config directives, like [lua_socket_read_timeout](#lua_socket_read_timeout).

This feature was first introduced in the `v0.5.7` release.

[回到目录](#nginx-api-for-lua)

ngx.socket.tcp
--------------
**语法:** *tcpsock = ngx.socket.tcp()*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

创建并返回一个TCP或流式unix domain socket对象（一种“cosocket"对象类型）。该对象支持以下方法：

* [connect](#tcpsockconnect)
* [sslhandshake](#tcpsocksslhandshake)
* [send](#tcpsocksend)
* [receive](#tcpsockreceive)
* [close](#tcpsockclose)
* [settimeout](#tcpsocksettimeout)
* [setoption](#tcpsocksetoption)
* [receiveuntil](#tcpsockreceiveuntil)
* [setkeepalive](#tcpsocksetkeepalive)
* [getreusedtimes](#tcpsockgetreusedtimes)

该对象尽量做到兼容[LuaSocket](http://w3.impa.br/~diego/software/luasocket/tcp.html)库的TCP API，但是100%非阻塞。另外，会引入一些新的API增强功能。

该函数创建的cosocket对象，生命周期与创建该对象的Lua处理函数相同。所以，不可以将cosocket对象传递给其他Lua处理函数（包括ngx.timer回调函数）。也不要在不同的Nginx请求之间共享cosocket对象。

对于一个连接上的每个cosocket对象，如果你不显式的关闭（通过调用[close](#tcpsockclose))或者将其放回连接池（通过[setkeepalive](#tcpsocksetkeepalive))，那么以下两个事件任意一个发生，它会自动关闭：

* 当前请求处理结束；
* Lua cosocket对象的值被Lua GC回收。

在操作cosocket过程中如果发生致命错误，cosocket总是会自动关闭当前连接（注意，读超时错误并不属于致命错误），并且，如果你在一个已关闭的连接上再调用[close](#tcpsockclose)，会得到一个"已关闭"错误。

从`v0.9.9`版本开始，cosocket对象就已经是全双工的了。也就是说，一个读"轻线程"和一个写"轻线程"可以同时操作同一个cosocket对象（但两个"轻线程"必须属于同一个Lua处理函数，原因见上面）。但是，你不能在同一个cosocket对象上同时有两个"轻线程"进行读操作（写操作，连接操作同样），否则，当你调用cosocket的方法时，可能会获得”socket busy reading“的错误。

该方法最早出现在`v0.5.0rc1`版本。

See also [ngx.socket.udp](#ngxsocketudp)。

[回到目录](#nginx-api-for-lua)

tcpsock:connect
---------------
**语法:** *ok, err = tcpsock:connect(host, port, options_table?)*

**语法:** *ok, err = tcpsock:connect("unix:/path/to/unix-domain.socket", options_table?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

以非阻塞方式连接远端服务器的TCP socket对象或流式unix domain socket文件。

在真正进行域名解析和连接远端机器之前，该方法总是会先在连接池中查找与之前调用该方法（或者[ngx.socket.connect](ngxsocketconnect)函数）创建的连接相匹配的空闲连接。

`host`参数可以是IP地址或域名。如果是域名，该方法会使用Nginx内核中的动态域名解析服务进行非阻塞的域名解析，这需要在`nginx.conf`中配置[resolver](http://nginx.org/en/docs/http/ngx_http_core_module.html#resolver) 指令。像这样：

```nginx

 resolver 8.8.8.8;  # 使用谷歌的公共DNS域名服务
```

如果域名服务器返回多个IP地址，该方法会随机的选取一个。

如果发生错误，该方法返回`nil`并跟上一个错误字符串。成功返回`1`。

以下是连接TCP服务器的例子：

```nginx

 location /test {
     resolver 8.8.8.8;

     content_by_lua '
         local sock = ngx.socket.tcp()
         local ok, err = sock:connect("www.google.com", 80)
         if not ok then
             ngx.say("failed to connect to google: ", err)
             return
         end
         ngx.say("successfully connected to google!")
         sock:close()
     ';
 }
```

也可以连接Unix Domain Socket文件:

```lua

 local sock = ngx.socket.tcp()
 local ok, err = sock:connect("unix:/tmp/memcached.sock")
 if not ok then
     ngx.say("failed to connect to the memcached unix domain socket: ", err)
     return
 end
```

以上假设memcached（或其他什么）监听的Unix Domain Socket文件`/tmp/memcached.sock`。

连接的超时时间通过[lua_socket_connect_timeout](#lua_socket_connect_timeout) 配置指令和[settimeout](#tcpsocksettimeout)方法控制。后者优先级更高，例如：

```lua

 local sock = ngx.socket.tcp()
 sock:settimeout(1000)  -- 一秒超时
 local ok, err = sock:connect(host, port)
```

这里调用该方法*之前*调用[settimeout](#tcpsocksettimeout)方法很重要。

在一个已经建立连接的socket对象上调用该方法会先关闭之前的连接。

最后一个参数是一个可选的Lua table，用以指定多个连接选项：

* `pool`
    给连接池指定一个自定义名字。如果忽略，连接池的名字会是这样`"<host>:<port>"`或者`"<unix-socket-path>"`。

支持可选的table参数的功能最早在`v0.5.7`版本引入。

该方法最早出现在`v.0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:sslhandshake
--------------------
**语法:** *session, err = tcpsock:sslhandshake(reused_session?, server_name?, ssl_verify?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

在当前已建立的连接上，做SSL/TLS握手操作。

选项`reused_seesion`对相同的目标使用之前通过`sslhandshake`返回的SSL会话数据。对于短链接，重用SSL会话通常可以一个数量级的加速握手，但是，如果使用连接池时，该选项并不多么有用。该参数默认为`nil`。如果该参数指定为`false`，该调用不返回SSL会话用户数据，只返回一个布尔值作为第一个返回值；否则总是返回当前SSL会话作为第一个作为第一个参数（如果成功的话）。

选项`server_name`用于指定用于新TLS扩展SNI的主机名。使用SNI可以在服务端同一个IP地址共享不同的服务器主机。而且，如果开启了SSL校验，该`server_name`参数也用于验证远端发来的服务证书中指定的服务器名称的有效性。

选项`ssl_verify`是一个布尔值，用于控制是否进行SSL校验。如果设为`true`，会根据[lua_ssl_trusted_certificate](#lua_ssl_trusted_certificate)指令指定的CA证书对服务证书进行校验。你也可能使用[lua_ssl_verify_depth](#lua_ssl_verify_depth)指令来控制跟踪证书链的深度。同样，当`ssl_vrify`指定为true，并指定了`server_name`，后者会用于主机名校验。

对于已经进行过SSL/TLS握手操作的链接，该方法立即返回。

该方法最早出现在`v0.9.11`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:send
------------
**语法:** *bytes, err = tcpsock:send(data)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

在当前TCP或Unix Domain Socket连接上以非阻塞的方式发送数据。

该方法是一个同步操作，也就是会等到*所有*的数据都拷贝到系统发送缓冲区或者错误发生时才会返回。

如果成功，该方法返回已发送的总字节数。否则，返回`nil`和描述错误的字符串。

输入参数`data`可以是Lua字符串，也可以是存储字符串片段的（嵌套的）Lua table。如果是table参数，该方法会将table的字符串元素逐个拷贝到底层的Nginx socket发送缓冲区，这通常比字符串拼接操作高效。

发送操作的超时时间通过[lua_socket_send_timeout](#lua_socket_send_timeout)配置指令和[settimeout](#tcpsocksettimeout)方法控制。后者优先级更高，例如：

```lua

 sock:settimeout(1000)  -- 一秒超时
 local bytes, err = sock:send(request)
```

这里调用该方法*之前*调用[settimeout](#tcpsocksettimeout)方法很重要。

如果发生任何连接错误，该方法总会自动关闭当前连接。

该方法首次出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:receive
---------------
**语法:** *data, err, partial = tcpsock:receive(size)*

**语法:** *data, err, partial = tcpsock:receive(pattern?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

按照读取pattern和大小，从已连接socket上获取数据。

与[send](#tcpsocksend)类似，该方法属于同步非阻塞方法。

如果成功，该方法返回获取的数据。如果失败，返回`nil`和描述错误的字符串，以及目前收到的部分数据。

如果参数是数字型（包括可以转化成数字的字符串），会被解释成`size`。该方法不会返回直到读取到指定大小数据或者发生错误。

如果参数是非数字类型，会被解释成`pattern`。支持以下patterns：

* `'*a'`: 从socket读取数据知道连接关闭。不翻译行尾标记；
* `'*l'`: 从socket读取一行文本。文本行以`LF`字符（ASCII 10）结尾，可能前面带有`CR`字符（ASCII 13）。CR和LF字符不包含在返回的文本行中。事实上，所有的CR字符都会被pattern忽略。

如果不指定参数，则默认使用pattern`'*l'`，即航读取模式。

读操作的超时时间可以通过[lua_socket_read_timeout](#lua_socket_read_timeout)配置指令和[settimeout](#tcpsocksettimeout)方法控制。后者优先级更高，例如：

```lua

 sock:settimeout(1000)  -- 一秒超时
 local line, err, partial = sock:receive()
 if not line then
     ngx.say("failed to read a line: ", err)
     return
 end
 ngx.say("successfully read a line: ", line)
```

这里调用该方法*之前*调用[settimeout](#tcpsocksettimeout)方法很重要。

从`v0.8.8`版本开始，如果读取超时错误发生，该方法不再自动关闭当前连接。如果发生其他连接错误，该方法总会自动关闭当前连接。

该方法首次出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:receiveuntil
--------------------
**语法:** *iterator = tcpsock:receiveuntil(pattern, options?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

该函数返回一个迭代器函数，用于读取数据流直到读到特定pattern或发生错误。

下面是使用该方法读取数据流的例子（以`--abcedhb`结束）：

```lua

 local reader = sock:receiveuntil("\r\n--abcedhb")
 local data, err, partial = reader()
 if not data then
     ngx.say("failed to read the data stream: ", err)
 end
 ngx.say("read the data stream: ", data)
```

如果不带任何参数，迭代器函数会返回在数据流中指定pattern之前的部分。还是上面的例子，如果数据流是`'hello, world! -agentzh\r\n--abcedhb blah blah'`，那么返回的字符串是`'hello, world! -agentzh'`。

如果发生错误，迭代器函数会返回`nil`和一个错误描述字符串以及到目前为止已经收到的数据。

迭代器函数可以被调用多次，也可以与其他cosocket方法或其他迭代器方法混合使用。

当调用时带有`size`参数时，迭代器函数的行为稍有不同。具体地说就是，它的每次调用只读取指定大小的数据，直到最后一次调用返回`nil`（或者读到边界pattern，或者发生错误）。对于最后一次成功的迭代器调用来说，`err`也会返回`nil`。迭代器函数在最后一次成功调用返回`nil`数据和`nil`错误后会被重置。看下面的例子：

```lua

 local reader = sock:receiveuntil("\r\n--abcedhb")

 while true do
     local data, err, partial = reader(4)
     if not data then
         if err then
             ngx.say("failed to read the data stream: ", err)
             break
         end

         ngx.say("read done")
         break
     end
     ngx.say("read chunk: [", data, "]")
 end
```

对于输入数据`'hello, world! -agentzh\r\n--abcedhb blah blah'`，上面的程序会得到下面的输出：

    read chunk: [hell]
    read chunk: [o, w]
    read chunk: [orld]
    read chunk: [! -a]
    read chunk: [gent]
    read chunk: [zh]
    read done


注意，当边界pattern存在二义性时，实际返回的数据可能比指定`size`大小的数据稍长一点。在数据流边界处，实际返回的数据字符串也可能比指定大小短。

迭代器函数的读操作超时时间通过[lua_socket_read_timeout](#lua_socket_read_timeout)配置指令和[settimeout](#tcpsocksettimeout)方法控制。后者优先级更高，例如：

```lua

 local readline = sock:receiveuntil("\r\n")

 sock:settimeout(1000)  -- 一秒超时
 line, err, partial = readline()
 if not line then
     ngx.say("failed to read a line: ", err)
     return
 end
 ngx.say("successfully read a line: ", line)
```

请在调用迭代器函数之前调用[settimeout](#tcpsocksettimeout)设置超时（这与调用`receiveuntil`不相关）。

从`v0.5.1`版本开始，该方法可以带有可选参数`options`，用于行为控制。以下是支持的选项：

* `inclusive`

`inclusive`是一个布尔值，用以控制在返回的数据中是否包含pattern字符串。默认为`false`。例如：

```lua

 local reader = tcpsock:receiveuntil("_END_", { inclusive = true })
 local data = reader()
 ngx.say(data)
```

对于输入字符串`"hello world _END_ blah blah blah"`，上面的程序会输出`hello world _END_`，包含pattern字符串`_END_`本身。

从`v0.8.8`版本开始，当读取超时时，该方法不再自动关闭当前连接。对于其他错误，该方法总是自动关闭连接。

该方法最早出现在`v0.5.0rc1`版本中。

[回到目录](#nginx-api-for-lua)

tcpsock:close
-------------
**语法:** *ok, err = tcpsock:close()*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

关闭当前的TCP或流式Unix Domain Socket。如果成功返回`1`；如果失败返回`nil`和描述错误的字符串。

注意，如果在socket对象上调用了[setkeepalive](#tcpsocksetkeepalive)方法，那么不需要在socket对象上调用该方法，因为socket对象已经关闭（并且，当前连接被保存在了内建的连接池中）。

没有调用该方法的socket对象（和关联的连接）会在Lua GC释放socket对象或当前的HTTP请求完成处理时被关闭。

该方法最早出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:settimeout
------------------
**语法:** *tcpsock:settimeout(time)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

对后续的socket操作（[connect](#tcpsockconnect)，[receive](#tcpsockreceive)和从[receiveuntil](#tcpsockreceiveuntil)返回的迭代器）设置超时时间（单位毫秒）。

该方法设置的超时优先级高于其他配置指令，如[lua_socket_connect_timeout](#lua_socket_connect_timeout)，[lua_socket_send_timeout](#lua_socket_send_timeout)，和[lua_socket_read_timeout](#lua_socket_read_timeout)等。

注意，该方法不影响[lua_socket_keepalive_timeout](#lua_socket_keepalive_timeout)设置，如果要设置keepalive超时，请使用[setkeepalive](#tcpsocksetkeepalive)方法。

该方法最早出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:setoption
-----------------
**语法:** *tcpsock:setoption(option, value?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

该函数的加入是为了兼容[LuaSocket](http://w3.impa.br/~diego/software/luasocket/tcp.html) API，现在不做任何事情。未来会实现其功能。

该功能最早出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:setkeepalive
--------------------
**语法:** *ok, err = tcpsock:setkeepalive(timeout?, size?)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

将当前的socket连接立即放入cosocket内建的连接池，并保持活动连接，直到其他的请求调用[connect](#tcpsockconnect)方法或者对应的空闲过期时间到达。

第一个可选参数`timeout`用于指定当前连接的最大空闲时间（单位毫秒）。如果没指定，使用[lua_socket_keepalive_timeout](#lua_socket_keepalive_timeout)配置指令的默认配置时间。如果指定为`0`，则表示永不超时。

第二个可选参数`size`用于指定当前server的连接池内允许的最大连接数。注意，连接池的大小一旦在连接池创建后不能改变。如果没有指定该参数，使用[lua_socket_pool_size](#lua_socket_pool_size)的默认配置。

如果连接池超过大小限制，通过LRU的剔除策略为当前连接腾出空间。

注意，cosocket连接池是针对每个Nginx工作进程的，而不是对每个Nginx server实例的，因此，此处指定的大小限制适用于单个Nginx工作进程。

连接池中空闲连接的任何异常事件都会被监控到，这里的异常事件包括连接断开，异常数据等，发生异常事件后，对应的异常连接会被自动关闭，并被移出连接池。

调用成功，程序返回`1`；否则，返回`nil`和一个错误描述字符串。

当当前连接的接收缓冲区有未读数据时，该方法会返回"connection in dubious state"的错误信息（第二个返回值），因为之前的会话存在未读数据没有被读取，重用该连接是不安全的。

该方法会使当前的cosocket对象进入”closed“状态，所以不需要手动调用[close](#tcpsockclose)关闭连接。

该方法最早出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

tcpsock:getreusedtimes
----------------------
**语法:** *count, err = tcpsock:getreusedtimes()*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

该方法调用成功时返回当前连接的重用次数。发生错误时，返回`nil`和一个错误描述字符串。

如果当前连接不是来源于内建的连接池，该方法总是返回`0`，也就是连接不会被重用。如果连接来自连接池，那么返回值总是非零值。所以，该方法也可以用于确定该连接是否来自连接池。

该方法最早出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

ngx.socket.connect
------------------
**语法:** *tcpsock, err = ngx.socket.connect(host, port)*

**语法:** *tcpsock, err = ngx.socket.connect("unix:/path/to/unix-domain.socket")*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

该方法相当于同时调用[ngx.socket.tcp()](#ngxsockettcp)和[connect()](#tcpsockconnect)。实现上像下面这样：

```lua

 local sock = ngx.socket.tcp()
 local ok, err = sock:connect(...)
 if not ok then
     return nil, err
 end
 return sock
```

该方法不能使用[settimeout](#tcpsocksettimeout)方法设置连接超时时间，只能使用[lua_socket_connect_timeout](#lua_socket_connect_timeout)指令配置超时时间。

该方法最早出现在`v0.5.0rc1`版本。

[回到目录](#nginx-api-for-lua)

ngx.get_phase
-------------
**语法:** *str = ngx.get_phase()*

**上下文:** *init_by_lua*, init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

获取当前的执行阶段。可能的返回值包括：

* `init`
	用于[init_by_lua](#init_by_lua)和[init_by_lua_file](#init_by_lua_file)上下文。
* `init_worker`
	用于[init_worker_by_lua](#init_worker_by_lua)和[init_worker_by_lua_file](#init_worker_by_lua_file)上下文。
* `set`
	用于[set_by_lua](#set_by_lua) or [set_by_lua_file](#set_by_lua_file).
* `rewrite`
	用于[rewrite_by_lua](#rewrite_by_lua)和[rewrite_by_lua_file](#rewrite_by_lua_file)上下文。
* `access`
	用于[access_by_lua](#access_by_lua)和[access_by_lua_file](#access_by_lua_file)上下文。
* `content`
	用于[content_by_lua](#content_by_lua)和[content_by_lua_file](#content_by_lua_file)上下文。
* `header_filter`
	用于[header_filter_by_lua](#header_filter_by_lua)和[header_filter_by_lua_file](#header_filter_by_lua_file)上下文。
* `body_filter`
	用于[body_filter_by_lua](#body_filter_by_lua)和[body_filter_by_lua_file](#body_filter_by_lua_file)上下文。
* `log`
	用于[log_by_lua](#log_by_lua)和[log_by_lua_file](#log_by_lua_file)上下文。
* `timer`
	用于[ngx.timer.*](#ngxtimerat)用户回调函数。

该API最早出现与`v0.5.10`版本。

[回到目录](#nginx-api-for-lua)

ngx.thread.spawn
----------------
**语法:** *co = ngx.thread.spawn(func, arg1, arg2, ...)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

生成一个新的用户“新线程“，带有Lua函数`func`和可选参数`arg1`，`arg2`等，返回表示该”轻线程“的Lua线程或Lua协程。

”轻线程“只是一种特殊类型的Lua协程，由ngx_lua模块调度。

在`ngx.thread.spawn`返回之前，带有可选参数的`func`会被调用直到返回、遇到错误退出或者由于IO操作经由[Nginx API for Lua](#nginx-api-for-lua)（如[tcpsock:receive](#tcpsockreceive)）暂停执行。

在`ngx.thread.spawn`返回之后，新创建的”轻线程“通常会在多个IO事件中保持异步运行。

由[rewrite_by_lua](#rewrite_by_lua)、[access_by_lua](#access_by_lua)和[content_by_lua](#content_by_lua)运行的所有lua代码块会在模板”轻线程“中由ngx_lua统一自动创建。该模板”轻线程“又叫”入口线程“。

默认情况下，对应的Nginx处理函数（如[rewrite_by_lua](#rewrite_by_lua)处理函数）不会终止，直到

1. ”入口线程“和所有的用户”轻线程“都终止了；
2. "轻线程"（”入口线程“或用户“轻线程”）通过调用[ngx.exit](#ngxexit), [ngx.exec](#ngxexec), [ngx.redirect](#ngxredirect)或[ngx.req.set_uri(uri, true)](#ngxreqset_uri)退出；
3. “入口线程”遇到Lua错误终止。

如果用户“轻线程”遇到Lua错误终止，这不会使其他正在运行的“轻线程”（如“入口线程”）终止。

由于Nginx子请求模型的限制，通常不允许退出正在运行的子请求。所以，退出一个正在运行的等待一个或多个子请求的“轻线程”的做法也是被禁止的。在退出之前，你必须调用[ngx.thread.wait](#ngxthreadwait)等待这些“轻线程”退出。一个例外是，你可以通过调用带有，而且只能带有，`ngx.ERRER`(-1)、`408`、`444`或`499`状态码的[ngx.exit](#ngxexit)来退出等待中的子请求。

“轻线程”并不是以抢占的方式进行调度。换句话说，并没有时间片的轮转。“轻线程”会独占CPU保持运行，直到

1. （非阻塞的）IO操作不能在一次运行中完成；
2. 调用[coroutine.yield](#coroutineyield)主动放弃运行；
3. 发生错误或调用[ngx.exit](#ngxexit), [ngx.exec](#ngxexec), [ngx.redirect](#ngxredirect)或[ngx.req.set_uri(uri, true)](#ngxreqset_uri)退出。

对于前两个条件，“轻线程”通常会在之后被ngx_lua的调度器重新执行，除非整个程序退出的事件发生。

用户“轻线程“可以自己曾经”轻线程“。[coroutine.create](#coroutinecreate)创建的普通用户协程也可以创建”轻线程“。创建“轻线程”的协程称为“父协程”。

“父协程”可以调用[ngx.thread.wait](#ngxthreadwait)等待它的子“轻线程”退出。

你可以在”子轻线程“上调用coroutine.status()和coroutine。yield()。

”轻线程“协程的状态可能为”zombie“，如果

1. 当前”轻线程“已经退出（或成功或失败）；
2. 它的父协程仍处于活跃状态；
3. 它的父协程没有使用[ngx.thread.wait](#ngxthreadwait)等待。

下面的例子展示了在”轻线程“协程中使用coroutine.yield()做人工时间片：

```lua

 local yield = coroutine.yield

 function f()
     local self = coroutine.running()
     ngx.say("f 1")
     yield(self)
     ngx.say("f 2")
     yield(self)
     ngx.say("f 3")
 end

 local self = coroutine.running()
 ngx.say("0")
 yield(self)

 ngx.say("1")
 ngx.thread.spawn(f)

 ngx.say("2")
 yield(self)

 ngx.say("3")
 yield(self)

 ngx.say("4")
```

产生如下输出：


    0
    1
    f 1
    2
    f 2
    3
    f 3
    4


”轻线程“最常用于在单个Nginx请求处理函数中并发处理上游请求，好似一个通用版的[ngx.location.capture_multi](#ngxlocationcapture_multi)，可以跟所有的[Nginx API for Lua](#nginx-api-for-lua)协同工作。下面的例子展示了在单个Lua处理函数中并发请求MYSQL，Memcached和上游HTTP服务，并顺序输出结果（很像Facebook的BigPipe模型）：

```lua

 -- 同时查询mysql、memcached和远端http服务，
 -- 并按它们结果返回的顺序依次输出结果。

 local mysql = require "resty.mysql"
 local memcached = require "resty.memcached"

 local function query_mysql()
     local db = mysql:new()
     db:connect{
                 host = "127.0.0.1",
                 port = 3306,
                 database = "test",
                 user = "monty",
                 password = "mypass"
               }
     local res, err, errno, sqlstate =
             db:query("select * from cats order by id asc")
     db:set_keepalive(0, 100)
     ngx.say("mysql done: ", cjson.encode(res))
 end

 local function query_memcached()
     local memc = memcached:new()
     memc:connect("127.0.0.1", 11211)
     local res, err = memc:get("some_key")
     ngx.say("memcached done: ", res)
 end

 local function query_http()
     local res = ngx.location.capture("/my-http-proxy")
     ngx.say("http done: ", res.body)
 end

 ngx.thread.spawn(query_mysql)      -- 创建轻线程1
 ngx.thread.spawn(query_memcached)  -- 创建轻线程2
 ngx.thread.spawn(query_http)       -- 创建轻线程3
```

该API最早在`v0.7.0`版本可用。

[回到目录](#nginx-api-for-lua)

ngx.thread.wait
---------------
**语法:** *ok, res1, res2, ... = ngx.thread.wait(thread1, thread2, ...)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

等待一个或多个子"线程"结束，并返回第一个”线程“的结果（可能成功也可能失败）。

参数`thread1`, `thread2`等是先前调用[ngx.thread.spawn](#ngxthreadspawn)返回的Lua线程对象。

返回值的含义与[coroutine.resume](#coroutineresume)的返回值含义相同，即第一个值为布尔值，表示”线程“是否成功返回；之后的返回值表示用户Lua函数的返回值，用于产生”线程“（成功的情况）或错误对象（失败的情况）。

只有直接"父协程"才能等待其"子线程"，否则产生Lua异常。

下面的例子展示了`ngx.thread.wait`和[ngx.location.capture](#ngxlocationcapture)的用法，用来模拟[ngx.location.capture_multi](#ngxlocationcapture_multi):

```lua

 local capture = ngx.location.capture
 local spawn = ngx.thread.spawn
 local wait = ngx.thread.wait
 local say = ngx.say

 local function fetch(uri)
     return capture(uri)
 end

 local threads = {
     spawn(fetch, "/foo"),
     spawn(fetch, "/bar"),
     spawn(fetch, "/baz")
 }

 for i = 1, #threads do
     local ok, res = wait(threads[i])
     if not ok then
         say(i, ": failed to run: ", res)
     else
         say(i, ": status: ", res.status)
         say(i, ": body: ", res.body)
     end
 end
```

这本质上实现了"wait all"模式。

下面的例子展示了"wait any"的模式：

```lua

 function f()
     ngx.sleep(0.2)
     ngx.say("f: hello")
     return "f done"
 end

 function g()
     ngx.sleep(0.1)
     ngx.say("g: hello")
     return "g done"
 end

 local tf, err = ngx.thread.spawn(f)
 if not tf then
     ngx.say("failed to spawn thread f: ", err)
     return
 end

 ngx.say("f thread created: ", coroutine.status(tf))

 local tg, err = ngx.thread.spawn(g)
 if not tg then
     ngx.say("failed to spawn thread g: ", err)
     return
 end

 ngx.say("g thread created: ", coroutine.status(tg))

 ok, res = ngx.thread.wait(tf, tg)
 if not ok then
     ngx.say("failed to wait: ", res)
     return
 end

 ngx.say("res: ", res)

 -- stop the "world", aborting other running threads
 ngx.exit(ngx.OK)
```

输出如下：


    f thread created: running
    g thread created: running
    g: hello
    res: g done


该API最早出现在`v0.7.0`版本。

[回到目录](#nginx-api-for-lua)

ngx.thread.kill
---------------
**语法:** *ok, err = ngx.thread.kill(thread)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, ngx.timer.**

杀死[ngx.thread.spawn](#ngxthreadspawn)创建的正在运行的”轻线程“。成功返回true，否则返回`nil`和一个错误描述字符串。

根据当前的实现，只有父协程（或者父“轻线程”）可以杀死线程。同样，由于Nginx内核的限制，一个正在运行的发起Nginx子请求（由[ngx.location.capture](#ngxlocationcapture)发起）的“轻线程”不能被杀死。

该API最早在`v0.9.0`版本可用。

[回到目录](#nginx-api-for-lua)

ngx.on_abort
------------
**语法:** *ok, err = ngx.on_abort(callback)*

**上下文:** *rewrite_by_lua*, access_by_lua*, content_by_lua**

将一个用户Lua函数注册为回调函数，当客户端连接提前关闭时，被自动调用。

如果回调注册成功返回`1`，否则返回`nil`和一个错误描述字符串。

回调函数中可以使用所有的[Nginx API for Lua](#nginx-api-for-lua)，因为回调函数运行在一个特殊的`轻线程`中。

回调函数本身自己决定如何处理客户端退出事件。例如，可以简单忽略该事件（什么也不做），当前的lua请求处理函数不中断地继续执行。也可以通过调用[ngx.exit](#ngxexit)终止当前请求的处理。例如：

```lua

 local function my_cleanup()
     -- 这里是自定义的清理工作，比如取消等待的数据库事务

     -- 现在退出在当前请求处理函数中运行的所有“轻线程”
     ngx.exit(499)
 end

 local ok, err = ngx.on_abort(my_cleanup)
 if not ok then
     ngx.log(ngx.ERR, "failed to register the on_abort callback: ", err)
     ngx.exit(500)
 end
```

当[lua_check_client_abort](#lua_check_client_abort)设置为`off` 时（默认情况），该函数会总是返回错误信息“lua_check_client_abort is off”。

根据当前的实现，该函数在单个请求处理函数中仅能调用一次；后续的调用会返回错误信息"duplicate call"。

该API最早出现在`v0.7.4`版本中。

也可以参考[lua_check_client_abort](#lua_check_client_abort).

[回到目录](#nginx-api-for-lua)

ngx.timer.at
------------
**语法:** *ok, err = ngx.timer.at(delay, callback, user_arg1, user_arg2, ...)*

**上下文:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

创建一个Nginx定时器，该定时器带有一个用户回调函数和可选用户参数。

第一个参数`delay`指定了定时器的延时秒数。可以指定小数部分如`0.001`表示1毫秒。也可以指定为`0`表示当处理函数执行时定时器立即失效。

第二个参数`callback`可以是任何Lua函数，该函数会在指定的延时后在后台轻线程中被执行。用户回调可以被Nginx内核自动调用，带有参数`premature`，`user_arg1`，`user_arg2`等。`premature`为布尔值，表示该定时器是否提前过期；`user_arg1`、`user_arg2`等参数是调用`ngx.timer.at`时指定的用户参数。

定时器提前过期发生在Nginx工作进程关闭时（`HUP`信号触发的Nginx配置重载或Nginx服务器关闭）。当Nginx工作进程试图关闭时，将不能调用`ngx.timer.at`创建非零延迟的计时器，此时，`ngx.timer.at`会返回`nil`和一个错误描述字符串"process exiting"。

然而，从`v0.9.3`版本开始，在Nginx工作进程关闭时允许创建零延时计时器。

当定时器过期时，运行在轻线程中的用户代码完全脱离创建定时器的请求。所以，和创建计时器有相同生命周期的对象，如[cosockets](#ngxsockettcp)，不能在原始请求和用户定时器回调函数之间共享。

这是一个例子：

```nginx

 location / {
     ...
     log_by_lua '
         local function push_data(premature, uri, args, status)
             -- 使用ngx.socket.tcp或ngx.socket.udp将uri，args和status
             -- 推向远端（为了节省IO操作，你可能想在Lua中缓存数据）
         end
         local ok, err = ngx.timer.at(0, push_data,
                                      ngx.var.uri, ngx.var.args, ngx.header.status)
         if not ok then
             ngx.log(ngx.ERR, "failed to create timer: ", err)
             return
         end
     ';
 }
```

也可以创建无限重计数的定时器，比如，创建每5秒触发一次的定时器，然后在计时器的回调函数中递归调用`ngx.timer.at`。就像这样：

```lua

 local delay = 5
 local handler
 handler = function (premature)
     -- 就像cron任务一样，在Lua中处理job
     if premature then
         return
     end
     local ok, err = ngx.timer.at(delay, handler)
     if not ok then
         ngx.log(ngx.ERR, "failed to create the timer: ", err)
         return
     end
 end

 local ok, err = ngx.timer.at(delay, handler)
 if not ok then
     ngx.log(ngx.ERR, "failed to create the timer: ", err)
     return
 end
```

因为运行于后端的定时器回调及其运行时间不会附加到任何用户请求的响应时间，因此它们很容易在服务端累积，并由于编程错误或访问量过大导致耗尽系统资源。为了避免使Nginx服务崩溃这种极端后果，有一些像挂起的计时器数和Nginx工作进程中运行的计时器数等内部限制。挂起的计时器指尚未过期的定时器，运行的定时器指那些正在运行用户回调的定时器。

通过[lua_max_pending_timers](#lua_max_pending_timers)指令可以控制一个Nginx工作进程允许的最大挂起计时器数目。[lua_max_running_timers](#lua_max_running_timers)指令控制最大运行计时器数目。

按照当前的实现，每个运行的定时器会在全局连接记录表中取一个（假）连接记录，全局连接记录表是在`nginx.conf`中通过[worker_connections](http://nginx.org/en/docs/ngx_core_module.html#worker_connections)指令配置的。所以，需要保证[worker_connections](http://nginx.org/en/docs/ngx_core_module.html#worker_connections)指令设置了一个足够大的值（包括真实的连接和虚假连接）。

在计时器回调的上下文中可以使用绝大多数的Nginx Lua API，比如，流/数据报cosocket（[ngx.socket.tcp](#ngxsockettcp)和[ngx.socket.udp](#ngxsocketudp)），共享内存字典（[ngx.shared.DICT](#ngxshareddict)），用户协程（[coroutine.*](#coroutinecreate)），用户轻线程（[ngx.thread.*](#ngxthreadspawn)），[ngx.exit](#ngxexit)，[ngx.now](#ngxnow)/[ngx.time](#ngxtime)，[ngx.md5](#ngxmd5)/[ngx.sha1_bin](#ngxsha1_bin)等都是允许的。但子请求API（比如[ngx.location.capture](#ngxlocationcapture)），[ngx.req.*](#ngxreqstart_time)系列API，下游输出API（比如[ngx.say](#ngxsay)，[ngx.print](#ngxprint)，[ngx.flush](#ngxflush)等）是在该上下文中显示禁用的。

你可以向计时器回调函数传递大多数的标准Lua值（nil，boolean，number，string，table，closure，文件句柄等），或者显示地以用户参数的形式，或者隐式地作为回调closure的上值。这里有一些例外：**不能**传递函数[coroutine.create](#coroutinecreate)，[ngx.thread.spawn](#ngxthreadspawn)返回的线程对象，或者[ngx.socket.tcp](#ngxsockettcp)，[ngx.socket.udp](#ngxsocketudp)和[ngx.req.socket](#ngxreqsocket)返回的cosocket对象，因为当定时器回调与创建请求的上下文脱离，并运行在自身请求（假的）上下文时，它们的生命周期与创建它们的请求上下文相关。如果你试图在请求间共享线程体活cosocket对象，你会得到“no co ctx found"错误（对于线程来说）或者"bad request"错误（对于cosocket来说）。但是，在定时器回调中创建这些对象是可以的。

该API首次出现在`v0.8.0`版本中。

[回到目录](#nginx-api-for-lua)

ngx.config.debug
----------------
**syntax:** *debug = ngx.config.debug*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua*, init_worker_by_lua**

This boolean field indicates whether the current Nginx is a debug build, i.e., being built by the `./configure` option `--with-debug`.

This field was first introduced in the `0.8.7`.

[回到目录](#nginx-api-for-lua)

ngx.config.prefix
-----------------

**syntax:** *prefix = ngx.config.prefix()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua*, init_worker_by_lua**

Returns the Nginx server "prefix" path, as determined by the `-p` command-line option when running the nginx executable, or the path specified by the `--prefix` command-line option when building Nginx with the `./configure` script.

This function was first introduced in the `0.9.2`.

[回到目录](#nginx-api-for-lua)

ngx.config.nginx_version
------------------------

**syntax:** *ver = ngx.config.nginx_version*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua*, init_worker_by_lua**

This field take an integral value indicating the version number of the current Nginx core being used. For example, the version number `1.4.3` results in the Lua number 1004003.

This API was first introduced in the `0.9.3` release.

[回到目录](#nginx-api-for-lua)

ngx.config.nginx_configure
--------------------------

**syntax:** *str = ngx.config.nginx_configure()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua**

This function returns a string for the NGINX `./configure` command's arguments string.

This API was first introduced in the `0.9.5` release.

[回到目录](#nginx-api-for-lua)

ngx.config.ngx_lua_version
--------------------------

**syntax:** *ver = ngx.config.ngx_lua_version*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua**

This field take an integral value indicating the version number of the current `ngx_lua` module being used. For example, the version number `0.9.3` results in the Lua number 9003.

This API was first introduced in the `0.9.3` release.

[回到目录](#nginx-api-for-lua)

ngx.worker.exiting
------------------

**syntax:** *exiting = ngx.worker.exiting()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua*, init_worker_by_lua**

This function returns a boolean value indicating whether the current Nginx worker process already starts exiting. Nginx worker process exiting happens on Nginx server quit or configuration reload (aka HUP reload).

This API was first introduced in the `0.9.3` release.

[回到目录](#nginx-api-for-lua)

ngx.worker.pid
--------------

**syntax:** *pid = ngx.worker.pid()*

**context:** *set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.*, init_by_lua*, init_worker_by_lua**

This function returns a Lua number for the process ID (PID) of the current Nginx worker process. This API is more efficient than `ngx.var.pid` and can be used in contexts where the [ngx.var.VARIABLE](#ngxvarvariable) API cannot be used (like [init_worker_by_lua](#init_worker_by_lua)).

This API was first introduced in the `0.9.5` release.

[回到目录](#nginx-api-for-lua)

ndk.set_var.DIRECTIVE
---------------------
**syntax:** *res = ndk.set_var.DIRECTIVE_NAME*

**context:** *init_worker_by_lua*, set_by_lua*, rewrite_by_lua*, access_by_lua*, content_by_lua*, header_filter_by_lua*, body_filter_by_lua*, log_by_lua*, ngx.timer.**

This mechanism allows calling other nginx C modules' directives that are implemented by [Nginx Devel Kit](https://github.com/simpl/ngx_devel_kit) (NDK)'s set_var submodule's `ndk_set_var_value`.

For example, the following [set-misc-nginx-module](http://github.com/openresty/set-misc-nginx-module) directives can be invoked this way:

* [set_quote_sql_str](http://github.com/openresty/set-misc-nginx-module#set_quote_sql_str)
* [set_quote_pgsql_str](http://github.com/openresty/set-misc-nginx-module#set_quote_pgsql_str)
* [set_quote_json_str](http://github.com/openresty/set-misc-nginx-module#set_quote_json_str)
* [set_unescape_uri](http://github.com/openresty/set-misc-nginx-module#set_unescape_uri)
* [set_escape_uri](http://github.com/openresty/set-misc-nginx-module#set_escape_uri)
* [set_encode_base32](http://github.com/openresty/set-misc-nginx-module#set_encode_base32)
* [set_decode_base32](http://github.com/openresty/set-misc-nginx-module#set_decode_base32)
* [set_encode_base64](http://github.com/openresty/set-misc-nginx-module#set_encode_base64)
* [set_decode_base64](http://github.com/openresty/set-misc-nginx-module#set_decode_base64)
* [set_encode_hex](http://github.com/openresty/set-misc-nginx-module#set_encode_base64)
* [set_decode_hex](http://github.com/openresty/set-misc-nginx-module#set_decode_base64)
* [set_sha1](http://github.com/openresty/set-misc-nginx-module#set_encode_base64)
* [set_md5](http://github.com/openresty/set-misc-nginx-module#set_decode_base64)

For instance,

```lua

 local res = ndk.set_var.set_escape_uri('a/b');
 -- now res == 'a%2fb'
```

Similarly, the following directives provided by [encrypted-session-nginx-module](http://github.com/openresty/encrypted-session-nginx-module) can be invoked from within Lua too:

* [set_encrypt_session](http://github.com/openresty/encrypted-session-nginx-module#set_encrypt_session)
* [set_decrypt_session](http://github.com/openresty/encrypted-session-nginx-module#set_decrypt_session)

This feature requires the [ngx_devel_kit](https://github.com/simpl/ngx_devel_kit) module.

[回到目录](#nginx-api-for-lua)

coroutine.create
----------------
**syntax:** *co = coroutine.create(f)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, init_by_lua*, ngx.timer.*, header_filter_by_lua*, body_filter_by_lua**

Creates a user Lua coroutines with a Lua function, and returns a coroutine object.

Similar to the standard Lua [coroutine.create](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.create) API, but works in the context of the Lua coroutines created by ngx_lua.

This API was first usable in the context of [init_by_lua*](#init_by_lua) since the `0.9.2`.

This API was first introduced in the `v0.6.0` release.

[回到目录](#nginx-api-for-lua)

coroutine.resume
----------------
**syntax:** *ok, ... = coroutine.resume(co, ...)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, init_by_lua*, ngx.timer.*, header_filter_by_lua*, body_filter_by_lua**

Resumes the executation of a user Lua coroutine object previously yielded or just created.

Similar to the standard Lua [coroutine.resume](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.resume) API, but works in the context of the Lua coroutines created by ngx_lua.

This API was first usable in the context of [init_by_lua*](#init_by_lua) since the `0.9.2`.

This API was first introduced in the `v0.6.0` release.

[回到目录](#nginx-api-for-lua)

coroutine.yield
---------------
**syntax:** *... = coroutine.yield(...)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, init_by_lua*, ngx.timer.*, header_filter_by_lua*, body_filter_by_lua**

Yields the executation of the current user Lua coroutine.

Similar to the standard Lua [coroutine.yield](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.yield) API, but works in the context of the Lua coroutines created by ngx_lua.

This API was first usable in the context of [init_by_lua*](#init_by_lua) since the `0.9.2`.

This API was first introduced in the `v0.6.0` release.

[回到目录](#nginx-api-for-lua)

coroutine.wrap
--------------
**syntax:** *co = coroutine.wrap(f)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, init_by_lua*, ngx.timer.*, header_filter_by_lua*, body_filter_by_lua**

Similar to the standard Lua [coroutine.wrap](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.wrap) API, but works in the context of the Lua coroutines created by ngx_lua.

This API was first usable in the context of [init_by_lua*](#init_by_lua) since the `0.9.2`.

This API was first introduced in the `v0.6.0` release.

[回到目录](#nginx-api-for-lua)

coroutine.running
-----------------
**syntax:** *co = coroutine.running()*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, init_by_lua*, ngx.timer.*, header_filter_by_lua*, body_filter_by_lua**

Identical to the standard Lua [coroutine.running](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.running) API.

This API was first usable in the context of [init_by_lua*](#init_by_lua) since the `0.9.2`.

This API was first enabled in the `v0.6.0` release.

[回到目录](#nginx-api-for-lua)

coroutine.status
----------------
**syntax:** *status = coroutine.status(co)*

**context:** *rewrite_by_lua*, access_by_lua*, content_by_lua*, init_by_lua*, ngx.timer.*, header_filter_by_lua*, body_filter_by_lua**

Identical to the standard Lua [coroutine.status](http://www.lua.org/manual/5.1/manual.html#pdf-coroutine.status) API.

This API was first usable in the context of [init_by_lua*](#init_by_lua) since the `0.9.2`.

This API was first enabled in the `v0.6.0` release.

[回到目录](#nginx-api-for-lua)

Obsolete Sections
=================

This section is just holding obsolete documentation sections that have been either renamed or removed so that existing links over the web are still valid.

[回到目录](#table-of-contents)

Special PCRE Sequences
----------------------

This section has been renamed to [Special Escaping Sequences](#special-escaping-sequences).

