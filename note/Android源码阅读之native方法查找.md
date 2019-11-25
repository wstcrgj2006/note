当阅读framework层代码时经常会看到`native`方法，这些方法通常都是由C/C++代码实现，使用JNI技术进行调用，那么在源码中应该怎么查找`native`方法对应的具体实现呢？

## 法1：按类名全限定名查找

例如`MessageQueue.java`中有native方法`private native static long nativeInit();`，在`/framework/base/core/jni/`目录下查找`android_os_MessageQueue.cpp`文件，查找到文件后，按函数的全限定名`android_os_MessageQueue_nativeInit`在文件中搜索即可找到函数实现。

## 法2：按函数全限定名搜索

某些类使用“按类名全限定名查找”的方法无法找到，此时可以尝试按照函数全限定名在目录的文件中搜索。例如`android.os.Process.java`，搜索`android_os_Process.cpp`无法找到，此时可以在`/framework/base/core/jni/`目录下搜索`android_os_Process_setThreadPriority`，查找到文件`android_util_Process.cpp`。

> 备注：类似情形的其他类如`android.os.Binder.java`

## 法3：在线搜索

有时候使用法1和法2都无法搜索到，例如`android.database.sqlite.SQLiteConnection#nativeExecute`，此时可以进入[AndroidXRef](http://androidxref.com)，选择想要查看源码的分支，打开搜索页面后，在“Symbol”输入框中输入`nativeExecute`，“In Project(s)”选择“select all”，点击“Search”按钮即可得到搜索结果，此处为`/frameworks/base/core/jni/android_database_SQLiteConnection.cpp`。

如果搜索文件，例如"Thread.java"，可以在“File Path”输入框中输出文件名`Thread.java`，然后搜索即可。

这种方法对于查找native函数调用很方便。

## native函数实现常见目录

* /framework/base/core/jni/
* /framework/base/services/core/jni/
* /framework/base/media/jni/
