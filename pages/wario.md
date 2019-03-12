---
layout: page-fullwidth
title: "Wario防火墙"
subheadline: "How to use Feeling Responsive"
teaser: "The documentation is a work in progress..."
categories:
  - documentation
permalink: "/wario/"
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



## 1.模拟WAF一次策略命中

WAF策略规则的式可以用多种描述形式描述，比如XML、YAML、纯文本、JSON等形式。而我们
这次模拟测试选择的规则的存储形式是JSON：

```lua
[{
  "Id":25,
  "RuleType":"args",
  "RuleItem":"(onmouseover|onerror|onload)\\="
}]
```

这种规则样式来至于XWAF的规则存储形式，在文章的最后我们提供了XWAF的相关信息，追溯
到更早是LoveShell的WAF形式，够成了这条规则的基础数，就是纯“(onmouseover|onerror|onload)\\=”
正则数据来至于loveshell的WAF项目。


完成这次模拟工作，我们需要几个大体的处理过程。

1. 生成规则文件。

2. 读取规则到Openresty的Share.Diction中。

3. 截取用户访问数据。

4. 判断计算用户请求数据是否命中策略。


### 1.生成规则文件。

为了下一步WAF功能的发展， 我们没有直接的用纯XWAF的JSON规则，而是我们在原有的基础上
加入一个新的字段来描述策略，就是Action这个字段，表示的是，策略命中是， WAF应该做出
什么动作。

ruler主要的作用是把文件中的原始转换成新的JSON数据形式，就是加了一个Action。


```lua
[{
  "Id":25,
  "RuleType":"args",
  "RuleItem":"(onmouseover|onerror|onload)\\=",
  "Action":1
}]
```

ruler.lua

```lua
local ruler = {}

function ruler.read(self, var)
    print(var)
    file = io.open(var,"r")
    if file==nil then
        return
    end
    t = {}
    for line in file:lines() do
        table.insert(t,line)
    end
    file:close()
    return(t)
end

function ruler.write(self,var, rule)
    file = io.open(var,"aw")
    if file==nil then
        return
    end

    file:write(rule,"\n")    
    file:close()
    return(t)
end

function ruler.dump(self, in_name, out_name)
    print(name)
    local ret = self:read(in_name)
    rules = {}
    local idx = 0
    for k,v in pairs(ret) do
        idx = idx + 1
        item = {Id=idx, RuleType="args", RuleItem=v, Action=1}
        table.insert(rules,item)
    end
    
    for k,v in pairs(rules) do
         for key,value in pairs(v) do
             print(key, value)
         end
    end 
    self:export(out_name, rules)
end

function ruler.export(self, filename, rule_name)
    local json = require"cjson"
    local ret = json.encode(rule_name)
    self:write(filename, ret)
end

function ruler.loading(self, filename)
    local json = require"cjson"
    local util = require "cjson.util"
    local ret = util.file_load(filename, env)
    local data = json.decode(ret)
    print(util.serialise_value(data))
end


return ruler 

```

测试如何用旧的JSON生成新的JSON。
test.lua

```lua
local ruler = require"ruler"
ruler::export("args.rule", "testcase")
```



### 2.读取规则到Openresty的Share.Diction中。

这部分的内容，我们是先用Blues正常的GET访问，模拟数据的加载，然后再把测试成功的代码
移到Init.lua中，把Content阶段执行的代码，放到Init中。

app.lua

```lua
app:get("/xwaf", function(request,id)
    local json_text = bjson.loadf("./app/data/rules/args.rule", env)
    local t = bjson.decode(json_text)

    buffer.sett("args", t)
    meta = buffer.gett("args")
    ngx.say(bjson.pprint(meta))
end)


app:get("/ltbl", function(request,id)
    local json_text = {Id=25, RuleType="args", RuleItem="(onmouseover|onerror|onload)\\="}
    local t = bjson.decode(json_text)

    buffer.sett("args", t)
    meta = buffer.gett("args")
    ngx.say(bjson.pprint(meta))
end)
```

XWAF有不单一个rules文件，我们选中了其中的args这个文件进行了读取，下面我们要在init.lua中读取这些数据，在init阶段读取，在content阶段的app.lua中读取这个args结构：

init.lua
```lua
local buffer = require "buffer"
local bjson = require "utils.bjson"

local json_text = bjson.loadf("./app/data/rules/testcase", env)
local t = bjson.decode(json_text)
buffer.sett("rule", t)
buffer.set("candylab", "Candylab:Blues")



--这是针对读取args规则的三句新加代码
local json_text = bjson.loadf("./app/data/rules/args.rule", env)
local t = bjson.decode(json_text)
buffer.sett("init_args", t)
```


### 3.截取用户访问数据。

在Init阶段如果我们已经将rule数据加入字典的话，在blues框架只要简单的访问一下字段中的数据就可以。

app.lua
```lua
app:get("/xwaf_rules", function(request,id)
    meta = buffer.gett("init_args")
    ngx.say(bjson.pprint(meta))
end)
```




我们要做数据过滤，而且是基于正则的，所以在项目最开始阶段，直接引入了XWAF的规则文件。在content阶段的读取的这些数据，在init阶段同样可以读取。

而下面的数据碰撞就是针对这个args规则进行演示的,我们继续在content阶段，用一个GET方法请求模拟这个WAF规则命中的过程，不选POST而选GET，因为GET取参数简单，便于集中经历说明规则进行简单的比较，而不是把重点放在解析POST过来的参数和内容上。


```lua
app:get("/greatball", function(request,id)
    meta = buffer.gett("init_args")
    ngx.say(bjson.pprint(meta))
    ngx.say(request.params.cmd_url)
end)
```

### 4.判断计算用户请求数据是否命中策略。

我们创建一个有greatball的请求，我们简单的模拟一些这个请求数据与WAF规则对比的过程。
在这个方法里，我们同时取得了,url和args这个规则所有的数据，下面就是按什么样的方式
进行数据配对了。


下面是代码实现，我们模拟的是在GET请求到的路由处理函数，直接读取规则，然后和用户
提供的请求从比较。

```lua
app:get("/testargs", function(request,id)
    local json_text = bjson.loadf("./app/data/rules/args.rule", env)
    local ret = bjson.decode(json_text)
    for _,rule in pairs(ret) do
        regular = rule["RuleItem"]
        if regular ~= "" and ngx.re.find(request.params.cmd_url, regular, "jo") then
            ngx.say("MATCH!")
        end 
    end 
end)
```


下个实验，我们会把这种策略命中的处理代码，独立成一个单独的功能文件，脱离Blues的路由系统,后 可以直接把这部分过滤用的代码做为Blues自己的安全审计模块，那就是下一篇要介绍的内容。



XWAF是一款开源的WAF产品，详细的介绍大家参考: [开源WEB防火墙XWAF介绍](https://www.openresty.com.cn/X-WAF-README.html) ，作者的官方的站点：[XWAF官网](https://waf.xsec.io/)。



## 2.WAF分组安全策略匹配

上次我们做了一个WAF系统策略命中的模拟，这次新实验将要加入更复杂的集团分组策略匹配。简单说就是加入了更多更复杂的策略，为了方便演示，重新组织了策略规则的存储形式，我们用loadstring的方式，加载规则文件。实现的代码，如下：


```lua
local metadata = [[
    local args = {
        {Id=1, RuleType="args", RuleItem="\.\./", action=1},
        {Id=2, RuleType="args", RuleItem="\:\$", action=1},
        {Id=3, RuleType="args", RuleItem="\$\{", action=1},
        {Id=4, RuleType="args", RuleItem="select.+(from|limit)", action=1},
        {Id=5, RuleType="args", RuleItem="(?:(union(.*?)select))", action=1},
        {Id=6, RuleType="args", RuleItem="having|rongjitest", action=1},
        {Id=7, RuleType="args", RuleItem="sleep\((\s*)(\d*)(\s*)\)", action=1},
        {Id=8, RuleType="args", RuleItem="benchmark\((.*)\,(.*)\)", action=1},
        {Id=9, RuleType="args", RuleItem="base64_decode\(", action=1},
        {Id=10, RuleType="args", RuleItem="(?:from\W+information_schema\W)", action=1},
        {Id=11, RuleType="args", RuleItem="(?:(?:current_)user|database|schema|connection_id)\s*\(", action=1},
        {Id=12, RuleType="args", RuleItem="(?:etc\/\W*passwd)", action=1},
        {Id=13, RuleType="args", RuleItem="into(\s+)+(?:dump|out)file\s*", action=1},
        {Id=14, RuleType="args", RuleItem="group\s+by.+\(", action=1},
        {Id=15, RuleType="args", RuleItem="xwork.MethodAccessor", action=1},
        {Id=16, RuleType="args", RuleItem="(?:define|eval|file_get_contents|include|require|require_once|shell_exec|phpinfo|system|passthru|preg_\w+|execute|echo|print|print_r|var_dump|(fp)open|alert|showmodaldialog)\(", action=1},
        {Id=17, RuleType="args", RuleItem="xwork\.MethodAccessor", action=1},
        {Id=18, RuleType="args", RuleItem="(gopher|doc|php|glob|file|phar|zlib|ftp|ldap|dict|ogg|data)\:\/", action=1},
        {Id=19, RuleType="args", RuleItem="java\.lang", action=1},
        {Id=20, RuleType="args", RuleItem="\$_(GET|post|cookie|files|session|env|phplib|GLOBALS|SERVER)\[", action=1},
        {Id=21, RuleType="args", RuleItem="\<(iframe|script|body|img|layer|div|meta|style|base|object|input)", action=1},
        {Id=22, RuleType="args", RuleItem="(onmouseover|onerror|onload)\=", action=1},
    }
    
    local urls = {
        {Id=1, RuleType="url", RuleItem="\.(htaccess|bash_history)", action=1},
        {Id=2, RuleType="url", RuleItem="\.(bak|inc|old|mdb|sql|backup|java|class|tgz|gz|tar|zip)$", action=1},
        {Id=3, RuleType="url", RuleItem="(phpmyadmin|jmx-console|admin-console|jmxinvokerservlet)", action=1},
        {Id=4, RuleType="url", RuleItem="java\.lang", action=1},
        {Id=5, RuleType="url", RuleItem="\.svn\/", action=1},
        {Id=6, RuleType="url", RuleItem="/(attachments|upimg|images|css|uploadfiles|html|uploads|templets|static|template|data|inc|forumdata|upload|includes|cache|avatar)/(\\w+).(php|jsp)", action=1},
    }
    
    local useragent = {
        {Id=1, RuleType="useragent", RuleItem="(HTTrack|harvest|audit|dirbuster|pangolin|nmap|sqln|-scan|hydra|Parser|libwww|BBBike|sqlmap|w3af|owasp|Nikto|fimap|havij|PycURL|zmeu|BabyKrokodil|netsparker|httperf|bench)", action=1},
    }
    
    return {args=args, urls=urls, cookie=cookie, useragent=useragent, post=post}
]]


local script = metadata 
local rules = assert(loadstring(script))()
return rules
```

我们通过最直接的可以被lua认识的语法方式，存储这些策略规则，接下来的任务就是将各种场景下的配对处理抽象成代码模块，数据结构已经定义完了， 看如何对这些数据进行操作。


```lua
local rules = require"meta"

local matcher = {}

function matcher.init(self)
    self.action_id = {"404","500","301"}
    self.match_map = { 
        matcher_group_get = {args=0, urls=0, cookie=0, useragent=0, post=0 },
        matcher_group_post= {args=0, urls=0, cookie=0, useragent=1, post=0 },
        matcher_group_whiteip= {args=0, urls=0, cookie=0, useragent=0, post=0 }
    }   
    
    self.action_seq =  {   
        {id=1, action="start", method = "GET"} ,
        {id=2, action="start", method = "POST"}, 
        {id=3, action="start", method = "WHITEIP"} 
    }   
    
    self.method_id = {GET="matcher_group_get", POST="matcher_group_post", WHITEIP="matcher_group_whiteip"}
end

function matcher.action(self, m_type, param)
    local group_tbl= self.match_map[m_type]
    for k,v in pairs(group_tbl) do
        if v == 1 then
            print(k)
            for key, val in pairs(rules[k]) do
                local regular = val["RuleItem"]
                if regular ~= "" and ngx.re.find(request.params.cmd_url, regular, "jo") then
                    local id = val["action"]
                    print(self.action_id[id])
                end 
            end 
        end 
    end 
end

function matcher.match(self, param)
    matcher:init()
    for k,v in pairs(self.action_seq) do
        local idx = v['method']
        local method = self.method_id[idx]
        matcher:action(method, "param")
    end 
end

matcher:match("param")
```

我们对比一下之前在Blues框架里的模拟处理的代码，如下：

```lua
app:get("/testargs", function(request,id)
    local json_text = bjson.loadf("./app/data/rules/args.rule", env)
    local ret = bjson.decode(json_text)
    for _,rule in pairs(ret) do
        regular = rule["RuleItem"]
        if regular ~= "" and ngx.re.find(request.params.cmd_url, regular, "jo") then
            ngx.say("MATCH!")
        end 
    end 
end)
```

经过数据结构模块的分离操作，我们把所有放在文件里的策略，变成了Lua的Table数据结构。然后将可能发生的用户请个场景，与在特定场景下如何进行配对进行了分层Pattern处理。

1. 确认用户当前请求的场景: GET、POST、PUT等。
2. 确人在特点方法下，将用户的请求的数据：URL、URI、参数、Cookie与那些策略进行配对。（args=1, urls=0, cookie=0, useragent=0, post=0 ）[1:与该组策略匹配。 0:不进行匹配]
3. 如果策略命中，进行预先定义好的Action响应操作。（上面的代码以print(self.action_id[id])来表示实行动作）


到此，一个较为复杂的策略命中的判断处理已经实现了， 这模块可以单成一个WAF基础项目，也可以作为框加基本的Filter模块，用于过滤用户请求中的非法数据。

下面的课题是，如何动态的维护和编辑这些策略，是采用命令行的方式，还是采用WEB界面的方式。如何将Action的拦截响应动作，交给Openresty来处理，而这些我们在尝试新的重构变化。




## 3.WAF策略规则插件化

在过去，我们实现了一个最小化的WAF规则分组配对，这种方案的好处就可以集中维护规则，不好的地方
也比较明显，维护改动一次规则，需要影响其它不相关的策略规则，针对这个问题，我们将规则按分类进行
解耦，进行插件化的维护这个策略，一个插件有自己独立的管理的规则。

对于这些插件模块的输入，可以理解为固定的输入，即所有Openresty可以看见的用户请求数据，这样的话，
插件就可以抽像出一些共通的行为与属性，因为Lua没有虚函数和接口的概念，我们直接用table变量和函数
进行简单的模拟。



### 1.上层抽象解耦

从目际构成来看，我们在 app所在目录创建两个子目录:plugin、rules。

在app的根目录创建两个文件:plugin_factory.lua、 plugin_config.lua。


我们按照至顶向下的的方式阶段这些层级文件。



plugin_factory.lua

```lua
local plugin_list = require "plugin_config"

local plugin_factory = {}

function plugin_factory.start(params)
    for k,v in pairs(plugin_list) do
        v:match('params')
    end 
end

return plugin_factory
```

上层的文件设计相对稳定，代码基本不会常变，将插件列表中所有插件循环遍历调用，然后将相对的输入传递给这些插件。

```lua
local factory = require"plugin_factory"
factory.start(params)
```
params中传的数值都是通过Openresty的API进行取得的。


plugin_config.lua

```lua
local xss_plugin = require"plugin.xss_plugin"
local sql_plugin = require"plugin.sql_plugin"
local cc_plugin = require"plugin.cc_plugin"

local plugin_list= { 
    xss=xss_plugin,
    sql=sql_plugin,
    cc=cc_plugin
}

return plugin_list
```

我们通过plugin_config这个封装，模拟接口类的批量生成，所有工作的插件都会在plugin_list里行注册，之后plugin_factory
才会知道这个插件的存在，然后插件的预定的函数会被依次的调用。


### 2.插件接口实现


接下来，我们看一下，一个接口是如何被定义的：

xss_plugin.lua

```lua
local xss_rules = require "rules.xss_rules"
local xss_plugin = {}

function xss_plugin.init(self)
    self.rules=xss_rules
end

function xss_plugin.action(self, param) 
    print("xss_plugin:action")
end

function xss_plugin.match(self, param)
    self:init()
    print("xss_plugin:match")
    for k, v in pairs(self.rules) do
        for _,value in pairs(v) do
            print(value)
           if regular ~= "" and ngx.re.find(params, regular, "jo") then
           end 
        end 
    end 
end

return xss_plugin
```

一个插件内部，规定了三个方法，其中match()这个方法是必须要定义的有，因为上个层级plugin_factory
会统一的调用这个方法。

init()的主要作用是读取rules文件中的数据结构。action()这个函数是比较具体的实现逻辑，你也可以省略
这个函数的实现，直接在match()函数中进行比较，对于一遍的简单逻辑，不分开也可以，对一些比较复杂的
比较插件逻辑，还是解开比较清晰。


最后，看一次rules目录的文件是如何实现的， 拿xss_plugin对应的rules文件为例：


### 3.规则定义

xss_rules.lua

```lua
local urls = { 
    {Id=1, RuleType="url", RuleItem="\.(htaccess|bash_history)", action=1},
    {Id=2, RuleType="url", RuleItem="\.(bak|inc|old|mdb|sql|backup|java|class|tgz|gz|tar|zip)$", action=1},
    {Id=3, RuleType="url", RuleItem="(phpmyadmin|jmx-console|admin-console|jmxinvokerservlet)", action=1},
    {Id=4, RuleType="url", RuleItem="java\.lang", action=1},
    {Id=5, RuleType="url", RuleItem="\.svn\/", action=1},
    {Id=6, RuleType="url", RuleItem="/(attachments|upimg|images|css|uploadfiles|html|uploads|templets|static|template|data|inc|forumdata|upload|includes|cache|avatar)/(\\w+).(php|jsp)", action=1},
}
return urls 
```

这个结构，和之前fileter.lua中提供的rules是子集关系，我们通过这种方式进行解耦。


## 4.Openresty基于Pipeline模式的WAF模块构建


一直以来主要考虑点是，不要把代码写乱套了。如何拆分和给组织模块，以什么形式传送数据变成了一个手艺。解耦最直接的方法是分层，先把数据与为业分开，再把业务代码和共通代码分开。数据层对我们系统来说就是规则，系统使用的共通代码都封装到了框架层，而系统功能业务共通的部分，以插件为机能单位分开并建立联系。
数据是面象用户的，框架是面向插件开发者的， 插件的实现就是机能担当要做的事情，不同的插件组合相对便捷的生成新机能，也是插件便利的益处与存在的意义。


![system](https://www.candylab.net/assets/img/projects/wario/pipeline_system_design.png)


### 1.为什么是Pipeline

从宏观的角度讲，设计师设计大楼，是一个构建美好城市的过程。从微观角度讲，工程师码砖盖楼和水泥也是建设国家。之前我们把WAF的过程化编码，通过插件的方式解耦，让系统相对更好的维护和添加新功能，而从设计的角度看，不从实装的细节来说，我们更倾向于用一个更抽象的语言符号系统来描述我们的问题。

现在我们就要用一个被广泛使用的一个比喻，或者说是概念来描述我们的插件：pipeline。

pipeline和stream有着千丝万缕的关系，也用很多的系统使用pipeline来描述软件系统中某些数据和程序处理，可以说pipeline是一个现实中可寻可见的东西， 也可以说是一种设计模式的应用实例化。

简单说吧，这些给Openresty WAF说明使用的Pipeline灵感来源于Graylog流日志处理和Gstreamer的视频流处理类似的Pipeline的概念，如果再有其他雷同，可能就是抄一块去了。


简单来说，基于Openresty的WAF，不考虑效率的问题是话，可以把WAF看成一个大的字符串过滤程序，从Openresty提供的Lua接口和API读取HTTP数据，然后对数据的按照一定的规则进行字符串查找，字符串正则查找。而我们对这个字符串查找程序进行了一次相对更一级的问题抽象：插件化。然而，被归为插件的程序也不是一成不变的，我们就从Pipeline的概念出发，看看可以把Openresty WAF的插件大体分几类，又如何在之后时行更进一步的细化。


### 2.插件分类

流数据与时间有强关联的，比如视频，比如用户操作行为的日志，都是时序相关的。而对一个WEB服务器来说，用户的ＨＴＴＰ请求过程，也分几个阶段。OpenResty就是典型的分出了几个阶段。

在OpenResty社区有一个非常典型的图，来说明这个问题,如下：

![OR](http://ww4.sinaimg.cn/mw690/6d579ff4gw1f3wljbt257j20rx0pa77c.jpg)

在Gstreamer流媒体框架中，除了核心的框架，几乎所有机能都被用各种类型的插件来实现，我们也基于这种考虑，把一些可以被抽出的公通模块以插件为单位来实现，如果你觉这部分的功的代码写的不好，原则上可以自己动手，重构这部分功能的插件。


下面就是一个简单的WAF的实现思路，如果用插件的单位来组强划分，需要用那些插件来构成一个Pipeline， 管道其实就是插件的串或是并行的组合，构成的一个处理流数据的程序组织结构，pipeline是用来组织划分插件程序模块的， 不是组强插件数据的，而数据就是典型的Input。


#### 2.1 Pipeline 

一个Pipeline由若干个插件构成，每个插件都有数据插槽src、sink。


#### 2.2 Slot

一个插件如果有sink插槽，说明这个插件是可以接受数据入力的，接受其它插件传过来的数据。如果一个插件有src插槽，说明这个插件可以产生出力数据，给其它的插件使用。

关于Pipeline，我们先轻描淡写的说一下，重一点说一下，一个简单的WAF需要那些式样的插件来组合，完成WAF的功能，关键的，这些插件都在OpenResty的那些阶段工作。


事前说一下，share diction算一个共享区域，不算一个可以对接的插件之间的src或是sink数据协议， Slot插槽本身就定义了一个插件输入输出的数据协议，而SC不是，SC是独立于插件的共享区，SC用不好就像全局变理一样，破坏插件间的耦合性。


### 3.插件分类

总体上，我们将插件分成三类：框架插件、业务插件、其它插件。

框架插件：就是驱动业务插件跑起来的，必须有的公通代码。

业务插件：比如XSS、SQL注入这种。

其它插件：没法分类的，没法提取“公约数”的代码。



从童年时期，对两个工作就有一些好奇，一个是木匠、另外一个是烧锅炉的，是相当有技术含量的技术工种，特别是烧锅炉，多危险，烧过了浪费煤，烧少了，不热有人投诉。

随着无声岁月的变化，时代慢慢淡化了木匠这个职业，如果有木匠这一技之长，谁家要是做个窗口凳子什么的，回头不给工钱，说不准，还能送个烧鸡什么的，也不是说非得要点什么，有人送烧鸡也是对木匠这个技术工作的一种认可，技术本应受到人们的重视，现在我们中的很多人都做了程序员， 有人送烧鸡距离我们越来越远，但有时我们又感慨，毕竟程序员是一个改变世界的职业，你们今天被勒索软件勒索了吗，威胁支付比特币了没有？




### 4. 最小化WAF的Pipeline图示


![管道元素](https://www.candylab.net/assets/img/projects/wario/pipeline_elements.png)

我们来看看最小化的Pipeline是什么样的？ 封面上的那个图，其实就是最小化的一个WAF的Pipeline模型。很简单的三个插件，我们来看看这三个插件在Openresty lua执行过程中所处阶段，和这三个的插件应该如何的实现。
之前，我们都是实现一个简易的lua框架来模拟WAF工作的流程， 这简易的框架在WAF构建过程中，主要起的作用请求路由和提供一些封装好的共通函数，这次的方案是使用pipeline插件，我们弱化一下路由，保留使用框架的公通函数，剩下的工作全部由分类的功能插件来完成。

方便其见，之后用字符图示的形式来描述Pipeline，如下图所示：

```
+---------+     +---------+     +-------+
| src     |     | filter  |     | log   |
sink    src - sink      src - sink     ....
+---------+     +---------+     +-------+
```


我们在init阶段，实现一个rulesource插件，用于读取规则。


```lua
local buffer = require "buffer"
local bjson = require "utils.bjson"

local src = { 
    args = bjson.loadf("./app/data/rules/args.rule", env)
}

local sink = nil

local rulesource_plugin = {}

function rulesource_plugin.init(self)
    self.source = src 
    self.sink = sink
    local t = bjson.decode(self.source['args'])
    buffer.sett("rule_args", t)
end

function rulesource_plugin.action(self, param) 
    self:init()
    return self.source,  self.sink
end

function rulesource_plugin.match(self, param)
end

return rulesource_plugin:action("param")
```
从上面的代码可以看出来，
```
local buffer = require "buffer"
local bjson = require "utils.bjson"
```
这两句功能实现是用blues框架完成，不属于WAF的代码， 而其它的代码就是WAF的rulesource插件本身了。


以前我们是通过配置文件，对插件进行初始化的，现在我们用pipeline组织一个新的插件组合，在这之前，我们要重写过去的那些插件，修改一个典型的filter型插件。

```lua
ocal buffer = require "buffer"
local xss_plugin = {}

local src = { 
    args = buffer.get("rule_args")

}
local sink = { 
    name = "xss_plugin"
}


function xss_plugin.push(self, param) 
end

function xss_plugin.init(self)
    self.source = src 
    self.sink = sink
end


function xss_plugin.action(self, param) 
end
function xss_plugin.match(self, param)
    self:init()
    for k, v in pairs(self.rules) do
        for _,value in pairs(v) do
            print(value)
        end 
    end 

    self.sink['found_flg']=true
    return self.source, self.sink
end

return xss_plugin
```

filter型插件是一种最典型的插件，以后的操作与规则分离设计的代码实现也，主要体现在这种插件中。看看过去我们构成和驱动插件工作的代码：

图示如下：

```
+----------+     +-----------+     +-----------+
| xss-plug |     | sql-plug  |     | cc-plug   |
sink      src - sink        src - sink        ....
+----------+     +-----------+     +-----------+
```

代码如下：

```lua
local xss_plugin = require"plugin.xss_plugin"
local sql_plugin = require"plugin.sql_plugin"
local cc_plugin = require"plugin.cc_plugin"

local plugin_list= { 
    xss=xss_plugin,
    sql=sql_plugin,
    cc=cc_plugin
}


return plugin_list


local plugin_list = require "plugin_config"

local plugin_factory = {}

function plugin_factory:start(params)
    for k,v in pairs(plugin_list) do
        v:match('params')
    end 
end

return plugin_factory:start("params")
```

而现在这种pipeline的形式，与上方法类型，但要以一个pipeline的顺序结构组织插件，然后启动插件运行方法。因为多了src,sink这两个slot，所以加入了push方法在插件之间传递数据。



###  5. 基于pipeline的插件系统与其它的插件系统有什么区别？

在pipeline序列中的插件是可以调换位置的。插件间共享编辑一个metadata和data共享流数据。metadata描述流数据的特征，data数据对插件来说是共享的， 就是通过nginx API获取的HTTP数据，不同的是，传给插件的HTTP Data是一个副本，在Pipeline中的前一个插件，可能会编辑这个数据，后一个数据在读取副本的HTTP Data，可能是在副本的基础上又被变更了。与此同时，每个插件在复到这个副本数据时，会在src或是sink的slot中，加入自己的tag或是私有数据，这数据对下面要进行数据处理的插件，有的是有价值的tag。


到这我们就引入一个问题，pipeline中的插件处理的流数据到底是什么，除了流数据，在src或是sink中还会有什么数据。


- Metadata

- Http Data 

- Plugin Tag


### 6. 插件执行与Nginx阶段的关系


nginx本身把用户请求的处理按阶段进行了划分，可以把这些执行阶段的先后，理解成插件按先后顺序执行，管道中的插件执行在各nginx阶段别分割成若干个子管道。

有的管道只有一个插件，而管道之间的插件是通过share diction来共享规则数据的，因为rule是与特定的处理插件关键的，在执行阶段rule是只读的一般不用更新编辑，而设计上市允许插件对流数据进行编辑的。

而那个插件去读share diction里的那个rule的key是写在插件自己的src和sink中的。

### 3. Pipeline执行与插件模板的回调函数

因为管道中的插件是会被顺序调用的，因此插件模板中的init和action函数也会被正常的回调，而这些回调函数在被调用时，管道系统会把流数据push给单元插件，而接到数据流的插件在接到回调push过来的数据后，进行相应的判断筛选，将编辑后的数据通过sink插槽push给后面的插件，直到管道尾端的插件报警或是记日志，一次管道启动运行的时序就结束了。


我们看一下，一个基于Lua实现的Pipeline插件系统，GST是基于C的单根继承的libc的模拟对象方式来实现pipeline系统，而用Lua简化模式，一些都变的简单了，不变的是类似的系统设计。

图示如下：
```
+------------+
| xss-plugin |
sink        src
+------------+
```
代码如下：
```lua

local xss_plugin = {}

local src = {
   args="xss args"

}
local sink = {
    name = "xss_plugin",
    ver = "0.1"
}


function xss_plugin.push(self, stream)
    for k,v in pairs(stream.metadata) do
        self.source[k]=v
        print(k,v)
    end
end

function xss_plugin.init(self)
    self.source = src 
    self.sink = sink
end

--回调函数，插件的主要处理逻辑在这里，需要的数据通过stream这个参数传入。
function xss_plugin.action(self, stream)
    for k,v in pairs(stream.metadata) do
        print(k,v)
    end
end

function xss_plugin.match(self, param)
    self.sink['found_flg']=false
    for kn,kv in pairs(self.source) do
         self.sink[kn] = kv
    end
    self.sink['metadata'] = { data=self.source['data'] }
    self:action(self.sink)
    return self.source, self.sink
end

return xss_plugin

```


这样设计的结果是，新的机能不是由单个插件来完成的，是把插件组装到管道中来完成的。

我们将http流数据和规则数据分开，然后可以讲管道系统脱离nginx在本地运行，最大限度的移植复用代码。


以上的模板代码，基本上都是固定了，不需要改太多代码，当你拿到这个模板时，如果是Filter类型的插件，只要做几个几个事情：


- 在action回调函数中，读取Pipeline源头插件传过来的metadata， Htttp的数据就在这里。
- 用Blues框架的Buffer库，按插件sink里的name属性的值：xss_plugin， 将属于这个插件的rule数据，读取出来就可以了。
- 如果你原意，可以修改传给下个插件的sink中内容，甚至可以修改其实metadata中的HTTP流数据。




### 7. 如何驱动一个Pipeline运转，并在插件间传递数据。


初期我们不用创建插件工厂，只要把插件当成库require引入即可：

```lua
local xss_plugin = require"plugin.content.xss_plugin"
local sql_plugin = require"plugin.content.sql_plugin"

local plugin_list= { 
    xss=xss_plugin,
    sql=sql_plugin,
}

return plugin_list
```

因为使用lua的原因，我们可以很简单的得到插件的实例。


```lua
local plugin_factory = {}

function plugin_factory:start(params)
    local src = { 
        metadata= { 
            data="http data"
        }   
    }   
    for k,v in pairs(plugin_list) do
        v:init()
        v:push(src)
        src, sink = v:match('params')
        src = sink
    end 
end

return plugin_factory:start("params")
```
通过上面的代码我们模拟了一次pipeline的执行过程，通过基于Pipeline的插件模板编写代码，我们可以快速的构建出很多的插件，并在现阶段进行串行链接，完成特定的任务。不同的插件组成不同的pipeline， 不同的pipeline执行不同的处理。






</div><!-- /.medium-8.columns -->



