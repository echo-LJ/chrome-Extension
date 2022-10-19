# 需求

在chrome浏览器养一只喵拢共分几步？

-   往chrome浏览器注入代码（chrome插件）
-   网页里展示一只猫（live2d模型）


# 实现

##  新建文件`manifest.json`文件
**manifest.json：** 是一个Chrome插件最重要也是必不可少的文件，用来配置所有和插件相关的配置，必须放在根目录。其中，`manifest_version`、`name`、`version3`个是必需的，`description`和`icons`是推荐的。

下面给出的是一些常见的配置项，完整的配置文档请戳[这里](https://developer.chrome.com/extensions/manifest?utm_source=devtools): 

```
{
	"manifest_version": 2, // 配置清单文件的版本，这个必须写，而且必须是2
	"name": "养猫", // 插件的名称。
	"version": "0.0.1", // 插件的版本。
	"description": "在chrome浏览器撸猫并使用猫语实时聊天", //插件描述。
        // * browser_action: 浏览器右上角图标设置，browser_action、page_action、app必须三选一。
	"icons": // 插件的logo图标。
	{
		"16": "resources/icon16.png",
		"48": "resources/icon48.png",
		"128": "resources/icon128.png"
	},
	"content_scripts":  //需要直接注入页面的JS，因为类型是数组，可以配置多个配置多个规则。
	[
		{
			"matches": ["<all_urls>"],  //["<all_urls>"] 表示匹配所有地址。
                        
                        // matches: ["http://*/*", "https://*/*"] 表示部分地址,matches字段不支持正则，需要模糊匹配的部分用*号。
                        
			"js": ["js/index.js"], // 多个JS按顺序注入。
			"run_at": "document_end" // 码注入的时间，可选值： "document_start", "document_end", or "document_idle"，最后一个表示页面空闲时，默认document_idle。
		}
	],
	"permissions": // 权限申请。
	[
		"webRequest", // web请求。
		"webRequestBlocking", // 让 webRequest 可以 block 网络请求。
		"storage", //  插件本地存储。
		"http://*/*",
		"https://*/*"
                //  contextMenus: 右键菜单。
                //  tabs: 标签。
                // notifications: 通知。
	],
        "web_accessible_resources": ["js/live2d-mini.js"], // 普通页面能够直接访问的插件资源列表，如果不设置是无法直接访问的。
	"homepage_url": "https://github.com/ezshine/chrome-extension-catroom",
        
        // content_scripts中要使用的插件本地资源，都需要在web_accessible_resources字段中注册，否则无法使用。
	"chrome_url_overrides":
	{
		
	}
}
```


## 引入`live2d-mini.js`第三方库

`Live2D`是一种应用于电子游戏的绘图渲染技术，该技术由日本Cybernoids公司（现已更名为Live2D）开发。通过一系列的连续图像和人物建模来生成一种类似三维模型的二维图像，对于以动画风格为主的游戏来说非常有用。

简单来说，Live2D就是一项在不增加绘图工作量的基础上，让2D人物“活”起来的技术。

例如：画师只需要绘制一张原画，然后将身体、五官、发型、服装等2D图片放到全3D的模型模板上。

成品将保留2D手绘的精细，同时在小范围旋转视角上也能做到毫无违和感。不过这一技术目前的缺点在于无法实现复杂图形、人物的大幅旋转。

如果对相关制作感兴趣的朋友可以看看Live2D的[官方网站](https://www.live2d.com/)，里面都有对应的制作工具和相关教程介绍。
* 目录的层级结构如下：

![截屏2022-10-18 上午10.20.05.png](https://p6-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/fcf8fcb4e3a3466081be3b18012ed87f~tplv-k3u1fbpfcp-watermark.image?)

## 编写js/index.js

 * 首先创建一个script标签，将live2d-mini.js动态的加载到网页中

 ```
 function injectCustomJs(jsPath,cb)
{
    var temp = document.createElement('script');
    temp.setAttribute('type', 'text/javascript');
    temp.src = chrome.extension.getURL(jsPath);
    temp.onload = function()
    {
        this.parentNode.removeChild(this);
        if(cb)cb();
    };
    document.head.appendChild(temp);
}
injectCustomJs('js/live2d-mini.js',function(){
    //加载成功后准备设置画布并显示猫咪模型
});
```
* 创建一个顶层canvas，作为用来显示猫咪的画布。
```
function setupCatPanel(){
        var canvas = document.createElement('canvas');
        canvas.id="live2d";
        canvas.width = 300;
        canvas.height = 400;
        canvas.style.width='150px';
        canvas.style.height='200px';
        canvas.style.position = "fixed";
        canvas.style.zIndex = 9999;
        canvas.style.right = 0;
        canvas.style.bottom = 0;
        canvas.style.pointerEvents='none';
        canvas.style.filter='drop-shadow(0px 10px 10px #ccc)';
  
        document.body.appendChild(canvas);
}
```
* 创建猫的模型数据，通过`live2d-mini.js`中的`loadlive2d`方法将猫渲染到画布中
 ```
 function setupModel(){
        var _cattype = localStorage.getItem('cattype');
        if(_cattype)cattype=_cattype;

        var model_url= 'https://cdn.jsdelivr.net/gh/ezshine/chrome-extension-catroom/src/resources/'+cattype+'cat/model.json';

        var loadLive = document.createElement("script");
        loadLive.innerHTML = '!function(){loadlive2d("live2d", "' + model_url + '");}()';
        document.body.appendChild(loadLive);
    }

 ```
 * 完整代码如下：
 ```
 /*
猫房
author:echo
reposity:https://github.com/ezshine/chrome-extension-catroom
*/

!function(){
    var cattype = "white";

    function setupCatPanel(){
        var canvas = document.createElement('canvas');
        canvas.id="live2d";
        canvas.width = 300;
        canvas.height = 400;
        canvas.style.width='150px';
        canvas.style.height='200px';
        canvas.style.position = "fixed";
        canvas.style.zIndex = 9999;
        canvas.style.right = '0px';
        canvas.style.bottom = '-10px';
        canvas.style.pointerEvents='none';
        canvas.style.filter='drop-shadow(0px 10px 10px #ccc)';
  
        document.body.appendChild(canvas);
     }
  
     function setupModel(){
        var _cattype = localStorage.getItem('cattype');
        if(_cattype)cattype=_cattype;

        var model_url= 'https://cdn.jsdelivr.net/gh/ezshine/chrome-extension-catroom/src/resources/'+cattype+'cat/model.json';

        var loadLive = document.createElement("script");
        loadLive.innerHTML = '!function(){loadlive2d("live2d", "' + model_url + '");}()';
        document.body.appendChild(loadLive);
    }

    function injectCustomJs(jsPath,cb)
    {
        var temp = document.createElement('script');
        temp.setAttribute('type', 'text/javascript');
        temp.src = chrome.extension.getURL(jsPath);
        temp.onload = function()
        {
            this.parentNode.removeChild(this);
            if(cb)cb();
        };
        document.head.appendChild(temp);
    }

    injectCustomJs('js/live2d-mini.js',function(){
      setupCatPanel();
      setupModel();
   });
}()
```
实现效果如下：

![屏幕录制2022-10-18-上午10.09.03.gif](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/07adfe0066bf4a75b42137f33f37f9a6~tplv-k3u1fbpfcp-watermark.image?)


## chrome引入插件
 * 打开菜单的更多工具
 * 打开扩展程序
 * 选择加载已解压的扩展程序
 * 选择manifest.json文件所在的目录
 
![截屏2022-10-18 上午10.18.32.png](https://p9-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/36a457922fe84a3c8d9170b0f84410e2~tplv-k3u1fbpfcp-watermark.image?)
			
上述代码写完之后，在扩展程序里刷新一下这个插件就可以在掘金网页里愉快的撸猫啦~

# 总结
本文旨在学习如何实现chrome插件，并实现了在chrome浏览器上撸猫的一个插件，后续将继续学习，希望可以在chrome浏览器撸猫并使用猫语实时聊天。

# 参考链接


1、 [我开发了一个Chrome插件，可以在掘金官网里撸猫！还可以实时和铲屎官们聊](https://juejin.cn/post/7026180392755888142)

2、 [【干货】Chrome插件(扩展)开发全攻略](https://www.cnblogs.com/liuxianan/p/chrome-plugin-develop.html)