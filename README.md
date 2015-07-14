# 如何在微信6.0.2+版本不接入微信API的情况下设置自定义分享内容（图片、链接、标题）

微信在6.0.2及以上版本已经回收客户端自定分享的权限，而是以授权api的形式开放出来。有时候我们只想简单地自定义分享的title,分享的图片以及分享的链接时，而不想或者缺乏资源去接入微信api的时候该怎么实现呢？

## 方法如下

1.设置分享title:动态改变document.title值即可:

```javascript
document.title = 'test'
```

2.设置分享图片：在页面隐藏一张尺寸大于290*290的图（图片需要容器包裹，设置容器css属性display:none即可）:

```javascript
<div style="display:none"><img src="share.jpg" /></div>
```
3.设置分享的链接：动态修改document.documentURI的值即可

```javascript
document.documentURI = "http://navyxie.github.io/"
```
以上方法即可在微信6.0.2+版本自定义分享内容，不需额外引入微信的js文件

-----------------------------------------------------------------------华丽丽分割线-----------------------------------------------------------------------

## 接下来，带大家通过解压微信app(android),分析js源代码来了解微信api的运行原理。

### 前期准备
1. 下载微信app(android),下载地址:[http://weixin.qq.com/cgi-bin/readtemplate?t=weixin_download_list&lang=zh_CN](http://weixin.qq.com/cgi-bin/readtemplate?t=weixin_download_list&lang=zh_CN)
2. 将weixin APK重名名成.zip格式，然后解压缩，在目录/assets/jsapi/下找到文件wxjs.js,我已经这个文件放在github,大家可以直接查看[wxjs.js](https://github.com/navyxie/weixin_js/blob/master/wxjs.js).这个就是我们要分析的文件了。

### 源码阅读与分析（目前微信Android最新版为6.2,我们以此为例子）

在微信6.0.2以前的版本，我们在做微信h5页面上做分享时，还清晰地记得一段这样的代码吗？

```javascript
WeixinJSBridge.on('menu:share:timeline', function(argv) { // 分享到朋友圈
	WeixinJSBridge.invoke('shareTimeline', {
		"img_url": window.ShareData.img,
		"link": window.ShareData.link,
		"title": window.ShareData.TimelineTitle,
		"desc": window.ShareData.TimelineTitle
	}, function(res) {	 
		window.ShareData.TimelineSuccess();
	});
});
```
所以我们通过搜索关键字：link,title,desc可以轻松地找到设置分享内容的位置（会找到多个，因为分享的渠道很多，朋友圈，朋友等）,我们找到分享到朋友圈那段设置分享内容的源代码(2549-2578行)：

```javascript
// share timeline
_on('menu:share:timeline',function(argv){
  _log('share timeline');

  var data;
  if (typeof argv.title === 'string') {
    data = argv;
    _call('shareTimeline',data);
  }else{
    data = {
        // "img_url": "",
        // "img_width": "",
        // "img_height": "",
        "link": document.documentURI || _session_data.init_url,
        "desc": document.documentURI || _session_data.init_url,
        "title": document.title
    };

    var shareFunc = function(_img){          
      if (_img) {
          data['img_url'] = _img.src;
          data['img_width'] = _img.width;
          data['img_height'] = _img.height;                        
      }

      _call('shareTimeline',data);
    };

    getSharePreviewImage(shareFunc);
  }
});
```
我们可以看到，当参数argv带有参数title时（通过调用微信api返回），会自然使用参数内容，否则title会使用document.title，desc会使用document.documentURI（当前页面打开的链接），link同样使用document.documentURI，那么自定图片的代码在哪里呢？上面代码的最后的函数getSharePreviewImage 就是获取页面分享图片的地址，我们追踪到函数getSharePreviewImage里去(2467-2576行)

```javascript
// 获取页面图片算法：
// 在页面中找到第一个最小边大于290的图片，如果1秒内找不到，则返回空（不带图分享）。
var getSharePreviewImage = function(cb){

  var isCalled = false;

  var callCB = function(_img){
    if (isCalled) {return;};
    isCalled = true;  

    cb(_img);
  }

  var _allImgs = _WXJS('img');
  if (_allImgs.length==0) {
    return callCB();
  }

  // 过滤掉重复的图片
  var _srcs = {};
  var allImgs = [];
  for (var i = 0; i < _allImgs.length; i++) {
    var _img = _allImgs[i];

    // 过滤掉不可以见的图片
    if (_WXJS(_img).css('display')=='none' || _WXJS(_img).css('visibility')=='hidden') {
      // _log('ivisable image !! ' + _img.src);
      continue;
    }
    
    if (_srcs[_img.src]) {
      // added
    } else {
      _srcs[_img.src] = 1; // mark added
      allImgs.push(_img);
    }
  };

  var results = [];

  var img;
  for (var i = 0; i < allImgs.length && i < 100; i++) {
    img = allImgs[i];

    var newImg = new Image();
    newImg.onload = function() {
      this.isLoaded = true;
      var loadedCount = 0;
      for (var j = 0; j < results.length; j++) {
        var res = results[j];
        if (!res.isLoaded) {
          break;
        }
        loadedCount++;
        if (res.width>290 && res.height>290) {
          callCB(res);
          break;
        }
      }
      if (loadedCount==results.length) {
        // 全部都已经加载完了，但还是没有找到。
        callCB();
      };
    }
    newImg.src = img.src;
    results.push(newImg);
  }

  setTimeout(function(){
    for (var j = 0; j < results.length; j++) {
      var res = results[j];
      if (!res.isLoaded) {
        continue;
      }
      if (res.width>290 && res.height>290) {
        callCB(res);
        return;
      }
    }
    callCB();
  },1000);
}
```

阅读上面代码，真相终于浮出水面，看注释：在页面中找到第一个最小边大于290的图片，如果1秒内找不到，则返回空（不带图分享），所以当你需要设置自定义分享的图片时，只需要在页面中隐藏一张尺寸大于290*290的图片即可（图片不能太大，因为1秒钟内如果加载不完这种图会视为失败，分享出去的内容就会不带图了，这也是为什么我们分享出去的内容有时候带图有时候不带图的原因）。

__注意，设置分享的图片必须包含在一个容器中，比如div,设置容器的css属性，display:none即可,直接设置图片display:none或者visibility:false,会被过滤掉,过滤代码如下__

```javascript
// 过滤掉不可以见的图片
if (_WXJS(_img).css('display')=='none' || _WXJS(_img).css('visibility')=='hidden') {
  // _log('ivisable image !! ' + _img.src);
  continue;
}
```

#总结

1. 当我们开发遇到问题时，最好的解决方法就是去阅读阅代码。
2. 我尝试着直接在browser模拟调用wxjs的api，会返回错误提示没权限，微信在安全授权这一块做得还是不错滴
3. 通过阅读阅代码，可以学习到很多知识，也能加深对该源代码的了解，为后续解决问题默默做了积累
4. 微信js内置了zepto的源代码
5. 微信webkit浏览器与native的通讯是通过创建一个隐藏的iframe,然后自定义通讯协议：weixin://。相信很多hybrid app 开发都在用这种方式。
6. 当你需要使用其他微api时，比如获取用户授权信息，就需接微信api了。

#小坑

1. 自定义分享的图片尺寸必须大于290*290,且必须由一个隐藏的容器包裹着
2. 虽然我们可以在不接入weixin api的情况下自定义分享的title,link以及image，但是我们不能自定分享的描述内容（desc），默认使用了document.documentURI

