---
title: 百度地图离线API制作
date: 2016-12-08 15:47:17
tags:
- 百度地图
- 工具
categories:
- 开发工具
---
{% asset_img baiduMap.jpg 配图木有名字 %}
百度地图离线API制作，以API v1.3为例。更高版本请自行搜索。

## 第一步
访问 [http://api.map.baidu.com/api?v=1.3](http://api.map.baidu.com/api?v=1.3) 得到如下内容
```javascript
(function(){
    window.wise=1;
    window.netSpeed=254;
    window.netType=1;
    window.BMap_loadScriptTime = (new Date).getTime();
    document.write('<script type="text/javascript" src="http://api.map.baidu.com/getscript?v=1.3&ak=&services=&t=20150527115231"></script>');
    document.write('<link rel="stylesheet" type="text/css" href="http://api.map.baidu.com/res/13/bmap.css"/>');
})();
```
这一步将获得一个js文件的位置和一个css文件的位置，然后分别下载下来，放在准备好的文件夹里面，我分别存储在/js和/css里面。js的路径后面有若干参数，不管，下载下来的文件重新命名就好，比如js文件命名为`apiv1.3.min.js`

## 第二步
修改这个js中的代码。由于代码是压缩在一行上的，可通过网上的代码在线格式化。
1. 搜索变量：`imgPath`，直到找到一段代码
    ```javascript
    var x=m?"https://sapi.map.baidu.com/":"http://api.map.baidu.com/";
    var cd={imgPath:x+"images/",cityNames:{"\u5317\u4eac":"bj",
    ```
    把`imgPath:x+"images/"`中的x去掉即可，这样就变成了跟网络地址无关的相对位置了。这里指出的`images`文件夹，是与将来的html文件夹平行的目录里面的。
    例如：
    ```javascript
    var x=m?"https://sapi.map.baidu.com/":"http://api.map.baidu.com/";
    var cd={imgPath:"images/",cityNames:{"\u5317\u4eac":"bj",
    ```
1. 搜索变量：`_baseUrl`，直到找到这样的一段代码
    ```javascript
    preLoaded:{},Config:{_baseUrl:x+"getmodules?v=1.3",_timeout:5000},
    ```
    同样，要去掉x，这个x也是前面这段代码中的x。同时，不仅要去掉x，更要把地址指向js目录。
    例如：
    ```javascript
    preLoaded:{},Config:{_baseUrl:"js/",_timeout:5000},
    ```
1. 继续搜索下一个`_baseUrl`，得到下面的代码：
    ```javascript
    window.setTimeout(function(){
        var cP=cN.Config._baseUrl+"&mod="+cN.Module._arrMdls.join(",");
        cy.request(cP);
        cN.Module._arrMdls.length=0;
        cN.delayFlag=false;
    },1);
    ```
    直接修改`cP`变量（变量名与版本有关，可能不同），注意：这里把加载地址直接写死
    ```javascript
    var cP=cN.Config._baseUrl+"modules";
    cy.request(cP);
    ```
    这样，我们把动态模块加载改为了手动加载离线模块文件。
    访问这个地址 [http://api.map.baidu.com/getmodules?v=1.3&mod=map,oppc,tile,control,marker,poly](http://api.map.baidu.com/getmodules?v=1.3&mod=map,oppc,tile,control,marker,poly) 我们就能拿到相应参数的模块源文件。
    新建文件`modules`，放在js文件夹里面即可。
1. 关键的一步：搜索`getTilesUrl`直到找到这样的一段（变量名aU可能不一样）
```javascript
aU.getTilesUrl = function(cN, cQ) {
    var cR = cN.x;
    var cO = cN.y;
    var T = "20150518";
    var cP = "pl";
    if (this.map.highResolutionEnabled()) {
        cP = "ph"
    }
    var cM = j[Math.abs(cR + cO) % j.length] + "?qt=tile&x="
		+ (cR + "").replace(/-/gi, "M") + "&y="
		+ (cO + "").replace(/-/gi, "M") + "&z=" + cQ + "&styles=" + cP
        + (a9.browser.ie == 6 ? "&color_dep=32&colors=50" : "")
        + "&udt=" + T;
    return cM.replace(/-(\d+)/gi, "M$1")
};
window.BMAP_NORMAL_MAP = new cv("\u5730\u56fe", aU, {
    tips : "\u663e\u793a\u666e\u901a\u5730\u56fe"
});
```
`getTilesUrl`这个方法，这里就是返回瓦片未知的关键方法。两个参数中，第一个参数是{x，y}，第二个参数就是z，这样xyz就都有了。
直接把它计算出来的cM的结果重新计算一下，改成：
```javascript
cM = "tiles/" + cQ + "/" + cR + "/" + cO + ".jpg";
```
.jpg后缀与你下载的瓦片图片格式有关。
这样就设置了`BMAP_NORMAL_MAP`地图类型的地图瓦片获取url，即为放在tiles目录下的瓦片资源。同理可以设置卫星`BMAP_SATELLITE_MAP`等地图类型的离线瓦片url。

## 第三步
下载相关资源
2. 图片文件
像一些logo等图片资源。可通过chrome开发者工具查看缺少的图片文件，以及url。然后自行下载。将图片放在`images`文件夹中，并修改`bmap.css`中相应的url为本地路径。
2. 依赖模块API文件
    如果缺少某个依赖模块，则无法使用相应的API。
    在运行前期工作中的在线地图时，就可发现，依赖的库参数是什么。例如：以下的代码运行，所请求的依赖库参数是[http://api.map.baidu.com/getmodules?v=1.3&mod=map,oppc,tile,control](http://api.map.baidu.com/getmodules?v=1.3&mod=map,oppc,tile,control)
    ```html
    <script type="text/javascript">
        var map = new BMap.Map("container",{mapType:BMAP_NORMAL_MAP});
        var point = new BMap.Point(116.404, 39.915);    // 创建点坐标
        map.centerAndZoom(point,5);                     // 初始化地图,设置中心点坐标和地图级别。

        //map.addControl(new BMap.MapTypeControl());
        map.addControl(new BMap.NavigationControl());
        map.enableScrollWheelZoom();                  // 启用滚轮放大缩小。
        map.enableKeyboard();                         // 启用键盘操作。
        //map.setCurrentCity("北京");          // 设置地图显示的城市 此项是必须设置的
    </script>
    ```
    总共有哪些依赖模块可以去`apiv1.3.min.js`中搜索`Dependency`
2. 瓦片资源
可以选择 `全能电子地图下载器` 下载，其中`1.9.5`版本有破解，不过有些瓦片无法下载。或者选择自己拼凑瓦片，有教程获取瓦片url。

## 最后
离线API算是做好了，目录结构大致为：
- css
    - bmap.css
- images
- js
    - apiv1.3.min.js
	- modules
- tiles

[HawkUI](https://github.com/lakehumin/hawkui)项目中便用到了该离线API，文件在`public`目录下。