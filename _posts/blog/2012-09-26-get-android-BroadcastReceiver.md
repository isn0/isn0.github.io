---
layout:     post
title:      三行代码获取特定广播的所有接收者
category: blog
description: 12年研究安卓时写的老文章
---
**12年写的文章，原文发表于[看雪论坛][]**

&nbsp;&nbsp;Android中收到短信等事件都是通过广播发送给应用程序的，360手机卫士等程序都是通过注册高优先级的BroadcastReceiver来实现短信防火墙等功能。对于我们来说很想知道系统中都有哪些程序注册了BroadcastReceiver，但是通过什么方法能获取系统BroadcastReceiver的列表呢？

&nbsp;&nbsp;我在群里问了一下，他们告诉我的答案居然是分析所有apk的AndroidManifest.xml，再加上解析dex里面的动态注册！而这根本不是我想达到的目的，我要知道的是目前系统中实时的BroadcastReceiver，而不是通过静态分析得到。于是我开始看Android Framework代码，想搞清楚广播的实现机制到底是怎样的。

&nbsp;&nbsp;注册BroadcastReceiver的过程是这样的：Activity调用registerReceiver，然后经过几层内部类接口的调用之后，通过Binder机制与ActivityManagerService通信，而ActivityManagerService里有一个ReceiverList保存着系统所有的BroadcastReceiver。

&nbsp;&nbsp;发送广播的过程是：Activity向ActivityManagerService发送广播，ActivityManagerService查找ReceiverList，通过比对IntentFilter找到所有对应的BroadcastReceiver，根据BroadcastReceiver的优先级进行排序后，扔进广播发送队列里。而后由专门的线程负责投递广播消息。

![广播][1]

&nbsp;&nbsp;大致的过程就是这样的，那么目标就是获取ActivityManagerService中的ReceiverList。一开始我想的办法是：因为ActivityManagerService是一个单独的进程（system_server），我们可以通过注入system_server进程来获得ReceiverList。但是由于ActivityManagerService是java实现的，无法直接获得ReceiverList，要解析java的数据结构难度就太大了。

&nbsp;&nbsp;于是又再次查看
> frameworks/base/services/java/com/android/server/am/ActivityManagerService.java：

        public final class ActivityManagerService extends ActivityManagerNative
        implements Watchdog.Monitor, BatteryStatsImpl.BatteryCallback {
      ......
    
      private final int broadcastIntentLocked(ProcessRecord callerApp,
          String callerPackage, Intent intent, String resolvedType,
          IIntentReceiver resultTo, int resultCode, String resultData,
          Bundle map, String requiredPermission,
          boolean ordered, boolean sticky, int callingPid, int callingUid) {
        intent = new Intent(intent);
    
        ......
    
        // Figure out who all will receive this broadcast.
        List receivers = null;
        List<BroadcastFilter> registeredReceivers = null;
        try {
          if (intent.getComponent() != null) {
            ......
          } else {
            ......
            registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false);
          }
        } catch (RemoteException ex) {
          ......
        }
    
        final boolean replacePending =
          (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;
    
        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        if (!ordered && NR > 0) {
          // If we are not serializing this broadcast, then send the
          // registered receivers separately so they don't wait for the
          // components to be launched.
          BroadcastRecord r = new BroadcastRecord(intent, callerApp,
            callerPackage, callingPid, callingUid, requiredPermission,
            registeredReceivers, resultTo, resultCode, resultData, map,
            ordered, sticky, false);
          ......
          boolean replaced = false;
          if (replacePending) {
            for (int i=mParallelBroadcasts.size()-1; i>=0; i--) {
              if (intent.filterEquals(mParallelBroadcasts.get(i).intent)) {
                ......
                mParallelBroadcasts.set(i, r);
                replaced = true;
                break;
              }
            }
          }
    
          if (!replaced) {
            mParallelBroadcasts.add(r);
    
            scheduleBroadcastsLocked();
          }
    
          registeredReceivers = null;
          NR = 0;
        }
    
        ......
    
      }
    
      ......
    }

看这一句：

    registeredReceivers = mReceiverResolver.queryIntent(intent, resolvedType, false); 

> frameworks\base\services\java\com\android\server\IntentReslover.java

中，注意到一个特性：

    public List<R> queryIntent(Intent intent, String resolvedType, boolean defaultOnly) {
            String scheme = intent.getScheme();
    
            ArrayList<R> finalList = new ArrayList<R>();
    
            final boolean debug = localLOGV ||
                    ((intent.getFlags() & Intent.FLAG_DEBUG_LOG_RESOLUTION) != 0);
    
            if (debug) Slog.v(
                TAG, "Resolving type " + resolvedType + " scheme " + scheme
                + " of intent " + intent);
           sortResults(finalList);
    ............
            if (debug) {
                Slog.v(TAG, "Final result list:");
                for (R r : finalList) {
                    Slog.v(TAG, "  " + r);
                }
            }
            return finalList;
        }
&nbsp;&nbsp;这个有个debug选项可以打印出最终获得的ReceiverList，还是按照优先级排过序的有木有？

&nbsp;&nbsp;于是乎获取接收器列表就太简单了，只需要三行代码即可：

    Intent intent = new Intent("android.provider.Telephony.SMS_RECEIVED");
    intent.addFlags(Intent.FLAG_DEBUG_LOG_RESOLUTION);
    sendBroadcast(intent);
&nbsp;&nbsp;重点就是第二句，设置了FLAG_DEBUG_LOG_RESOLUTION，这样就会在LogCat中打印出所有注册了android.provider.Telephony.SMS_RECEIVED的BroadcastReceiver。用IntentResolver作为TAG过滤一下看起来更方便：
![效果][2]
[看雪论坛]:    http://bbs.pediy.com/showthread.php?t=156436  "看雪论坛"


  [1]: images/android/8025841287_bb57504a4c.jpg
  [2]: images/android/8025842496_c55e86107f_b.jpg