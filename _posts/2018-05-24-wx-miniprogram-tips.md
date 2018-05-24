---
title: 微信小程序第三方开发需要注意的点
layout: post
guid: urn:uuid:85cc4b63-e9f0-4454-8ca7-fbdsdc2a2ff
tags:
  - 小程序
  - 第三方
  - mpvue
---


### 1.关于小程序与公众号unionid数据互通

小程序和公众号必须为同一主体，而且必须绑定在同一个开放平台上，其中又分为两点：

- 1.开放平台主动创建
  此时需要手动将小程序和公众号绑定在开放平台上，不可使用接口绑定；

- 2.开放平台由第三方接口创建
  在公众号和小程序授权第三方开放平台管理权限后，可由接口进行绑定，若要绑定在同一个开放平台上，前提是公众号和小程序的主体相同，同时也支持接口解除绑定；

### 2.关于小程序ext.json

开发过程中可在小程序的根目录下添加ext.json文件来进行调试，比如在文件里添加以下配置：

```json
{
  "extEnable": true,
  "extAppid": "wx90d87655678517ee",
  "ext": {
    "id": 100
  }
}
```

设置extEnable属性为true，后续小程序的逻辑将依照ext.json中extAppid来执行，并可以通过以下两种方式来获取ext中的参数：

```javascript
//方式一
if(wx.getExtConfig) {
  wx.getExtConfig({
    success: function (res) {
      console.log(res.extConfig)
    }
  })
}
//方式二
let extConfig = wx.getExtConfigSync? wx.getExtConfigSync(): {}
console.log(extConfig)
```
另外第三方在帮助授权小程序提交代码的时候也可以指定ext字段，具体方式参考：[微信第三方代小程序实现接口](https://open.weixin.qq.com/cgi-bin/showdocument?action=dir_list&t=resource/res_list&verify=1&id=open1489140610_Uavc4&token=&lang=zh_CN)
其中需要注意的一点是```ext_json```的格式要非常注意，因为```ext_json```结构很松散，所以很多时候需要我们自己拼装这个字段，那么它需要为类似以下格式：

```
"{\"extAppid\": \"wx90d87655678517ee\",\"ext\":{\"id\":100}}"
```

另外通过接口提交的```ext_json```会覆盖配置的ext.json中的对应信息，但在开发者工具无法获取接口提交的```ext_json```的信息，只能获取ext.json中信息，这点需要注意。

### 3.mpvue结合ext.json

目前最新版本的mpvue好像还不支持在main.js中添加ext.json的配置，所以若是需要使用它有以下两种方式：

- 1.写一个插件，将src下的ext.json编译的dist目录下（mpvue编译后的小程序源码目录）
- 2.直接在dist目录下创建一个ext.json文件
但记得要将extEnable属性设置为true。

### 4.用户小程序授权之后的解密算法（js版）

```javascript
import crypto from 'crypto-browserify'

function WXBizDataCrypt (appId, sessionKey) { //小程序appid，用户登录利用code从接口获得的sessionKey
  this.appId = appId
  this.sessionKey = sessionKey
}

WXBizDataCrypt.prototype.decryptData = function (encryptedData, iv) {
  // base64 decode
  var sessionKey = new Buffer(this.sessionKey, 'base64')
  encryptedData = new Buffer(encryptedData, 'base64')
  iv = new Buffer(iv, 'base64')

  try {
     // 解密
    var decipher = crypto.createDecipheriv('aes-128-cbc', sessionKey, iv)
    // 设置自动 padding 为 true，删除填充补位
    decipher.setAutoPadding(true)
    var decoded = decipher.update(encryptedData, 'binary', 'utf8')
    decoded += decipher.final('utf8')
    decoded = JSON.parse(decoded)
  } catch (err) {
    console.log(err)
    throw new Error('Illegal Buffer')
  }
  if (decoded.watermark.appid !== this.appId) {
    throw new Error('Illegal Buffer')
  }

  return decoded
}

let DataCrypt = {
  getData (appId, sessionKey, encryptedData, iv) {
    var pc = new WXBizDataCrypt(appId, sessionKey)
    return pc.decryptData(encryptedData, iv)
  }
}

export default DataCrypt

使用方式：

import DataCrypt from "path/to/DataCrypt"

const data = DataCrypt.getData(...)

```

