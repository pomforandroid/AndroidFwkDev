# Android O FBE 引起的灾难性问题

## 表现

- 恢复出厂或者开机后所有应用无法打开
- 崩溃报错没有data/data下面的数据库等
-
    ````
    E vold : Failed to get encryption policy for /data/data: No such file or directory
    Line 39967: 07-01 08:00:05.832   666   672 E vold    : Failed to set policy on: /data/data
    ````
## 原因

- 系统应用设置了android:directBootAware="true"但是没有设置android:defaultToDeviceProtectedStorage="true"
- ps: 放在system/app下等目录的就算是系统应用了。
- 这个会导致当FBE还没有加密完成的时候就执行 mkdir file in data/data/apk_package/cache ，从而导致FBE加密失败
- 最后当after user unlock device ,  will send unlock_user_key,prepare_user_storage to vold . 然后就会导致 vold cannot set policy, and data/data/  data/user/0 will not mount

## 修改方案

- Android P 已经修复 前后有两笔
- https://android.googlesource.com/platform/frameworks/base.git/+/ac094513e0af65b081be3b0334b0046ba80dfd92%5E%21/#F0


## 思考

- 官方现在是建议android:directBootAware="true"需要设置android:defaultToDeviceProtectedStorage="true"
- CE空间和DE空间的区别。DE空间允许unlock之前的mkdir操作，比较安全，FBE加密针对的是CE空间

## FBE 和 UnlockUser的关系
- 后面补充
