Name
====

a little library for using [Chrome DevTools Protocol](https://chromedevtools.github.io/devtools-protocol/) in [aardio](http://www.aardio.com/)

Table of Contents
=================

* [Name](#name)
* [Dependents](#Dependents)
* [Attributes](#Attributes)
	* [timeout](#timeout)
	* [lastTaskDuration](#lastTaskDuration)
	* [onTaskBegin](#onTaskBegin)
	* [onTaskEnd](#onTaskEnd)
* [Methods](#Methods)
    * [open](#open)
    * [connect](#connect)
    * [waitEvent](#waitEvent)
    * [run](#run)
* [Properties](#Properties)
* [Examples](#Examples)
	* [获取节点内容](#获取节点内容)

Dependents
==========

* [HP-Socket-aardio](https://github.com/btx638/HP-Socket-aardio)

Attributes
==========

Methods
=======

Properties
==========

Examples
=======

获取节点内容
------------
* 运行前请把谷歌浏览器更新到最新版本，低版本对 windows 下的无头模式支持不好

![运行动画](https://raw.githubusercontent.com/btx638/dp/master/aaz/chrome/dp/example/3.gif)

````javascript
import win.ui;
/*DSG{{*/
var winform = win.form(text="aardio form";right=663;bottom=383)
winform.add(
button={cls="button";text="运行";left=552;top=320;right=650;bottom=348;z=1};
edSelector={cls="edit";text=".items-area .item dl dt a";left=72;top=320;right=416;bottom=348;edge=1;z=6};
edUrl={cls="edit";text="https://www.cnbeta.com/";left=72;top=280;right=416;bottom=308;edge=1;z=4};
lvResult={cls="listview";left=9;top=7;right=652;bottom=271;edge=1;fullRow=1;z=2};
static={cls="static";text="url";left=11;top=286;right=64;bottom=310;align="right";transparent=1;z=3};
static2={cls="static";text="selector ";left=8;top=320;right=63;bottom=344;align="right";transparent=1;z=5}
)
/*}}*/

io.open()

import win.ui.statusbar;
import aaz.chrome.dp;

var cdp, err = aaz.chrome.dp()
if(!cdp){
    winform.msgboxErr(err);
	return ; 
}
cdp.timeout = 9000;

var statusbar = win.ui.statusbar(winform);
statusbar.addItem("运行状态：", 70);
statusbar.addItem("就绪", -1);

winform.lvResult.insertColumn("序号",50);
winform.lvResult.insertColumn("标题",-1);

var uiLog = function(str){
	statusbar.setText(str, 2);
}

var currentTaskStepName;
var uiLogRunning = function(name){
    var str = string.format("正在%s...", name);
    currentTaskStepName = name;
	uiLog(str);	
}

var uiLogFail = function(name, err){
    name := currentTaskStepName;
    var str = string.format("%s失败，原因：%s", name, err);
	uiLog(str);
}

var uiResult = function(str){
	winform.lvResult.addItem({
		tostring(winform.lvResult.count+1);
		str
	});
}

var uiInit = function(){
	uiLog("任务开始", 2);
	winform.lvResult.clear();
}

var closeChrome = function(){
	return cdp.Browser.close(); 
}

var doFail = function(name, err){
    uiLogFail(name, err);
	closeChrome();
}

// 任务开始时触发
cdp.onTaskBegin = function(){
	winform.button.disabled = true
	currentTaskStepName = null;
	uiInit();
}

// 任务结束时触发，可以接收任务函数的返回值
cdp.onTaskEnd = function(result, err){
	winform.button.disabled = false
	
	if(!result){
		doFail(currentTaskStepName, err)
		return ; 
	}
	
	uiLog("任务完成，用时：" ++ cdp.lastTaskDuration ++ "毫秒")
	
	// 读取结果
	for(i=1;#result;1){
		uiResult(result[i]);
	}
}

var task = function(url, selector){
 	uiLogRunning("打开浏览器");
 	// 开启无头模式，没有界面
	var ok, err = cdp.open(null, true);
    if(!ok){
    	return null, err; 
    }
	
	uiLogRunning("连接浏览器");
    var ok, err = cdp.connect();
    if(!ok){
    	return null, err; 
    }
    
	uiLogRunning("订阅 Page 事件");	
	var ok, err = cdp.Page.enable();
    if(!ok){
    	return null, err; 
    }
	
	// 打开网址
	uiLogRunning("打开网址");	
	var ok, err = cdp.Page.navigate(
		url = url;
	)
    if(!ok){
    	return null, err; 
    }
	
	uiLogRunning("等待页面加载完成");
	var ok, err = cdp.waitEvent("Page.loadEventFired");
    if(!ok){
    	return null, err; 
    }
 
	uiLogRunning("开启 runtime");
	var ok, err = cdp.Runtime.enable()
	if(!ok){
		return null, err; 
	}
	
	uiLogRunning("编译脚本");
	var ret, err = cdp.Runtime.compileScript(
		sourceURL = url;
		persistScript = true;
		expression = /**
		 var getInnerText = function(selector){
       		var doms = document.querySelectorAll(selector);
       		var ret = new Array();
       		for(var i=0;i<doms.length;i++){
           		ret.push(doms[i].innerText);
       		}
       		return ret; 
		 }
		**/
	)

	if(!ret){
    	return null, err; 
	}
	// 检查编译的结果
	// 发现错误
	if(ret.exceptionDetails){
    	return 
    		null, 
        	" 行号：" ++ ret.exceptionDetails.lineNumber ++
        	" 列号：" ++ ret.exceptionDetails.columnNumber ++ 
        	" 内容：" ++ ret.exceptionDetails.exception.description; 
	}
	
	
	uiLogRunning("运行脚本");
	var ret, err = cdp.Runtime.runScript(
		scriptId = 	ret.scriptId;
		returnByValue = true;
	)
	if(!ret){
		return null, err; 
	}
	
	uiLogRunning("对全局对象的表达式求值");
	var ret, err = cdp.Runtime.evaluate(
		expression = `getInnerText("` ++ selector ++ `")`;
		returnByValue = true;
	)
	if(!ret){
		return null, err; 
	}
	var result = ret.result;
	
	// 检测结果的类型
	// 看看有没有错误
	select(result.subtype) {
		case "error" {
			return null, result.description; 
		}
	}

	uiLogRunning("关闭浏览器");
	var ok, err = closeChrome();
	if(!ok){
		return null, err; 
	}
	
	// 返回结果
	return result.value; 
}

winform.button.oncommand = function(id,event){
	assert(cdp.run(task, winform.edUrl.text, winform.edSelector.text))
}

winform.show();
win.loopMessage();
return winform;
````

[Back to TOC](#table-of-contents)