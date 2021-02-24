# Android O SurfaceView调用setVisibility的错误日志

## 错误体现

- 日志：Failed to find layer SurfaceView XXX in layer parent(no-parent).
- 行为：只要SurfaceView对象调用setVisibility(隐藏）即可出现

## 修复patch,亲测有效

- <https://cs.android.com/android/_/android/platform/frameworks/native/+/0617894190ea0c3ee50889bee1d4df0f369b0761>
- <https://cs.android.com/android/_/android/platform/frameworks/native/+/8b3871addb9bbd5776f4ed59e67af2baa9c583fd>

## 验证
- cd frameworks/native/services/surfaceflinger mma -j64 单编即可
- push libgui.so libsurfaceflinger.so 到对应的目录 reboot 起来跑demo

## 原理（待补充）
