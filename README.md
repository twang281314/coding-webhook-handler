---
title: 利用coding.net的webhook自动更新代码
date: 2017-09-30 10:00:00
categories: git
tags: webhook
keywords: 
---

### coding.net对webhook的解释： 
>Webhook 允许第三方应用监听 Coding.net 上的特定事件，在这些事件发生时通过 HTTP POST 方式通知( 超时5秒) 到第三方应用指定的 Web URL。例如项目有新的内容 Push，或是Merge Request 有>更新等。 Coding.net 用户可以在自己的项目 → 设置 → Webhook 中创建、设置 Webhook 所需监听的事件，并配置第三方应用的 Web URL 。WebHook 可方便用户实现自动部署，自动测试，自动打包，>监控项目变化等。

本文实现的功能是通过webhook实现远程git仓库的自动更新
<!-- more -->

### 服务端NodeJS 的监听程序 deploy.js

``` javascript
var http = require('http')
var createHandler = require('coding-webhook-handler')
var handler = createHandler({
  path: '/webhook',
  token: 'yourtoken' // maybe there is no token 
})

function run_cmd(cmd, args, callback) {
  var spawn = require('child_process').spawn;
  var child = spawn(cmd, args);
  var resp = "";

  child.stdout.on('data', function(buffer) { resp += buffer.toString(); });
  child.stdout.on('end', function() { callback (resp) });
}

http.createServer(function (req, res) {
  handler(req, res, function (err) {
    res.statusCode = 404
    res.end('no such location')
  })
}).listen(7777)

handler.on('error', function (err) {
  console.error('Error:', err.message)
})

handler.on('push', function (event) {
  console.log('Received a push event for %s to %s',
    event.payload.repository.name,
    event.payload.ref);
  run_cmd('sh', ['./deploy.sh'], function(text){ console.log(text) });
})

```
```
cnpm install -g coding-webhook-handler
```
### 部署脚本 deploy.sh

``` shell
#!/bin/bash

WEB_PATH='/usr/coding.net/blog/'$1
WEB_USER='root'
WEB_USERGROUP='root'

echo "Start deployment"
cd $WEB_PATH
echo "pulling source code..."
git reset --hard origin/master
git clean -f
git pull
git checkout master
echo "changing permissions..."
chown -R $WEB_USER:$WEB_USERGROUP $WEB_PATH
echo "Finished."

```


### 参考文章
* https://open.coding.net/webhooks/
* http://blog.csdn.net/auv1107/article/details/51999592
* http://www.jianshu.com/p/e4cacd775e5b
* https://github.com/rvagg/github-webhook-handler
* https://github.com/spacelan/coding-webhook-handler