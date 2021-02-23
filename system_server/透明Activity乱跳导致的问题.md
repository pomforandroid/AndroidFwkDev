# 透明Activity跳转导致底下页面从Launcher乱跳到其他页面的问题
## 情景
- 当在Launcher启动了Activity A后，接着启动 Acitivity B ，然后Acitivity A 在启动Acitivity B的前后 finish掉了，
  此时透明Acitivity B底下本应是launcher的刷的一声转到了之前的一个Acitivity C。
  这个Acitivity C 很巧合每一次都是在Launcher 启动 Acivitivity A之前 的一个步骤—“启动了Activity C 然后回到launcher ”。



## 关键点

-  Activity A 是通过NEW_TASK 这个FLAG 启动，并且在启动Acvitivity B 前后 finish掉了。那么问题来了这两个条件去掉一个是不是就不会出现这种问题了。
   是的，不通过NEW_TASK启动的Acitivity A，即使finish掉了，底下也不会乱跳。 以及，只要Acitivity A不finIsh掉的话也不会出现这种问题

## stack
- Home Stack Id 0
- Focuse Stack Id 1

## FWK分析
- ActivityStackSupervisor.java 里面 ensureAcitvitiesVisibleLocked
- ensureAcitvitiesVisibleLocked方法里也是按从1先开始遍历，也就是说先去更新Focuse Stack。
- 执行到具体的每一个Stack 的ensureActivitiesVisibleLocked 时，去到 ActivityStack.java的ensureActivitiesVisibleLocked 方法。

## 分析关键
- 这里面有个很关键的点 ，就是会判断一个Acivitiy如果是finish状态的话就会跳过。
  这样的话会导致经历了Stack 1的遍历后 处于Stack 1 的top visible的Acivity是 Activity C， 而不是透明Activity A。

- 以至于到了 Stack 0的遍历时，Activity Launcher 会由于Stack 1 top visible的Acvitity是个非透明、全屏Activity C ，所以Launcher的visbile设置成False。
  最后的现象就是Acitivity C 变成可见了， Launcher变得不可见了。然后系统根据Acvititys 的visible 状态重新绘制。结果就是我们看到的乱跳现象了。

````
    if(r.finishing){
      ...
      continue;
    }
````
