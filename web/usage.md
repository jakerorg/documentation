
## 快速集成步骤
- 访问[CDNBye后台系统OMS](https://oms.cdnbye.com)，按提示绑定域名
- 选择一个网站目前在用的[播放器](/web/players.md)（例如DPlayer），或者直接使用官方提供的[php版代码](/web/players?id=cbplayer)
- 将播放器部分代码替换成demo的代码，修改播放地址，p2p插件也可以[本地化](https://cdnbye.oss-cn-beijing.aliyuncs.com/web_sdk/dist.zip)
- 调试通过后部署到服务器
- 访问[CDNBye后台系统OMS](https://oms.cdnbye.com)，即可查看P2P效果
- 如果P2P不起作用，请参照[问题排查步骤](/FAQ.md?id=web端插件p2p无效问题排查步骤)进行排查

## 快速入门Demo
将以下拷贝到您的网页中并运行。再打开另一个相同的网页。见证奇迹的时候到了！您已在两个网页之间建立了一个P2P连接，在不安装任何插件的情况下。如果在这个频道中（一个m3u8标识了一个频道）没有其它参与者，那么您打开的第一个网页将作为种子为第二个网页提供数据。
```html
<script src="https://cdn.jsdelivr.net/npm/cdnbye@latest"></script>
<video id="video" controls></video>
<p id="version"></p>
<h3>download info:</h3>
<p id="info"></p>
<script>
    document.querySelector('#version').innerText = `hls.js version: ${Hls.version}  cdnbye version: ${Hls.engineVersion}`;
    if(Hls.isSupported()) {
        var video = document.getElementById('video');
        var hls = new Hls({
            p2pConfig: {
                logLevel: true,
                live: false,        // 如果是直播设为true
                // Other p2pConfig options provided by CDNBye
            }
        });
        hls.loadSource('https://example.m3u8');
        hls.attachMedia(video);
        hls.on(Hls.Events.MANIFEST_PARSED,function(event, data) {
            video.play();
        });
        hls.p2pEngine.on('stats', function ({totalHTTPDownloaded, totalP2PDownloaded, totalP2PUploaded}) {
            var total = totalHTTPDownloaded + totalP2PDownloaded;
            document.querySelector('#info').innerText = `p2p ratio: ${Math.round(totalP2PDownloaded/total*100)}%, saved traffic: ${totalP2PDownloaded}KB, uploaded: ${totalP2PUploaded}KB`;
        });
    }
</script>
```

## 使用原生hls.js项目
如果您在项目中引入了hls.js的script标签，那么只需要将该标签如：
 ```html
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
```
替换为
 ```html
<script src="https://cdn.jsdelivr.net/npm/cdnbye@latest"></script>
```
就是这么简单！


## 第三方播放器集成
参考[第三方播放器](/web/players.md)。

## 引入插件

#### Script标签引入
通过script标签引入已经和hls.js打包的最新版本（推荐）：
```html
<script src="https://cdn.jsdelivr.net/npm/cdnbye@latest"></script>
```
或者引入没有与hls.js打包的独立版本：
```html
<script src="https://cdn.jsdelivr.net/npm/hls.js@latest"></script>
<script src="https://cdn.jsdelivr.net/npm/cdnbye@latest/dist/hlsjs-p2p-engine.min.js"></script>
```

#### 文件引入
[点击下载](https://cdnbye.oss-cn-beijing.aliyuncs.com/web_sdk/dist.zip)<br>注意js代码需要放在播放器代码之前执行，可以在引入播放器代码的script标签之前引入。

#### Browserify / Webpack
```shell
npm install --save cdnbye
```
在播放器模块中通过`require`引入cdnbye：
```javascript
var Hls = require('cdnbye');
```
或者使用ES6的`import`语法：
```javascript
import Hls from 'cdnbye';
```

## 使用插件
#### Bundle
在`hlsjsConfig`对象字面量中加入`p2pConfig`字段，然后在实例化hls.js时把`hlsjsConfig`作为参数传入。
```javascript
var hlsjsConfig = {
    debug: true,
    // Other hlsjsConfig options provided by hls.js
    p2pConfig: {
        logLevel: 'debug',
        // Other p2pConfig options if applicable
    }
};
// Hls constructor is overriden by included bundle
var hls = new Hls(hlsjsConfig);
// Use `hls` just like the usual hls.js ...
hls.loadSource(contentUrl);
hls.attachMedia(video);
hls.on(Hls.Events.MANIFEST_PARSED,function() {
    video.play();
});
```
#### Engine(没有打包hls.js的插件，需要自己引入hls.js，注意引入CDNBye提供的hls.js则不需要再引入Engine)
实例化hls.js并将`hlsjsConfig`作为参数传入。然后实例化`P2PEngine`并将`p2pConfig`作为参数传入。调用hls.js的`loadSource`和`attachMedia`方法。
```javascript
var hlsjsConfig = {
    maxBufferSize: 0,       // Highly recommended setting in live mode
    maxBufferLength: 5,     // Highly recommended setting in live mode
    liveSyncDuration: 30,   // Highly recommended setting in live mode
};

var p2pConfig = {
    logLevel: 'debug',
    // Other p2pConfig options if applicable
};

var hls = new Hls(hlsjsConfig);
if (P2PEngine.isSupported()) {
    new P2PEngine(hls, p2pConfig);        // Key step
}

// Use `hls` just like your usual hls.js…
hls.loadSource(contentUrl);
hls.attachMedia(video);
hls.on(Hls.Events.MANIFEST_PARSED,function() {
    video.play();
});
```

## 兼容IE浏览器
由于插件使用了ES6的API，会导致在IE浏览器下报错，可以在项目中引入polyfill来解决这个问题，首先通过npm安装：
```bash
npm install --save @babel/polyfill
```
然后在相应模块中引入：
```javascript
require("@babel/polyfill");
```
或者可以在html中引入插件之前加入一行script标签：
```html
<script src="https://cdn.bootcss.com/babel-polyfill/7.4.4/polyfill.min.js"></script>
```

## Electron
本插件同样支持 [Electron](https://electronjs.org/) 平台（CDNBye版本>=0.10.0），只需求将从控制台获取的token等信息传进config中即可，如下所示：
```javascript
var hlsjsConfig = {
    p2pConfig: {
        token: YOUR_TOKEN,
        appName: YOUR_APP_NAME,    // 应用的名称
        appId: YOUR_APP_ID,        // 需要与控制台输入的保持一致
        // Other p2pConfig options if applicable
    }
};
```
参考[如何获取token](/bindings.md?id=绑定-app-id-并获取token)
