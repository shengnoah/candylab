---
layout: page-fullwidth
title: "Blues框架"
subheadline: "How to use Feeling Responsive"
teaser: "The documentation is a work in progress..."
categories:
  - documentation
permalink: "/blues/"
header:
   image_fullwidth: "header_roadmap_2.jpg"
---



<div class="row">
<div class="medium-4 medium-push-8 columns" markdown="1">
<div class="panel radius" markdown="1">
**Table of Contents**
{: #toc }
*  TOC
{:toc}
</div>
</div><!-- /.medium-4.columns -->


<div class="medium-8 medium-pull-4 columns" markdown="1">

{% include _improve_content.html %}



## 1.Blues的Request库的使用

在说到http-src这个插件时，说过要用blues框架的nginx库来取得用户请求的数据，现在我
们就看看这两个库的的实现的，简要的说明，因为在介绍LazyTable时，专门说了nginx.lua
的实现，而request库是直接调用blues的nginx来取得用户数据，所以我们先看blues的request
是如何使用的,httpsrc插件的使用方法一样。


request库的代码实现很简单，从Blues使用Request库来说明。

在blues框架中创建一个GET匿名方法，用于测试request功能,代码如下：


```lua
app:get("/blockip", function(request,id)
    ngx.say(request.params['cmd_url'])
end)
```

启动Openresty，测试一下这个路由过程：

```
hi start
curl 0.0.0.0/blockip
```

显示结果：
```
/blockip
```


调用时序是：app->request->nginx

匿名函数很简单，我们观察一下request如何让自己的params结构中添充数据的，这数据的
取得就是来至于nginx的lazytable实现，代码如下：


```lua
local params = require "nginx"
local Request = {}

function Request.getInstance()

        local name = "request"
        local instance = { 
                            getName = function() end 
                        }   

        instance.params = params
        setmetatable(instance, { __index = self,
                                 __call = function()
                                        end 
                                 })  
        return instance
end

return Request
```

可能说Request简直就是nginx一层简单的封装，最主要的一句就是：

```lua
instance.params = params
```
就是这句话调用LazyTable实现了数据的初始化，让request.params中有了用户请求数据。

关于nginx.lua库实现不展开说，之前有说明：[使用LazyTable在Openresty中取得用户请求信息](https://www.candylab.net/lazytable-and-request/)

nginx.lua的如何实现不展开，但是目前nginx.lua支持返回那些用户请求数据可查看数据
结构，如下：


```lua
local ngx_request = {
  headers = function()
    return ngx.req.get_headers()
  end,
  cmd_meth = function()
    return ngx.var.request_method
  end,
  cmd_url = function()
    return ngx.var.request_uri
  end,
  body = function()
    ngx.req.read_body()
    local data = ngx.req.get_body_data()
    return data
  end,
  content_type = function()
    local content_type = ngx.header['content-type']
    return content_type
  end
}
```

总结，以下：

```lua
params['header']
params['cmd_meth']
params['cmd_url']
params['body']
params['content_type']
```

使用nginx.lua库的例子：

```lua
local params = require "nginx"
url = params['cmd_url']
```

我们直接用引用后，取值即可，在httpsrc插件中也是这么用：


```lua
function httpsrc_plugin.action(self, stream)
    local params = require "nginx"
    self.sink['request']['url'] = params['cmd_url']
end
```


以上就是如何在httpsrc插件中使用blues的nginx.lua库取得用户请求数据。




## 2.Blues框架引入Pipeline模式做插件系统

---
Blues框架基本是作为Wario项目的一个公共库存在， 代码量很少，我们试着在Blues中也引入
Pipline模式的插件管理系统，这样意着可以控制框架各种功能的开关。

之前的代码比较啰嗦，我们重构了代码：

blues.lua

Blues类，就是过去的Application类，我们把匿名函函数的形参变了， 变成了app数据结结构本身作为参数。

```lua
local Route = require("proute")
local Request = require("request")

local Blues = {}

Blues.blues_id = 1 

function Blues.new(self, lib)
        app.app_id = 1 
        app.router = Route:getInstance()
        app.req = Request:getInstance()

        app.get = function(self, url, callback)
                app:router(url, callback, "GET")
        end 

        app.post = function(self, url, callback)
                app:router(url, callback, "POST")
        end 

        app.run = function(self)
                fun = Route:run(app.router)
                if fun then
                    local ret = fun(app)
                    local rtype = type(ret)
                    if rtype == "table"  then
                        json = require "cjson"
                        ngx.header['Content-Type'] = 'application/json; charset=utf-8'
                        ngx.say(json.encode(ret))
                    end 
                    if rtype == "string"  then
                        ngx.header['Content-Type'] = 'text/plain; charset=UTF-8'
                        ngx.say(ret)
                    end 
                end 
        end 

        return app 
end

return Blues:new {
    pipeline='Pipeline System.'
}


```

Pipeline组件的位置就是代码，如下：

```lua
return Blues:new {
    pipeline='Pipeline System.'
}
```

这就是未来的放插件的位置：


proute.lua

简单路由基上没变动，除了给函数加了self形参。


```lua
local tinsert = table.insert
local req = require "nginx"

local Route =  {}

function Route.getInstance(self)
        local instance = {}
        instance.map = {
            get = {},   --get
            post = {}   --post
        }   

        instance.idea = 1 

        local router = {}
        function router.register(self, app, url, callback, method)
                if method == "GET" then
                    tinsert(self.map.get, {url, callback})
                elseif method == "POST" then
                    tinsert(self.map.post, {url, callback})
                end 
        end 

        router.__call = router.register
        setmetatable(instance, router)
        return instance
end

function Route.run(self, router)

        local url = req.cmd_url
        local method = req.cmd_meth

        if method == "POST" then
                for k,v in pairs(router.map.post) do
                        if router.map.post[k][1] == url then
                                return router.map.post[k][2]
                        end 
                end 
        end 

        if method == "GET" then
                for k,v in pairs(router.map.get) do
                        local match = string.find(url, router.map.get[k][1])
                        --if router.map.get[k][1] == url then
                        if match then
                                return router.map.get[k][2]
                        end
                end
        end

end

return Route
```

然后，把Wario的app.lua里的代码改成最新的形式：


```lua
local app = require "blues"

app:get("/rule", function(self)
    ngx.say(self.app_id)
    self.app_id = 6 
    ngx.say(self.app_id)
end)

return app
```

这样匿名函数可以取得所有加入到app中的库的引用。


```lua
function router.register(self, app, url, callback, method)
    if method == "GET" then
        tinsert(self.map.get, {url, callback})
    elseif method == "POST" then
        tinsert(self.map.post, {url, callback})
    end 
end 
```

router.register()函数的第二个参数，就是调用类Blues的引用，可以通过这个参数，取得所有Blues类的数据，所以
在Route类里不用引入nginx这个库了，在最外层的Blues类中定义这个成员就可以了。
```lua
local req = require "nginx"
```

如果还可以改造的话，就是这样，Blues所有非共通库，都用pipeline组织：


```lua
function Blues.new(self, lib)
end

return Blues:new {
    pipeline='Pipeline System.',
    
    run=function(self) 
        local src = { 
            metadata= { 
                data="http data",
                request = { 
                    uri="http://www.candylab.net"
                }   
            }   
        }   
        for k,v in pairs(self.element_list) do
            v:init()
            v:push(src)
            local src, sink = v:match('params')
            if type(sink) == "table" then
                self:output(sink, 0)
            end 
            src = sink
        end 
    end 

}
```

在new出blues时，以参数传入。可以把 new {} 插号中的位置，当成类声明，直接加入接口与属性数据，这个new出来的
类就是pipeline主类。

不推荐这种写法，看着就乱，独立到文件里更合适，但的确是可以这么写。



## 3.快速解析JSON


之前在使用别的语言框架时，发现路由处理json时，引用很多东西，觉得使用起来太啰嗦，
框架改的意思也不大，自己在重写一个小的框架时，就把这种实现简化了。

```lua
    app.json = function(self)
        local jsondata= self.request.params.body
        local t = self.bjson.decode(jsondata)
        return t    
    end 
```

把在匿名函数的调用封装到一个函数里。



```lua
local Blues = {}

Blues.blues_id = 1

function Blues.new(self, lib)

        local app = {}
        app.app_id = 1

        app.bjson = lib.bjson
        app.request = lib.request
        app.router = lib.router
        app.router.req = lib.nginx

        app.get = function(self, url, callback)
            app:router(url, callback, "GET")
        end

        app.post = function(self, url, callback)
            app:router(url, callback, "POST")
        end

        app.run = function(self)
            fun = app.router:finder()
            if fun then
                local content = fun(app)
                app:render(content)
            end 
        end 

        app.json = function(self)
            local jsondata= self.request.params.body
            local t = self.bjson.decode(jsondata)
            return t    
        end 
    
        return app
end

return Blues:new  {
    nginx = require("nginx"),
    bjson = require("utils.bjson"),
    request = require("request"):getInstance(),
    router = require("route"):getInstance()
}

```

Wario在调用时的形参也发生了变化：


```lua
local bjson = require "utils.bjson"
local app = require "blues"

app:get("/json", function(self)
    local ret = self.request.params.body
    local t = bjson.decode(ret)
    return t    
end)

app:get("/getjson", function(self)
    return self:json()
end)

return app 
```

## 4.POST处理方法

之前Blues的路由简单的都是用于测试GET方法的，激活一下Blues的Post方法，处理Post
匿名函数请求。



便利查找POST匿名函数表的方式和GET的一样，简单粗暴。


```lua
function Route.post(self)
    local url = self.req.cmd_url
    for k,v in pairs(self.map.post) do
        if self.map.post[k][1] == url then
            return self.map.post[k][2]
        end 
    end 
end
```


之前GET路由时序很简单，看一眼就明白了，看一POST函数处理和BLUES里和GET有什么区别：


curl调用

```
curl -X POST www.candylab.net/mypost -d '123'
```

代码在形式上真的和get没啥区别，就看self中关于post的相关的数据取得,代码如下：

```lua
app:post("/mypost", function(self)
    ngx.say("post")
end)
```

然后，我的就要处理如何在nginx.lua中解析post的请求数据。


我们尝试不在框架中解析POST函数，只在匿名函数中直接取得POST请求数据。


```lua
ngx.req.read_body()
local req = ngx.req.get_post_args()
ngx.say(req["key"])
```



</div><!-- /.medium-8.columns -->



