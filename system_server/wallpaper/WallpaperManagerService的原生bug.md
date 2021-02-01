# WallpaperManagerService

## LiveWallpaperPreview

-
  ```
    mWallpaperIntent = new Intent(WallpaperService.SERVICE_INTERFACE)
                .setClassName(info.getPackageName(), info.getServiceName());
  ```
-
    ```
    private void setLiveWallpaper(IBinder windowToken) {
        mWallpaperManager.setWallpaperComponent(mWallpaperIntent.getComponent());
        mWallpaperManager.setWallpaperOffsetSteps(0.5f, 0.0f);
        mWallpaperManager.setWallpaperOffsets(windowToken, 0.5f, 0.0f);
    }
    ```
## WallpaperManager
-
  ```
   @SystemApi
    @RequiresPermission(android.Manifest.permission.SET_WALLPAPER_COMPONENT)
    public boolean setWallpaperComponent(ComponentName name) {
        return setWallpaperComponent(name, mContext.getUserId());
    }
  ```

## WallpaperManagerService

### setWallpaperComponentChecked &darr;
  -
    ```
      //严格的判断条件，可能是之前Launcher设置不成功的原因`
      if (isWallpaperSupported(callingPackage) && isSetWallpaperAllowed(callingPackage)) {
          setWallpaperComponent(name, userId);
      }
    ```
### setWallpaperComponent &darr;

### bindWallpaperComponentLocked 把设置的wallpaperservicebind起来 &darr;

-
  ```
    //开头就有return true的条件，避免重复执行，但是这里会导致问题
    //force传入为false
    if (!force && changingToSame(componentName, wallpaper)) {
        return true;
    }
  ```
-
  ```
    // 通过bindserviceAsUser把设置的服务拉起
    if (!mContext.bindServiceAsUser(intent, newConn,
                    Context.BIND_AUTO_CREATE | Context.BIND_SHOWING_UI
                            | Context.BIND_FOREGROUND_SERVICE_WHILE_AWAKE,
                    new UserHandle(serviceUserId))) {
      ...
      wallpaper.connection = newConn;
    }
   ```
-
  ```
  private boolean changingToSame(ComponentName componentName, WallpaperData wallpaper) {
      if (wallpaper.connection != null) {
          if (wallpaper.wallpaperComponent == null) {
              if (componentName == null) {
                  if (DEBUG) Slog.v(TAG, "changingToSame: still using default");
                  // Still using default wallpaper.
                  return true;
              }
          } else if (wallpaper.wallpaperComponent.equals(componentName)) {
              // Changing to same wallpaper.
              if (DEBUG) Slog.v(TAG, "same wallpaper");
              return true;
          }
      }
      return false;
  }

  ```
- 出现问题的情景：
  执行了bindServiceAsUser返回结果true并不代表一定保证服务启动成功
  当该通过bindServiceAsUser 启动该服务却启动超时时
  日志体现:
  `am_process_start_timeout: [0,2499,10015,com.rightware.kanzi.carmodel]`
  `ActivityManager: Forcing bringing down service: ServiceRecord{bfd0085 u0 com.rightware.kanzi.carmodel/.LiveWallpaperService`
  所以会wallpaper.connection = newConn，等到后面进程重启，重新执行setWallpaperComponent后，changingToSame就会return true
  导致再也无法设置壁纸。


# bindService流程

## 抽象类context `bindService`
## 实现类ContextImpl
- bindService &darr;
- bindServiceCommon 通过binder调用到AMS
  -
    ```
      int res = ActivityManager.getService().bindService(
                  mMainThread.getApplicationThread(), getActivityToken(), service,
                  service.resolveTypeIfNeeded(getContentResolver()),
                    sd, flags, getOpPackageName(), user.getIdentifier());
    ```

- AMS.bindService &darr;
- ActiveServices.bindServiceLocked
-
  ```
  //简单理解成这个service存不存在，合不合理，成不成立，下面这两个条件不为true，后面就return 1,
  //也就是这个bindservice的结果返回了，所以并不能保证服务一定能启动
    ServiceLookupResult res =
            retrieveServiceLocked(service, resolvedType, callingPackage, Binder.getCallingPid(),
                    Binder.getCallingUid(), userId, true, callerFg, isBindExternal, allowInstant);
    if (res == null) {
            return 0;
    }
    if (res.record == null) {
        return -1;
    }
  ```
- requestServiceBindingLocked 真正的开始执行把服务启动的一系列工作。

# bindService过程出现Start proc Time out，会回调onServiceDisconnected吗？

## Ams.processStartTimedOutLocked
  - ActiveServices.processStartTimedOutLocked
  -
      ```
      ActiveServices.bringDownServiceLocked
      for (int conni=r.connections.size()-1; conni>=0; conni--) {
          ArrayList<ConnectionRecord> c = r.connections.valueAt(conni);
          for (int i=0; i<c.size(); i++) {
              ConnectionRecord cr = c.get(i);
              // There is still a connection to the service that is
              // being brought down.  Mark it as dead.
              cr.serviceDead = true;
              try {
                  cr.conn.connected(r.name, null, true);
              } catch (Exception e) {
                  Slog.w(TAG, "Failure disconnecting service " + r.name +
                        " to connection " + c.get(i).conn.asBinder() +
                        " (in " + c.get(i).binding.client.processName + ")", e);
              }
          }
      ```
  - LoadedApk.connected
    - doConnected(name, service, dead);
    -  
        ```
         public void doConnected(ComponentName name, IBinder service, boolean dead) {
            //old 为null，因为进程没起来，下面if(service!=null)没进去过
            //mActiveConnections 从未push过这个service
            old = mActiveConnections.get(name);
            if (service != null) {
                    // A new service is being connected... set it all up.
                    info = new ConnectionInfo();
                    info.binder = service;
                    info.deathMonitor = new DeathMonitor(name, service);
                    try {
                        service.linkToDeath(info.deathMonitor, 0);
                        mActiveConnections.put(name, info);
                    } catch (RemoteException e) {
                        // This service was dead before we got it...  just
                        // don't do anything with it.
                        mActiveConnections.remove(name);
                        return;
                    }
              }
            ...
            //所以下面只会执行onBindingDied
            if (old != null) {
                mConnection.onServiceDisconnected(name);
            }
            //传进来是true
            if (dead) {
                mConnection.onBindingDied(name);
            }

          }
        ```


# 修复此bug方案
## 方案一
  - 在onBindingDied 里clearWallpaper
## 方案二
  - 在调用方app里调用clearWallpaer
