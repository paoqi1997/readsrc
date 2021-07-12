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
logger  = nil -- skynet_error 输出到标准输出
logpath = "." -- log 保存到指定目录

-- 当前 skynet 节点的地址和端口（即使你只使用一个节点，也需要开启控制中心，并额外配置这个节点的地址和端口）
address = "127.0.0.1:2526"

-- 如果把这个 skynet 进程作为主进程启动，那么需要配置 standalone 这一项，表示这个进程是主节点
-- 它需要开启一个控制中心，监听一个端口，让其他节点接入
standalone = "0.0.0.0:2013"
-- 指定 skynet 控制中心的地址和端口，如果配置了 standalone 项，那么 master 这一项通常与 standalone 相同
master = "127.0.0.1:2013"

bootstrap = "snlua bootstrap" -- skynet 启动的第一个服务及其启动参数
start     = "main"            -- bootstrap 最后一个环节将启动的 lua 服务，也就是你定制的 skynet 节点的主程序

cpath     = root.."cservice/?.so" -- 用 C 编写的服务模块的位置
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

```lua
local skynet = require "skynet"
local sprotoloader = require "sprotoloader"

local max_client = 64

skynet.start(function()
    skynet.error("Server start")
    skynet.uniqueservice("protoloader")
    if not skynet.getenv "daemon" then
        local console = skynet.newservice("console")
    end
    skynet.newservice("debug_console", 8000)
    skynet.newservice("simpledb")
    local watchdog = skynet.newservice("watchdog")
    skynet.call(watchdog, "lua", "start", {
        port = 8888,
        maxclient = max_client,
        nodelay = true,
    })
    skynet.error("Watchdog listen on", 8888)
    skynet.exit()
end)
```

### [protoloader.lua](https://github.com/cloudwu/skynet/blob/master/examples/protoloader.lua)

```lua
-- module proto as examples/proto.lua
package.path = "./examples/?.lua;" .. package.path

local skynet = require "skynet"
local sprotoparser = require "sprotoparser"
local sprotoloader = require "sprotoloader"
local proto = require "proto"

skynet.start(function()
    sprotoloader.save(proto.c2s, 1)
    sprotoloader.save(proto.s2c, 2)
    -- don't call skynet.exit(), because sproto.core may unload and the global slot become invalid
end)
```

### [proto.lua](https://github.com/cloudwu/skynet/blob/master/examples/proto.lua)

```lua
local sprotoparser = require "sprotoparser"

local proto = {}

proto.c2s = sprotoparser.parse [[
]]

proto.s2c = sprotoparser.parse [[
]]

return proto
```

### [simpledb.lua](https://github.com/cloudwu/skynet/blob/master/examples/simpledb.lua)

### [watchdog.lua](https://github.com/cloudwu/skynet/blob/master/examples/watchdog.lua)

### [client.lua](https://github.com/cloudwu/skynet/blob/master/examples/client.lua)
