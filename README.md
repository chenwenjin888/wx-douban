>小程序-豆瓣电影项目实例，包含三个功能模块（首页列表、搜索列表、详情页） ，适合刚入门学习的同学  

## 资料
[1. 小程序官方文档 ](https://mp.weixin.qq.com/debug/wxadoc/introduction/index.html)  
[2. 微信web开发者工具下载地址 ](https://mp.weixin.qq.com/debug/wxadoc/dev/devtools/download.html)  
[3. 微信小程序解决方案专辑 ](http://www.wxapp-union.com/special/solution.html)  
[4. 豆瓣电影API ](https://developers.douban.com/wiki/?title=movie_v2)  

## 效果  
![豆瓣电影.gif](http://upload-images.jianshu.io/upload_images/5019151-9957905f85011a35.gif?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## 开始  
>这里先说明下，此案例使用的是体验模式，未使用AppID，这是因为我们的请求用的是第三方豆瓣API，配置安全域名的话也是没有权限访问，所以我就偷了个懒，直接使用自己写的一个node代理服务来请求资源。如果使用了AppID的话，那就配置一个自己能访问的域名信息。

打开微信web开发者工具，新建个小程序  
![新建项目.png](http://upload-images.jianshu.io/upload_images/5019151-0ef22861ef868554.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
选择体验小程序，确定后小程序自动会帮助我们生成一个项目目录。下面是我的项目的整体结构  
```
// 目录结构
.
├── common
│   ├── images
│   ├── filter.wxs // 自己封装的过滤器
│   └── head.wxml  // 头部模板
├── node-proxy     // node服务器
│   ├── index.js
│   ├── package.json
├── component      // 自定义公共组件
│   └── movies     // 电影列表组件目录
├── store
│   ├── index      // 首页：展示正在上映电影列表
│   ├── search     // 搜索页：展示搜索后电影列表
│   └── detail     // 电影详情
├── app.js         // 注册小程序
├── app.json       // 小程序全局配置文件
├── app.wxss       // 全局样式
└── project.config.json // 工具配置文件
.
```  
## node服务  
在node-proxy目录下创建我们的服务，怎么搭建这里不做多介绍，不会的直接拷贝文件后运行命令`npm install`或者淘宝的镜像`cnpm install`即可

```
// node-proxy/package.json
{
  "name": "node-proxy",
  "version": "1.0.0",
  "description": "豆瓣电影服务器端",
  "main": "index.js",
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "author": "",
  "license": "ISC",
  "dependencies": {
    "express": "^4.16.2",
    "superagent": "^3.8.2"
  }
}
```
```
// node-proxy/index.js
// 引入模块
const express = require('express'),
    request = require('superagent');

// 创建APP
const app = express();

// 豆瓣电影API
const host = 'https://api.douban.com/v2';

app.all('*', function (req, res, next) {
    if (!req.get('Origin')) return next();
    // use "*" here to accept any origin
    res.set('Access-Control-Allow-Origin', '*');
    res.set('Access-Control-Allow-Methods', 'GET');
    res.set('Access-Control-Allow-Headers', 'X-Requested-With, Content-Type');
    if ('OPTIONS' == req.method) return res.send(200);
    next();
});

// 正在上映 API
app.get('/movie/in_theaters', function (req, res) {
    reqHttp(req, res);
});

// 搜索
app.get('/movie/search', function (req, res) {
    reqHttp(req, res);
})

// 详情
app.get('/movie/subject/:id', function (req, res) {
    reqHttp(req, res);
})

function reqHttp(req, res){
    let reqHttp = request.get(host + req.originalUrl);
    reqHttp.pipe(res);
    reqHttp.on('end', (err, res) => {
    });
}

// 监听
app.listen(5200);
```
然后我们通过`node index.js` 来启动服务
## 全局配置  
我们有三个模块（首页、搜索、详情页），所以先配置好他们的路由

```
{
  "pages":[
    "pages/index/index",
    "pages/search/search",
    "pages/detail/detail"
  ],
  "window":{
    "backgroundTextStyle":"light",
    "navigationBarBackgroundColor": "#fff",
    "navigationBarTitleText": "小程序-豆瓣电影",
    "navigationBarTextStyle":"black"
  }
}
```

## 首页  
首页有两大块：头部和电影列表  
**头部：** 我们把头部设想为一个公共的模板页面来引入  
**电影列表：** 因为首页和搜索页面都需要展示电影列表，那就让它们也共用吧，直接使用小程序的自定义组件来来实现此功能，关于自定义组件，请[参考官方文档](https://mp.weixin.qq.com/debug/wxadoc/dev/framework/custom-component/) 
```
// pages/index/index.wxml
<!-- 模板引入 -->
<import src="../../common/head.wxml"/>

<view class="container">
    <!-- 使用模板 -->
    <template is="head"/>
    <!-- 以下是一个自定义组件的引用 -->
    <movies movie-type="in_theaters"></movies>
</view>
```
>这里我们使用了一个组件名为movies的标签，这个名称是怎么来的，我下面讲解
```
// common/head.wxml
<template name="head">
    <view class="page-head">
        <form class="head-form" bindsubmit="formSubmit">
            <input placeholder="输入电影名称" class='head-input' name="searchKey" />
            <button formType="submit" class='head-button'>搜索</button>
        </form>
    </view>
</template>
```
头部搜索后`bindsubmit="formSubmit"`有个事件处理，然后根据条件进入搜索界面
```
// pages/index/index.js
Page({
    data: {},
    formSubmit(e){
        let _searchKey = e.detail.value.searchKey;
        // 这里做值校验
        if(!_searchKey)return;
        wx.navigateTo({
            url: '/pages/search/search?q=' + _searchKey
        })
    }
})
```  
现在我们头部功能已经完成，好像漏了点东西，那就是我们的全局样式。代码有点多，我就直接放压缩的了， 要看非压缩的我下面会提供项目完整路径，可下载

```
// app.wxss
page{height:100%;font-size:28 rpx;line-height:1.6;color:#666;background-color:#f8f8f8}checkbox,radio{margin-right:10 rpx}form{width:100%}.padding-10{padding:10 rpx}.container{min-height:100%;justify-content:space-between;font-size:32 rpx;font-family:-apple-system-font, Helvetica Neue, Helvetica, sans-serif}.page-head{background-color:#fff;border-top:1px solid #e7e7e7;border-bottom:1px solid #e7e7e7;height:80 rpx;width:100%;font-size:28 rpx}.head-form{display:block;padding-top:12 rpx;text-align:center;line-height:56 rpx}.head-form .head-input{display:inline-block;border:1px solid #ccc;background-color:#fff;border-radius:8 rpx;text-align:left;padding:0 10 rpx}.head-form .head-button{display:inline-block;background-color:#fff;border-radius:4 rpx;width:100 rpx;line-height:56 rpx;height:56 rpx;padding:0;margin-left:14 rpx;font-size:28 rpx}
```

下面开始我们的电影列表的自定义组件
## 自定义组件-电影列表  
一个自定义组件由 json wxml wxss js 4个文件组成。有个快捷操作能直接生成这4个文件
![自定义组件.png](http://upload-images.jianshu.io/upload_images/5019151-138d9e266e164b69.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  
要编写一个自定义组件，首先需要在 json 文件中进行自定义组件声明（将 component 字段设为 true 可这一组文件设为自定义组件），如果选用快捷方式生成的文件， 这里已经自动帮我们写好了
```
// component/movies/movies.json
{
  "component": true
}
```
下面开始编写我们的电影列表组件模板,以及样式。编写组件样式时，需要注意一些问题：
- 组件和引用组件的页面不能使用id选择器（#a）、属性选择器（[a]）和标签名选择器，请改用class选择器。
- 组件和引用组件的页面中使用后代选择器（.a .b）在一些极端情况下会有非预期的表现，如遇，请避免使用。
- 子元素选择器（.a>.b）只能用于 view 组件与其子节点之间，用于其他组件可能导致非预期的情况。
- 继承样式，如 font 、 color ，会从组件外继承到组件内。
- 除继承样式外， app.wxss 中的样式、组件所在页面的的样式对自定义组件无效。
- 组件可以指定它所在节点的默认样式，使用 :host 选择器
```
// component/movies/movies.wxml
<!-- 组件模板 -->
<view>
    <text class="padding-10">{{movieTitle}}</text>
</view>
<view class='movie-list'>
    <view class='movie-item' wx:for="{{movieList}}" wx:for-item="movie">
        <navigator url="/pages/detail/detail?id={{movie.id}}">
            <view class='item-inner'>
                <image mode="scaleToFill" src="{{movie.images.medium}}" class='movie-image'/>
                <view>
                    <text class="movie-name">{{movie.title}}</text>
                </view>
                <image src="../../common/images/rating{{movie.rating.stars}}.png" class="bigstar"/>
                {{movie.rating.average}}
            </view>
        </navigator>
    </view>
</view>

```

```
// component/movies/movies.wxss
.padding-10{
    padding: 10rpx;
}
.movie-list{
    display: flex;
    flex-wrap: wrap;
    padding: 5rpx;
}
.movie-item{
    width: 33.333333%;
    text-align: center;
    overflow: hidden;
}
.item-inner{
    margin: 5rpx;
    color: #666;
    font-size: 28rpx;
}
.movie-image{
    width: 100%;
    height: 300rpx;
}
.bigstar{
    width: 120rpx;
    height: 24rpx;
}
.movie-name{
    display: block;
    font-size: 28rpx;
    text-align: center;
    white-space:nowrap;
    overflow: hidden;
    text-overflow: ellipsis;
}
```
在自定义组件的 js 文件中，需要使用 Component() 来注册组件，并提供组件的属性定义、内部数据和自定义方法。在实现交互之前，我们先配置一下请求的全局变量，也就是node服务地址

```
// app.js
App({
  globalData: {
    apiUrl: 'http://localhost:5200'
  }
})
```

```
// component/movies/movies.js
const app = getApp();
Component({
    properties: {
        // 定义组件的对外属性，属性值可以在组件使用时指定
        movieType: {
            type: String,
            value: 'in_theaters',
        }
    },
    data: {
        // 这里是一些组件内部数据
        movieTitle: '',
        movieList: []
    },
    ready(){
        // 验证请求来源 搜索、首页
        let _url = 'in_theaters';
        // 获取电影列表
        this.getMovieList(_url);
    },
    methods: {
        // 获取电影列表
        getMovieList (url) {
            let _this = this;
            wx.showLoading();
            wx.request({
                url: app.globalData.apiUrl + '/movie/' + url,
                method: 'GET',
                data: { count: 15 },
                success: function (res) {
                    // 此处省略请求是否成功校验
                    _this.setData({
                        movieTitle: res.data.title,
                        movieList: res.data.subjects
                    });
                    wx.hideLoading();
                }
            })
        }
    }
})
```
至此，我们还差最关键的一步，在我们需要使用该组件页面的JSON文件中进行申明引入`usingComponents`
```
// pages/index/index.json
{
    "usingComponents": {
        "movies": "../../component/movies/movies"
    }
}
```
>我们上面提到index.wxml有一个movies标签组件是怎么来的问题，它就是这里配置而来  

现在我们的首页已经完成，不出意外的话你可以看到如下页面：
![首页.png](http://upload-images.jianshu.io/upload_images/5019151-41f1049d51f3a9cc.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

## 搜索页  
电影列表完成后，我们搜索界面已经非常简单了，在JSON文件中引入申明后页面直接引入组件
```
// pages/search/search.json
{
  "navigationBarTitleText": "电影搜索",
  "usingComponents": {
    "movies": "../../component/movies/movies"
  }
}
```

```
// pages/search/search.wxml
<view class="container">
    <!--以下是对一个自定义组件的引用 -->
    <movies movie-type="search"></movies>
</view>
```
你可能在想，那我们的search.js要不要做什么事呢，很清楚的告诉你， 啥都不用管，让它自生自灭吧。我们只需要自定义组件movies.js加入一个类型校验即可。  
眼尖的你可能发现我们在页面引入组件的时候都有一个`movie-type`的属性，在自定义组件movies.js中有一个properties来接收，properties定义组件的对外属性，属性值可以在组件使用时指定

```
// index.wxml
<movies movie-type="in-theaters"></movies>
// search.wxml
<movies movie-type="search"></movies>
```

```
Component({
    properties: {
        // 定义组件的对外属性，属性值可以在组件使用时指定
        movieType: {
            type: String,
            value: 'in_theaters',
        }
    },
    data:{}
    ...
})
```
所以我们就可以根据MovieType验证它的请求来源，来做相应的请求操作。改动的代码很少，只要在ready方法中加入一个判断即可

```
// component/movies/movies.js
Component({
    ...
    ready(){
        // 验证请求来源 搜索、首页
        let _url = 'in_theaters';
        if (this.properties.movieType == 'search') {
            // 搜索页面请求处理
            var _pages = getCurrentPages(),
                _curPage = _pages[_pages.length - 1];
            // 获取到搜索带过来的参数并复制到请求的url接口中
            _url = 'search?q=' + _curPage.options.q;
        }
        // 获取电影列表
        this.getMovieList(_url);
    }
    ...
})
```  
搜索功能也完成，是不是炒鸡简单，输入搜索内容后看下页面效果
![电影搜索.png](http://upload-images.jianshu.io/upload_images/5019151-c935047815e40f3c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

## 详情页  
电影详情只要根据ID查询对应的数据在展示页面，相信大家都知道怎么玩了，那就直接上代码了  

```
// pages/detail/detail.wxml
<wxs module="filter" src="../../common/filter.wxs"></wxs>

<view>
    <image class="movie-image" mode="scaleToFill" src="{{movieInfo.images.large}}"/>
</view>
<view class="padding-10">
    <view class="movie-title">{{movieInfo.title}}</view>
    <view>导演：{{filter.joinName(movieInfo.directors)}}</view>
    <view>主演：{{filter.joinName(movieInfo.casts)}}</view>
    <view class="movie-summary">简介{{utils.formatTime()}}</view>
    <view>{{movieInfo.summary}}</view>
</view>
```

```
// pages/detail/detail.wxss
.movie-image{
    width: 100%;
    height: 600rpx;
}
.movie-title{
    margin-top: 30rpx;
    font-size: 40rpx;
    font-weight: bold;
    color: #333;
}
.movie-summary{
    border-top: 1rpx solid #eee;
    margin-top: 20rpx;
    padding-top: 20rpx;
    color: #444;
    font-size: 32rpx;
}
```

```
// pages/detail/detail.json
{
  "navigationBarTitleText": "电影详情"
}
```

```
// pages/detail/detail.js
const app = getApp();
Page({
    // 组件的初始数据
    data: {
        movieInfo: {}
    },
    // 组件加载事件
    onLoad(options){
        let _this = this;
        wx.showLoading();
        wx.request({
            url: app.globalData.apiUrl + '/movie/subject/' + options.id,
            method: 'GET',
            success: function (res) {
                // 此处省略请求是否成功校验
                _this.setData({
                    movieInfo: res.data
                })
                _this.movieInfo = res.data;
                wx.hideLoading();
            }
        })
    }
})
```  
>PS:在这里我们要注意的一个是我们在detail.wxml文件中使用了过滤器，如果我们有时候需要对一些特殊字段进行处理的时候可以直接使用此方式。在wxs文件中不支持es6语法，否则会编译失败，这个也的注意下，希望后面小程序能解决此问题  

```
// common/filter.wxs
var filter = {
    joinName: function (arrObj) {
        var _arrName = [];
        if (!arrObj)return '';
        for (var i = 0; i < arrObj.length; i++) {
            _arrName.push(arrObj[i].name)
        }
        return _arrName.join('、')
    }
}

module.exports = {
    joinName: filter.joinName
}
```  
效果图  

![电影详情.png](http://upload-images.jianshu.io/upload_images/5019151-cfccb2f6080d79d5.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)  

## 完工  
好了，一个简单的豆瓣电影小程序功能就开发完成。新手练习可把此项目复制到自己的GitHub上，后在项目基础上加入一些简单的功能来巩固自己，如：下拉刷新、加载更多。在厉害点可加入豆瓣图书的功能，底部有两个tab标签，一个电影一个图书去扩展等等。  
>源码地址：[小程序-豆瓣电影](https://github.com/chenwenjin888/wx-douban)，如果觉得有帮助，可star一下啊