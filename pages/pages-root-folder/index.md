---
#
# Use the widgets beneath and the content will be
# inserted automagically in the webpage. To make
# this work, you have to use › layout: frontpage
#

layout: frontpage

tags: candylab

header:
  image_fullwidth: header_unsplash_12_1.jpg
  
widget1:
  title: "发布的文章"
  url: 'https://yqfy.net/'
  image: widget-1-302x182.jpg
  text: '在一些平台上公开发布过的文章。'
   
widget2:
  title: "Lua从入门到放弃"
  url: 'https://lua.ren/'
  image: widget-never-give-up-302x182.jpg  
  text: '一个LUA和Moonscript为主的博客。'
   
widget3:
  title: "OpenResty中国"
  url: 'http://orchina.org'
  image: /widget-7-302x182.jpg
  text: 'Openresty社区记录一些朋友发布的实践文章。'
  
widget4:
  title: "freebuf专栏"
  url: 'http://zhuanlan.freebuf.com/column/index/?name=%E7%B3%96%E6%9E%9C%E5%AE%9E%E9%AA%8C%E5%AE%A4'
  image: widget-freebuf-302x182.jpg
  text: '糖果实验室freebuf专栏'

widget5:
  title: "小专栏"
  url: 'https://xiaozhuanlan.com/candylab'
  image: widget-point-line-302x182.jpg  
  text: '糖果实验室小专栏'

widget6:
  title: "SOC日志收集实践"
  url: 'http://www.freebuf.com/articles/es/174281.html'
  image: widget-mailproxy-302x182.jpg
  text: '企业邮件服务日志收集'

widget7:
  title: "ClickHouse与威胁日志分析"
  url: 'http://www.freebuf.com/column/164671.html'
  image: widget-clickhouse-302x182.jpg
  text: '基于Clickhouse做威胁数据聚合分析。'

widget8:
  title: "Redis流量监听"
  url: 'http://www.freebuf.com/column/164671.html'
  image: widget-redis-monitor-302x182.jpg
  text: '用Lua处理Redis的端口监听数据。'

widget9:
  title: "开源SOC的设计与实践"
  url: 'http://www.freebuf.com/articles/network/173282.html'
  image: widget-opensoc-302x182.jpg
  text: '如何用一些开源产品实现一个SOC。'




widget19:
  title: "Download Theme"
  url: 'https://github.com/Phlow/feeling-responsive'
  image: widget-github-303x182.jpg
  text: '<em>Feeling Responsive</em> is free and licensed under a MIT License. Make it your own and start building. Grab the <a href="https://github.com/Phlow/feeling-responsive/tree/bare-bones-version">Bare-Bones-Version</a> for a fresh start or learn how to use it with the <a href="https://github.com/Phlow/feeling-responsive/tree/gh-pages">education-version</a> with sample posts and images. Then tell me via Twitter <a href="http://twitter.com/phlow">@phlow</a>.'



#
# Use the call for action to show a button on the frontpage
#
# To make internal links, just use a permalink like this
# url: /getting-started/
#
# To style the button in different colors, use no value
# to use the main color or success, alert or secondary.
# To change colors see sass/_01_settings_colors.scss
#
callforaction:
  url: https://shengyang.xyz/blog/ 
  text: 文章归档 ›
  style: alert
permalink: /index.html
#
# This is a nasty hack to make the navigation highlight
# this page as active in the topbar navigation
#

homepage: true
---

<meta name="keywords" content="candylab, 糖果实验室">





