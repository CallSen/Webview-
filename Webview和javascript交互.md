# Webview-
操作前提

因为我们的目标是与 JavaScript 进行交互，因此 webView.getSettings().setJavaScriptEnabled(true); 当然是必须的。

通过 WebView 调用 JavaScript 方法

如果只是单纯地想要调用 JavaScript 的方法，可以直接使用 webView.loadUrl("javascript:METHOD") 来实现，METHOD 就是你期望的 JavaScript 方法，相当于直接在 Chrome 中按 F12 进入 Console 操作。

需要注意的是，调用 loadUrl() 在 4.4 之前的 WebView 中，会导致网页重载（4.4 之后现象无明显影响），而网页重载会导致软键盘输入焦点消失等问题。

向 JavaScript 中传递参数

首先我们需要注意的一件事是，JavaScript 是弱类型的，所以对于从 Java 中传递的参数，我们需要自己在 JavaScript 中处理，而不能想当然的传进去就用。

对于字符串形式的参数，一定要记住使用单引号 ' 将其包裹，否则 JavaScript（可能）会无法解析这个字符串，提示未定义。

除此意外还要注意一个坑，假设我们希望向 JavaScript 中传递字符串中包含 \ 或者 %5C 的子串，那么都会导致在 JavaScript 中的转义，因此我们需要对 \ 或 %5C 进行处理。比如对于 \ 我们就可以做一个字符串替换，变为 \\ ，而 \\ 会被转义为我们真正需要的 \ 。

剩下的事情就很简单啦，比如 JavaScript 中有这么一个方法：

// 插入一段 HTML
function insertHTML(html) {
    ...
} 
在 WebView 中我们就可以这么调用：

String html = ...;
html = html.replaceAll("\\\\", "\\\\\\\\");
webView.loadUrl("javascript:insertHTML('" + html + "')"); 
JavaScript 调用 Java 方法

想要在 JavaScript 中调用相关的 Java 方法，就需要对 WebView 做一些设置了。

首先我们需要在 WebView 中添加 JavaScript 可以直接调用的接口，为了方便起见我就直接将整个 WebView 作为接口对象传递到 addJavascriptInterface() 中，同时记住给接口对象起一个名字；

接着我们要给所有 JavaScript 可以调用的 Java 方法添加 @JavascriptInterface 的注解；

然后就可以在你的 JavaScript 代码里调用 Java 方法啦。整体代码看起来大概是这样的：

public class MyWebView extends WebView {
    public MyWebView(Context context) {
        ...
        getSettings().setJavaScriptEnabled(true);
        addJavascriptInterface(this, "MyName");
        ...
    }

    @JavascriptInterface
    public void example() {
        ...
    }
} 
在 JavaScript 中这样写：

// 调用 MyWebView 中的 example() 方法
function example() {
    MyName.example();
} 
获取 JavaScript 的返回值

很多时候我们使用 JavaScript 不止是执行一个方法，而是希望得到一个返回值，比如一个布尔值，或者一个字符串。

在 Android 4.4 及其以上版本中， Google 在 WebView 中提供了一个叫做 evaluateJavascript() 的方法，当你期望的 JavaScript 方法执行完毕时，会产生一个回调，传回返回值的字符串形式，因此你需要自己手动转换为所需的类型。具体的示例可以参考这个链接。

当然我们也可以使用简单并且兼容低版本 Android 的方法来实现。比如我们在 MyWebView 中有这么一个方法：

@JavascriptInterface
public void log(String tag, boolean value) {
    Log.v(tag, String.valueOf(value));
} 
在 JavaScript 中这样写：

function log() {
    MyName.log("tag", true);
} 
JavaBridge 会自动将参数转换为所需的类型，当然前提是你要保证转换一定正确。

关于 UI 层面的修改

Android 只允许在主线程中修改 UI 。如果你在 JavaScript 中调用了 WebView 的方法来修改界面，由于 JavaBridge 并不在主线程中，我们要新建一个位于主线程的 Handler 来操作 UI 。

public class MyWebView extends WebView {
    private Handler handler = new Handler(Looper.getMainLooper());

    ...

    @JavascriptInterface
    public void changeUI() {
        handler.post(new Runnable() {
            ...
        });
    }
} 
