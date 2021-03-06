## Introduction

CDNBye implements [WebRTC](https://en.wikipedia.org/wiki/WebRTC) datachannel to scale live/vod video streaming by peer-to-peer network using bittorrent-like protocol.

To use CDNBye hlsjs-p2p-engine, WebRTC support is required (Chrome, Firefox, Opera, Safari).

## Use Hls.js wrapped with P2PEngine

### `Hls.engineVersion` (static method)
Show the current version of CDNBye plugin.

### `Hls.WEBRTC_SUPPORT` (static method)
Is WebRTC natively supported in the environment?
```javascript
if (Hls.WEBRTC_SUPPORT) {
  // WebRTC is supported
} else {
  // Use a fallback
}
```

## Create instance

### `var hls = new Hls({p2pConfig: [opts]});` 
Create a new `Hls` instance.

### `var engine = hls.p2pEngine;`
Get the `P2PEngine` instance from `Hls` instance.

If `opts` is specified, then the default options (shown below) will be overridden.

| Field | Type | Default | Description |
| :-: | :-: | :-: | :-: |
| `logLevel` | string or boolean | 'none' | Print log level(warn, error, none，false=none, true=warn).
| `live` | boolean | false | tell engine whether in live or VOD mode, set to false will pre-buffer for smooth playing.
| `wsSignalerAddr` | string | 'wss://signal.cdnbye.com' | The address of signal server.
| `announce` | string | 'https://tracker.cdnbye.com/v1' | The address of tracker server.
| `wsMaxRetries` | number | 15 | The maximum number of reconnection attempts that will be made by websocket before giving up.
| `wsReconnectInterval` | number | 30 | The number of seconds to delay before attempting to reconnect by websocket.
| `memoryCacheLimit` | Object | {"pc": 1024 * 1024 * 512, "mobile": 1024 * 1024 * 256} | The max size of binary data that can be stored in the cache.
| `p2pEnabled` | boolean | true | Enable or disable p2p engine.
| `wifiOnly` | boolean | false | Only allow uploading on Wi-Fi and Ethernet.
| `dcDownloadTimeout` | number | 25 | Max download timeout for WebRTC datachannel.
| `webRTCConfig` | Object | {} | A [Configuration dictionary](https://github.com/feross/simple-peer) providing options to configure WebRTC connections.
| `useHttpRange` | boolean | false | Use HTTP ranges requests where it is possible. Allows to continue (and not start over) aborted P2P downloads over HTTP(True in live mode by default).
| `getStats` | function | - | Get the downloading statistics, including totalP2PDownloaded, totalP2PUploaded and totalHTTPDownloaded.
| `getPeerId` | function | - | Emitted when the peer Id of this client is obtained from server.
| `getPeersInfo` | function | - | Emitted when successfully connected with new peer.
| `channelId` | function | - | Pass a function to generate channel Id.(See advanced usage)
| `validateSegment` | function | - | Pass a function to check segment validity downloaded from peers.

<!--
| `segmentId` | function | - | Pass a function to generate segment Id.(See advanced usage)
-->

## P2PEngine API

### `P2PEngine.version` (static method)
Get the version of `P2PEngine`.

### `P2PEngine.isSupported()` (static method)
Returns true if WebRTC data channel is supported by the browser.

### `var engine = new P2PEngine(hlsjs, p2pConfig);`
Create a new `P2PEngine` instance. Or you can get `P2PEngine` instance from hlsjs:
```javascript
var hls = new Hls();
var engine = hls.p2pEngine;
```

### `engine.enableP2P()`
Resume P2P if it has been stopped.

### `engine.disableP2P()`
Disable engine to stop p2p and free used resources.

### `engine.destroy()`
Stop p2p and free used resources, it will be called automatically before hls.js is destroyed.  

## P2PEngine Events

### `engine.on('peerId', function (peerId) {})`
Emitted when the peer Id of this client is obtained from server.

### `engine.on('peers', function (peers) {})`
Emitted when successfully connected with new peer.

### `engine.on('stats', function (stats) {})`
Emitted when data is downloaded/uploaded.</br>
stats.totalHTTPDownloaded: total data downloaded by HTTP(KB).</br>
stats.totalP2PDownloaded: total data downloaded by P2P(KB).</br>
stats.totalP2PUploaded: total data uploaded by P2P(KB).

## Advanced Usage
### Another way to get the downloading statistics
```javascript
p2pConfig: {
    getStats: function (totalP2PDownloaded, totalP2PUploaded, totalHTTPDownloaded) {
        // do something
    }
}
```

### Another way to get peer Id
```javascript
p2pConfig: {
    getPeerId: function (peerId) {
        // do something
    }
}
```

### Another way to get peers information
```javascript
p2pConfig: {
    getPeersInfo: function (peers) {
        // do something
    }
}
```

### Dynamic m3u8 path issue
Some m3u8 urls play the same live/vod but have different paths on them. For example, 
example.com/clientId1/file.m3u8 and example.com/clientId2/file.m3u8. In this case, you can format a common channelId for them. `It is strongly recommended to add a unique identifier to the channelid to prevent conflicts with other channels. If there's a collision, our backend is going to match peers that aren't watching the same content together, and that can lead to unpredictable results.`
```javascript
p2pConfig: {
    channelId: function (m3u8Url) {
        const formatedUrl = 'YOUR_UNIQUE_ID' + format(m3u8Url);   // format a channelId by removing the different part
        return formatedUrl;
    }
}
```

<!--
### Dynamic ts path issue
Like dynamic m3u8 path issue, you should format a common segmentId for the same ts file. We have da that for you. If you want to set the path as segment ID, override the segmentID like this:
```javascript
p2pConfig: {
    segmentId: function (level, sn, tsUrl) {
        // const formatedUrl = `${level}-${sn}`;  // default implementation
        const formatedUrl = tsUrl;  // the actual path of ts file
        return formatedUrl;
    }
}
```
-->

### Config STUN Servers
```javascript
p2pConfig: {
    webRTCConfig: { 
        config: {         // custom webrtc configuration (used by RTCPeerConnection constructor)
            iceServers: [
                { urls: 'stun:stun.l.google.com:19302' }, 
                { urls: 'stun:global.stun.twilio.com:3478?transport=udp' }
            ] 
        }
    }
}
```

### Allow Http Range Request
If http range request is activated, we are able to get chunks of data from peer and then complete the segments by getting other chunks from the CDN, thus, reducing your CDN bandwidth. To activate range requests, See [Allow Http Range Request](../m3u8.md?id=allow-http-range-request). Besides, the code below is needed：
```javascript
p2pConfig: {
    useHttpRange: true,
}
```

### How to Check Segment Validity
Sometimes we need to prevent a peer from sending a fake segment
 (such as the bittorrent with a hash function). 
 CDNBye provides a validation callback with buffer of the 
 downloaded segment, developer should implement the actual 
 validator. For example, you can create a program that generates 
 hashes for the segments and stores them in a specific file or 
 injects into m3u8 playlist files the hashes information. If 
 the callback returns false, then the segment is not valid. 
 ```javascript
p2pConfig: {
    validateSegment: function (level, sn, buffer) {
        var hash = hashFile.getHash(level, sn);
        return hash === md5(buffer);
    }
}
```

### Online Debugging
CDNBye provides two query parameters for online debugging:
- You can display the log information in console by adding a query parameter `_debug=1` to the url, such as `http://your_website.com?_debug=1`.
- In the case that P2P has been enabled, to temporarily disable P2P, you can add query parameter `_p2p=0` to the url, such as `http://your_website.com?_p2p=0`.
