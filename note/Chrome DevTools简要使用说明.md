Chrome DevTools是内置在Chrome浏览器中的一套Web开发工具，可以实时编辑网页以及定位网页中的问题。

## 打开DevTools
有如下方式可以打开DevTools
- 在Chrome浏览器网页上任意位置点击右键，在弹出菜单中选择“检查”
- 在网页上的某个元素（如网页上的一张图片）上点右键，选择“检查”，这样打开DevTools后将直接定位到此元素上

## DOM
打开DevTools后切换到”Elements“标签，点击网页源码中的元素，如`<body>`，网页上会高亮展示所选择的内容

## CSS
选中“Elements”标签，在“Elements”标签下的面板上右侧选中“Styles”标签，在“element.style”中添加设置项目`background-color: blue`，例如修改[https://devtools.glitch.me/network/getstarted.html](https://devtools.glitch.me/network/getstarted.html)中“Get Data”按钮的背景

## Console
### 查看Console消息
选中“Console”标签，输入`console.log('Hello, Console!')`就能在Console中看到输出的内容

[示例](https://devtools.glitch.me/console/log.html)

有的网站会在首页添加一些console消息，例如：https://www.baidu.com/，https://sina.cn，https://www.zhihu.com/等

如果网页上的内容在加载时异常，也会在Console中输出相应的消息，可以帮助我们定位网页加载中的问题

### 在Console中运行JavaScript
例如在Console中输入：1+2，将输出结果3

也可以直接在Console中输入代码定义函数然后调用，例如定义函数
```
function add(a, b) {
    return a+b;
}
```
接着执行`add(1, 2);`将得到结果3

## 网络
打开网址[https://devtools.glitch.me/network/getstarted.html](https://devtools.glitch.me/network/getstarted.html)，将DevTools切换到“Network”标签，然后刷新网页，能看到所有的网络请求，单独点击一个网络请求，可以看到此请求的详情，如网络请求Header等。

另外，在“Network”标签下查看网络请求时，可以看到网络请求的“Name”、“Status”等，在此标题栏上点击右键，可以选择展示更多的项目，如网络请求的“Method”。

## 命令菜单
打开DevTools，点击“Customize And Control DevTools”按钮，在下拉菜单中选择“Run Command”即可打开命令菜单弹窗，例如在菜单中输入”java“，选择”Disable Javascript“，使用**网络**中的示例，点击”Get Data“按钮，就无法执行JS了。

## JavaScript调试
### 调试
打开[示例网址](https://googlechrome.github.io/devtools-samples/debug-js/get-started)，在两个输入框分别输入一个数字，点击按钮，发现未得到期望的结果，此时就可以使用DevTools的调试功能。具体步骤为：打开”Sources“标签页，找到可能出现问题的代码处，像使用普通IDE那样打上断点，再次在网页上点击按钮触发操作，即可开始调试。

### 修正
当找到可能存在的问题后，可以直接在源代码上进行修改，如在本例中将`getNumber1()`改为`parseInt(getNumber1())`，`getNumber2()`改为`parseInt(getNumber2())`，保存，再次运行，会发现得到了正确的结果

## 远程调试
在Chrome地址栏输入[chrome://inspect](chrome://inspect)，将安卓手机连接到电脑上，如果手机上有应用使用了WebView，在Chrome上就能看到手机应用中的WebView，在WebView下点击”inspect“打开DevTools，DevTools的使用方式和Chrome使用方式完全一致。

注意：在Android应用中需要对WebView进行如下设置
```
if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
    WebView.setWebContentsDebuggingEnabled(true);
}
```

## 小技巧
- 点击DevTools面板左上角的”Select an element in the page to inspect it“后，点击网页上的元素，可以快速在网页源码中定位到此元素
- 点击DevTools面板左上角的”Toggle Device toolbar“可以查看在手机模式下的页面样式
- 点击DevTools面板顶端右侧“Customize and control DevTools”按钮，点击“Dock side”可以切换面板的放置方式
- 在DevTools打开时，长按Chrome地址栏的刷新按钮，会弹出选项列表，可以选择一种刷新方式，如“清空缓存并硬性重新加载”
- 点击DevTools面板顶端右侧“Customize and control DevTools”按钮，点击“Settings”，在Appearance中选择”Dark“可将DevTools切换到深色主题

## 参考
[Chrome DevTools](https://developers.google.com/web/tools/chrome-devtools/)