# AccessibilityManagerService 导致system_server crash

## crash体现

- AccessibilityManagerService Failed registering death link 后执行resetLocked中unlinkToDeath引起的NoSuchElementException从而Crash System_server


## 原因

- 当onServiceConnected后mService(客户端的binder对象)，客户端进程died了
- 以至于该binder对象所在的进程不在了，所以执行linktoDeath方法失败
- 走到了下面的binderDied()方法，里面应该是一些清理和重置的逻辑，其中resetLocked()里面执行了unlinkToDeath。

## 修改方案

- resetLocked()方法里面对unlinkeToDeatch进行try catch。
- 之前的代码在unlinkeToDeatch下面mService就会被设置为null，并且没有linkToDeath成功
- unlinkToDeath失败也不会有影响，所以此修改应该是安全的

## 思考

- 使用无障碍服务的客户端进程一般都是有保活，出异常后会立马重启
- 但是遇到重启进程也无法解决的错误后会出现不断的频繁重启
- 从而出现这种进程died了，刚好卡在linktoDeath方法失败的时机，导致问题出现

## 调试和验证
- Android Studio Debug AccessibilityManagerServer的linktoDeatch,停住，然后把客户端进程kill接着走下去。
- 观察代码修改前后的系统表现，修改ok！不再崩溃。
