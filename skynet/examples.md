# examples

Skynet 提供的一些示例。

## main

首先看到下面这个例子：

```
$ ./skynet examples/config
$ ./3rd/lua/lua examples/client.lua
```

### [config](https://github.com/cloudwu/skynet/blob/master/examples/config)

```lua
include "config.path"

thread  = 8   -- 启动8个工作线程
harbor  = 1   -- 节点编号
logger  = nil -- 输出到标准输出
logpath = "." -- 日志的保存目录

address    = "127.0.0.1:2526" -- 当前 skynet 节点的 IP 地址和端口
standalone = "0.0.0.0:2013"   -- 如果把这个 skynet 进程作为主进程启动，那么需要配置这一项，表示该进程是主节点，它需要开启一个控制中心，以供其他节点接入
master     = "127.0.0.1:2013" -- 指定 skynet 控制中心的 IP 地址和端口，如果配置了 standalone 项，那么这一项通常与 standalone 项相同

bootstrap  = "snlua bootstrap" -- skynet 启动的第一个服务及其启动参数
start      = "main"            -- bootstrap 最后一个环节将启动的 lua 服务，也就是定制的 skynet 节点的主程序

cpath      = root.."cservice/?.so" -- 用 C 编写的服务模块的位置
```

### [config.path](https://github.com/cloudwu/skynet/blob/master/examples/config.path)

```lua
root = "./"

-- lua 服务代码所在的位置
luaservice = root.."service/?.lua;" .. root.."test/?.lua;" .. root.."examples/?.lua;" .. root.."test/?/init.lua"
-- 用哪一段 lua 代码加载 lua 服务
lualoader  = root.."lualib/loader.lua"
-- 将添加到 package.path 中的路径，供 require 调用
lua_path   = root.."lualib/?.lua;" .. root.."lualib/?/init.lua"
-- 将添加到 package.cpath 中的路径，供 require 调用
lua_cpath  = root.."luaclib/?.so"

-- 用 snax 框架编写的服务的查找路径
snax = root.."examples/?.lua;" .. root.."test/?.lua"
```

### [main.lua](https://github.com/cloudwu/skynet/blob/master/examples/main.lua)

### [client.lua](https://github.com/cloudwu/skynet/blob/master/examples/client.lua)
