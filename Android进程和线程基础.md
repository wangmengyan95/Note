对于一个基本的Android app，它所包含的Activity，Service等全部运行在一个进程里。

## Activity的实验

每个基本的Activity会对应三个Thread

* main thead 这个thread是由[ActivityThread](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/app/ActivityThread.java#L6116)这个类创建出的
* Binder_1
* Binder_2

## Service的实验

和Activity类似，每个基本的Service的实验会对应三个Thread

* main thead 这个thread是由[ActivityThread](https://github.com/android/platform_frameworks_base/blob/master/core/java/android/app/ActivityThread.java#L6116)这个类创建出的
* Binder_1
* Binder_2

## 多个Activity,Service的实验
为了验证不同的Activity，Service都运行于同一个进程，进行一个简单的实验。创建一个空的Activity和Service，两个类包含成员变量`public static int TEST`。
