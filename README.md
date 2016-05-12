<h1>BroadcastReceiver源码解析</h1>
<h4>本文分析版本:Android API 23</h4>
<h3 >1，简介</h3>
BroadcastReceiver，中文直译为“广播接收者”，在Android 系统中，广播主要用在组件与组件之间进行消息传递。组件与组件之间可以是同一个进程，也可以是不同进程。既然是可以跨进程的，那么可以想像底层应该是基于Binder来实现的，事实也正是如此。

<h3 >2，为什么要有广播</h3>
既然BroadcastReceiver是基于Binder的，那么用纯Binder进行通信就行了，为什么还要创造出BroadcastReceiver？
我们知道AIDL也是基于Binder的，AIDL中采取的是proxy-stub模式分别在Client端对transact（），在Server端对onTransact（）进行封装。Client端拿到Server端的远程代理对象，才调用到Server端的方法，这需要在程序中事先就写好Server端，以便提供远程代理对象mRemote。
如果程序事先不提供远程代理对象mRemote，而是由Client动态注册呢？这时候广播就登场了，在广播机制中，广播发送者不需要知道广播接收者的存在，这样就降低了发送者和接收者之间的耦合度，提高了程序的扩展性和维护性。

 ![广播发送路径](http://img.blog.csdn.net/20160510111949027)
 
<h3>3，两种Receiver的注册</h3>
上图可以看出，在Android广播机制中，ActivityManagerService（以下简称AMS）扮演着路由器的角色，一根光纤进来，分出不同的线路发送给不同的接收者。因此广播的注册就是把广播接收者注册到AMS的过程（相当于，把电脑网线插进路由器，于是能接收到光纤的网络）。

Android系统中的Receiver注册分为动态和静态。动态receiver必须在程序运行期动态注册，其实际的注册动作由ContextImpl对象完成。静态receiver则是在AndroidManifest.xml中声明的。相比动态注册，静态稍显麻烦，因为在广播传递之时，静态receiver所从属的进程可能还没有启动，这时就需要先启动该进程，然后再接收AMS发送来的广播消息。

<h4>3.1静态注册实例 </h4>
首先编写一个接收器用来接收AMS发送来的广播
```
public class MyReceiver extends BroadcastReceiver {
	private static final String TAG = "MyReceiver";  
	@Override
	public void onReceive(Context context, Intent intent) {
		// TODO Auto-generated method stub 
                	doSomething();
	}
}
```
然后在AndroidManifest.xml中配置广播接收
```
<receiver android:name=".MyReceiver">
	<intent-filter>
		<action android:name="android.intent.action.MY_BROADCAST"/>
		<category android:name="android.intent.category.DEFAULT" />
	</intent-filter>
</receiver>
```
<h4>3.2动态注册实例 </h4>
动态注册直接在程序中声明一个IntentFilter和一个MyReceiver对象，并调用registerReceiver(receiver, filter)，讲两个参数传递进去完成注册。
```
MyReceiver receiver = new MyReceiver();
IntentFilter filter = new IntentFilter();
filter.addAction("android.intent.action.MY_BROADCAST");
registerReceiver(receiver, filter);

```

<h3>4，BroadcastReceiver之源码分析 </h3>
<h4>4.1，动态注册过程源码分析</h4>
在Activity中动态注册广播时，在注册方法之前其实省略了Context，也就是实际上调用的是Context. registerReceiver()。Context是一个抽象类，它是Client端和AMS,WMS等系统服务进行通信的接口，Activity、Service和Application都是继承它的子类。Context的实现类是ContextImpl，也就是说最终调用到的是ContextImpl中的registerReceiver（）。

以下代码位于类ContextImpl中：
```
@Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter) {	    
	    //1接着回调用四个参数的registerReceiver（）
        return registerReceiver(receiver, filter, null, null);
    }
    @Override
    public Intent registerReceiver(BroadcastReceiver receiver, IntentFilter filter,
            String broadcastPermission, Handler scheduler) {
            
	    //2接着调用registerReceiverInternal（）
        return registerReceiverInternal(receiver, getUserId(),
                filter, broadcastPermission, scheduler, getOuterContext());
    }

 private Intent registerReceiverInternal(BroadcastReceiver receiver, int userId,
            IntentFilter filter, String broadcastPermission,
            Handler scheduler, Context context) {
        IIntentReceiver rd = null;
        if (receiver != null) {
            if (mPackageInfo != null && context != null) {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
       //3调用PackageInfo的getReceiverDispatcher（）将广播接收者及其他参数封装成一个对象
                rd = mPackageInfo.getReceiverDispatcher(
                    receiver, context, scheduler,
                    mMainThread.getInstrumentation(), true);
            } else {
                if (scheduler == null) {
                    scheduler = mMainThread.getHandler();
                }
                rd = new LoadedApk.ReceiverDispatcher(
                        receiver, context, scheduler, null, true).getIIntentReceiver();
            }
        }
        try {
        //4ActivityManagerNative.getDefault()返回的其实是AMS在Client的代理对象
        return ActivityManagerNative.getDefault().registerReceiver(
                    mMainThread.getApplicationThread(), mBasePackageName,
                    rd, filter, broadcastPermission, userId);
        } catch (RemoteException e) {
            return null;
        }
    }  
```
注释1,2略。
注释3：rd=mPackageInfo.getReceiverDispatcher（），mPackageInfo（LoadedApk类声明）调用其getReceiverDispatcher（）最终返回的是IIntentReceiver.Stub类型的对象InnerReceiver ，该类实现了IIntentReceiver接口并继承Binder（显然就是AIDL中的stub部分）。在Android系统中，广播最终都是由AMS转送出来的。AMS利用Binder机制，而此处的rd对象正是承担传递工作的Binder实体。

为了方便管理这个binder实体，Android中定义了一个ReceiveDispatcher的类,每一个动态注册都会新建一个ReceiveDispatcher对象，并且这些对象保存在一个HashMap中，而这个HashMap又保存在LoadedApk中。

以下代码位于类LoadedApk 中

```
public final class LoadedApk {
	......
      static final class ReceiverDispatcher {

        final static class InnerReceiver extends IIntentReceiver.Stub {
            final WeakReference<LoadedApk.ReceiverDispatcher> mDispatcher;
            final LoadedApk.ReceiverDispatcher mStrongRef;

            InnerReceiver(LoadedApk.ReceiverDispatcher rd, boolean strong) {
                mDispatcher = new WeakReference<LoadedApk.ReceiverDispatcher>(rd);
                mStrongRef = strong ? rd : null;
            }         
           ......
```
在Android系统中，系统每加载一个apk，就会有一个LoadedApk对象。而每个LoadedApk对象里会有一张HashMap，用来记录每个apk里面动态注册了那些广播接收者。

注释4：ActivityManagerNative.getDefault().registerReceiver（）
作用将上诉的到的rd等等数据发送给AMS。ActivityManagerNative.getDefault()得到的是ActivityManagerProxy，紧接着会调用到ActivityManagerProxy中的registerReceiver（）；

以下代码位于类ActivityManagerProxy中：
```
 public Intent registerReceiver(IApplicationThread caller, String packageName, IIntentReceiver receiver, IntentFilter filter, String perm, int userId) throws RemoteException
    {
    
    //5封装数据到Parcel 对象，并准备发送到AMS
    
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        data.writeString(packageName);
        data.writeStrongBinder(receiver != null ? receiver.asBinder() : null);
        filter.writeToParcel(data, 0);
        data.writeString(perm);
        data.writeInt(userId);
        
        //6，发射消息到AMS（此时开始切换线程从ClientThread到BinderThread）
        mRemote.transact(REGISTER_RECEIVER_TRANSACTION, data, reply, 0);
        reply.readException();
        Intent intent = null;
        int haveIntent = reply.readInt();
        if (haveIntent != 0) {
            intent = Intent.CREATOR.createFromParcel(reply);
        }
        reply.recycle();
        data.recycle();
        return intent;
    }
```
注释5：封装数据到Parcel对象data中，准备被Binder的代理对象mRemote发射到AMS
注释6：发送一个REGISTER_RECEIVER_TRANSACTION的进程间通信请求，此时开始切换线程。

接下来的过程就进入到AMS中了。首先流程进入到AMS中的registerReceiver（），（PS：其实是先到ActivityManagerNative，在ActivityManagerNative中对数据进行解包，然后调用到AMS中的registerReceiver（））。
以下代码位于ActivityManagerService 中：
```
public final class ActivityManagerService extends ActivityManagerNative{
......
  public Intent registerReceiver(IApplicationThread caller, String callerPackage,
            IIntentReceiver receiver, IntentFilter filter, String permission, int userId) {
            ......
            //7得到请求注册的应用所在进程
              synchronized(this) {
            if (caller != null) {
                callerApp = getRecordForAppLocked(caller);
            }
            ......
            synchronized (this) {
            if (callerApp != null && (callerApp.thread == null
                    || callerApp.thread.asBinder() != caller.asBinder())) 
                return null;
            }
            //8每次发来的一个注册请求都会对应ReceiverList 中的一条数据。
            ReceiverList rl = mRegisteredReceivers.get(receiver.asBinder());
            if (rl == null) {
                rl = new ReceiverList(this, callerApp, callingPid, callingUid,
                        userId, receiver);
                if (rl.app != null) {
                    rl.app.receivers.add(rl);
                } else {
                    try {
                        receiver.asBinder().linkToDeath(rl, 0);
                    } catch (RemoteException e) {
                        return sticky;
                      ......
           //9创建一个BroadcastFilter 对象，BroadcastFilter 继承自IntentFilter
                       BroadcastFilter bf = new BroadcastFilter(filter, rl, callerPackage,
	                    permission, callingUid, userId);
			            rl.add(bf);
            if (!bf.debugCheck()) {
                Slog.w(TAG, "==> For Dynamic broadcast");
            }
            mReceiverResolver.addFilter(bf);      
          if (allSticky != null) {
                ArrayList receivers = new ArrayList();
                receivers.add(bf);
}
```
注释7：得到是那一个进程请求注册广播接收者。

注释8：每一个BroadCastReceiver被注册的时候会生成一个ReceiverList，每个ReceiverList都对应着Client端的一个ReceiverDispatcher（内含binder），从如果rl为空那么调用  rl = new ReceiverList(this, callerApp, callingPid, callingUid,userId, receiver);。其中参数receiver为IIntentReceiver类型，其对应着ReceiverDispatcher传递过来的binder实体，对应关系在mRegisteredReceivers中，其为一张hashmap。而每个ReceiverList里面保存的都是可以触发该BroadCastReceiver的IntentFilter（实际是BroadcastFilter ，原因BroadcastFilter 继承自IntentFilter）。

注释9： 创建一个BroadCastFilter对象bf，bf对应的是正在注册的广播接收者，紧接着 rl.add(bf)把bf加入到mReceiverResolver和receivers 中。当AMS接收到广播的时候，AMS就会通过其成员变量receivers 找到该广播。

小结：
客户端方面：客户端广播注册的时候，loadedApk对应着多个context对象（一个进程对应多个activity和service以及一个application），每个context里面有一张hashmap存放注册的广播接收者（每个activity里面可以注册多个广播，都存放在hashmap里），每张hashmap里面包含许多ReceiverDispatcher（内含binder）；

AMS方面：每个Client端的ReceiverDispatcher都对应一个ReceiverList（内含binder），该ReceiverList中保存着可以启动该广播接收者的IntentFilter列表。
Client端和AMS端分别的对应图：
![客户端方面](http://img.blog.csdn.net/20160510105454243)
![服务端方面](http://img.blog.csdn.net/20160510105920514)


<h4>4.2静态注册</h4>
 BroadcastReceiver静态注册指的是在AndroidManifest.xml中声明的接收者，在系统启动的时候，会由PackageManagerService（以下简称PMS）去解析。当AMS调用PMS的接口来查询广播注册的时候，PMS会查询记录并且返回给AMS
 以下代码位于PackageManagerService中:
```
 @Override
    public List<ResolveInfo> queryIntentServices(Intent intent, String resolvedType, int flags,
            int userId) {
        if (!sUserManager.exists(userId)) return Collections.emptyList();
        ComponentName comp = intent.getComponent();
        if (comp == null) {
            if (intent.getSelector() != null) {
                intent = intent.getSelector();
                comp = intent.getComponent();
            }
        }
        if (comp != null) {
            final List<ResolveInfo> list = new ArrayList<ResolveInfo>(1);
            final ServiceInfo si = getServiceInfo(comp, flags, userId);
            if (si != null) {
                final ResolveInfo ri = new ResolveInfo();
                ri.serviceInfo = si;
                list.add(ri);
            }
            return list;
        }       
        synchronized (mPackages) {
            String pkgName = intent.getPackage();
            if (pkgName == null) {
                return mServices.queryIntent(intent, resolvedType, flags, userId);
            }
            final PackageParser.Package pkg = mPackages.get(pkgName);
            if (pkg != null) {
                return mServices.queryIntentForPackage(intent, resolvedType, flags, pkg.services,
                        userId);
            }
            return null;
        }
    
```
PMS查询到的List集合返回给AMS，然后AMS依照集合内容执行操作。

<h4>4.3 广播的发送</h4>
Android中发送广播，大致分为：无序广播sendBroadcast，有序广播sendOrderedBroadcast，粘广播sendStickyBroadcast。无序广播发送无序，注册过的接收者都能收到；有序广播对于接收者有一定的优先级，按照这种优先级依次往下发送，再这个过程中上级可以拦截发送到下一级别的广播；粘广播指的是未注册的接收者，一旦日后注册进AMS，那么还能够接收到错过的粘广播。
sendBroadcast（）同样具体实现是在ComtextImpl中。
以下下代码位于ComtextImpl中：
```
 @Override
    public void sendBroadcast(Intent intent) {
        warnIfCallingFromSystemProcess();
        String resolvedType = intent.resolveTypeIfNeeded(getContentResolver());
        try {
            intent.prepareToLeaveProcess();
            //10 拿到AMS的远程代理对象，准备发射到AMS
            ActivityManagerNative.getDefault().broadcastIntent(
                    mMainThread.getApplicationThread(), intent, resolvedType, null,
                    Activity.RESULT_OK, null, null, null, AppOpsManager.OP_NONE, null, false, false,
                    getUserId());
        } catch (RemoteException e) {
            throw new RuntimeException("Failure from system", e);
        }
    }

```
注释10：和注册BroadcastReceiver一样拿到AMS代理对象调用ActivityManagerProxy里面的broadcastIntent。

以下代码运行在ActivityManagerProxy中：

```
 public int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle map,
            String[] requiredPermissions, int appOp, Bundle options, boolean serialized,
            boolean sticky, int userId) throws RemoteException
    {
    
    //11写入数据到data 
        Parcel data = Parcel.obtain();
        Parcel reply = Parcel.obtain();
        data.writeInterfaceToken(IActivityManager.descriptor);
        data.writeStrongBinder(caller != null ? caller.asBinder() : null);
        intent.writeToParcel(data, 0);
        data.writeString(resolvedType);
        data.writeStrongBinder(resultTo != null ? resultTo.asBinder() : null);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeStringArray(requiredPermissions);
        data.writeInt(appOp);
        data.writeBundle(options);
        data.writeInt(serialized ? 1 : 0);
        data.writeInt(sticky ? 1 : 0);
        data.writeInt(userId);        
        //12 发送数据到AMS，（线程从ClientThread切换到BinderThread）
        mRemote.transact(BROADCAST_INTENT_TRANSACTION, data, reply, 0);
        reply.readException();
        int res = reply.readInt();
        reply.recycle();
        data.recycle();
        return res;
    }
```
注释11：和注册广播时候一样，把要发送的数据打包成Parcel对象data。
注释12：发送数据到AMS，此时会切换线程。
接下来会执行到AMS中的broadcastIntent。

以下代码位于ActivityManagerService中：
```
  public final int broadcastIntent(IApplicationThread caller,
            Intent intent, String resolvedType, IIntentReceiver resultTo,
            int resultCode, String resultData, Bundle resultExtras,
            String[] requiredPermissions, int appOp, Bundle options,
            boolean serialized, boolean sticky, int userId) {
        enforceNotIsolatedCaller("broadcastIntent");
        synchronized(this) {
        
        //13检查intent是否合法
            intent = verifyBroadcastLocked(intent);
            
		//14获取广播发送者的身份
            final ProcessRecord callerApp = getRecordForAppLocked(caller);
            final int callingPid = Binder.getCallingPid();
            final int callingUid = Binder.getCallingUid();
            final long origId = Binder.clearCallingIdentity();
            
         //15调用broadcastIntentLocked发送广播
            int res = broadcastIntentLocked(callerApp,
                    callerApp != null ? callerApp.info.packageName : null,
                    intent, resolvedType, resultTo, resultCode, resultData, resultExtras,
                    requiredPermissions, appOp, null, serialized, sticky,
                    callingPid, callingUid, userId);
            Binder.restoreCallingIdentity(origId);
            return res;
        }
    }

```
注释12,13,14,15见代码
接着调用AMS的broadcastIntentLocked（），该方法主要用来查找目标广播接收者。
下面代码位于ActivityManagerService中：

```
   private final int broadcastIntentLocked(ProcessRecord callerApp,
            String callerPackage, Intent intent, String resolvedType,
            IIntentReceiver resultTo, int resultCode, String resultData,
            Bundle resultExtras, String[] requiredPermissions, int appOp, Bundle options,
            boolean ordered, boolean sticky, int callingPid, int callingUid, int userId) {
            
            //16为intent添加FLAG_EXCLUDE_STOPPED_PACKAGES标记；
            intent = new Intent(intent);
        intent.addFlags(Intent.FLAG_EXCLUDE_STOPPED_PACKAGES);
            ......
            
            //17 处理和package相关的广播,处理其他一些系统广播，判断当前是否有权力发出广播。
            ......
            
            //18判断是不是粘广播，并且如果要发送粘广播，那么就更新sticky广播列表。
              if (sticky) {
            if (checkPermission(android.Manifest.permission.BROADCAST_STICKY,
                    callingPid, callingUid)
                    != PackageManager.PERMISSION_GRANTED) {
                String msg = "Permission Denial: broadcastIntent() requesting a sticky broadcast from pid="
                        + callingPid + ", uid=" + callingUid
                        + " requires " + android.Manifest.permission.BROADCAST_STICKY;
                Slog.w(TAG, msg);
                throw new SecurityException(msg);
            }
            if (requiredPermissions != null && requiredPermissions.length > 0) {
                Slog.w(TAG, "Can't broadcast sticky intent " + intent
                        + " and enforce permissions " + Arrays.toString(requiredPermissions));
                return ActivityManager.BROADCAST_STICKY_CANT_HAVE_PERMISSION;
            }
            if (intent.getComponent() != null) {
                throw new SecurityException(
                        "Sticky broadcasts can't target a specific component");
            }
         
            if (userId != UserHandle.USER_ALL) {
              
                ArrayMap<String, ArrayList<Intent>> stickies = mStickyBroadcasts.get(
                        UserHandle.USER_ALL);
                if (stickies != null) {
                    ArrayList<Intent> list = stickies.get(intent.getAction());
                    if (list != null) {
                        int N = list.size();
                        int i;
                        for (i=0; i<N; i++) {
                            if (intent.filterEquals(list.get(i))) {
                                throw new IllegalArgumentException(
                                        "Sticky broadcast " + intent + " for user "
                                        + userId + " conflicts with existing global broadcast");
                            }
                        }
                    }
                }

	//在AMS中查找是否存在一个与参数intent对应的粘性广播	
            ArrayList<Intent> list = stickies.get(intent.getAction());
            if (list == null) {
				//如果不存在那么就创建一个list
                list = new ArrayList<>();
				//保存在stickies中
                stickies.put(intent.getAction(), list);
            }
            final int stickiesCount = list.size();
            int i;
	//for循环检查粘性广播list中是否存在与参数intent一致的广播
            for (i = 0; i < stickiesCount; i++) {
                if (intent.filterEquals(list.get(i))) {
                    
   //如果存在那么就用参数intent描述的广播替换它
                    list.set(i, new Intent(intent));
                    break;
                }
            }
            if (i >= stickiesCount) {
	//如果不存在就把intent描述的广播  加入到list中
                list.add(new Intent(intent));
            }
        }
           ......
 //19判断发送给什么类型广播（静态注册还是动态注册）
            if ((intent.getFlags()&Intent.FLAG_RECEIVER_REGISTERED_ONLY)
                 == 0) {
                 //到PMS找到静态注册的接收者保存在receivers中
            receivers = collectReceiverComponents(intent, resolvedType, callingUid, users);
        }
        if (intent.getComponent() == null) {
            if (userId == UserHandle.USER_ALL && callingUid == Process.SHELL_UID) {
                
                UserManagerService ums = getUserManagerLocked();
                for (int i = 0; i < users.length; i++) {
                    if (ums.hasUserRestriction(
                            UserManager.DISALLOW_DEBUGGING_FEATURES, users[i])) {
                        continue;
                    }
                    List<BroadcastFilter> registeredReceiversForUser =
                            mReceiverResolver.queryIntent(intent,
                                    resolvedType, false, users[i]);
                    if (registeredReceivers == null) {
                        registeredReceivers = registeredReceiversForUser;
                    } else if (registeredReceiversForUser != null) {
                        registeredReceivers.addAll(registeredReceiversForUser);
                    }
                }
            } else {
            //找到动态注册的接受者并保存在registeredReceivers中
                registeredReceivers = mReceiverResolver.queryIntent(intent,
                        resolvedType, false, userId);
            }
        }

            }
            ......
           //20封装广播
        final boolean replacePending =
                (intent.getFlags()&Intent.FLAG_RECEIVER_REPLACE_PENDING) != 0;

        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueing broadcast: " + intent.getAction()
                + " replacePending=" + replacePending);

        int NR = registeredReceivers != null ? registeredReceivers.size() : 0;
        if (!ordered && NR > 0) {
        
            final BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType, requiredPermissions,
                    appOp, brOptions, registeredReceivers, resultTo, resultCode, resultData,
                    resultExtras, ordered, sticky, false, userId);
            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing parallel broadcast " + r);
            final boolean replaced = replacePending && queue.replaceParallelBroadcastLocked(r);
            if (!replaced) {
                queue.enqueueParallelBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
            registeredReceivers = null;
            NR = 0;
            ......
            ......
            //21整理两个receiver列表         
            int NT = receivers != null ? receivers.size() : 0;
            int it = 0;
            ResolveInfo curt = null;
            BroadcastFilter curr = null;
            while (it < NT && ir < NR) {
                if (curt == null) {
                    curt = (ResolveInfo)receivers.get(it);
                }
                if (curr == null) {
                    curr = registeredReceivers.get(ir);
                }
                if (curr.getPriority() >= curt.priority) {
                    // Insert this broadcast record into the final list.
                    receivers.add(it, curr);
                    ir++;
                    curr = null;
                    it++;
                    NT++;
                } else {
                    // Skip to the next ResolveInfo in the final list.
                    it++;
                    curt = null;
                }
            }
        }
        while (ir < NR) {
            if (receivers == null) {
                receivers = new ArrayList();
            }
            receivers.add(registeredReceivers.get(ir));
            ir++;
        }
            ......
            //22向接收者发送广播
             if ((receivers != null && receivers.size() > 0)
                || resultTo != null) {
            BroadcastQueue queue = broadcastQueueForIntent(intent);
            BroadcastRecord r = new BroadcastRecord(queue, intent, callerApp,
                    callerPackage, callingPid, callingUid, resolvedType,
                    requiredPermissions, appOp, brOptions, receivers, resultTo, resultCode,
                    resultData, resultExtras, ordered, sticky, false, userId);

            if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Enqueueing ordered broadcast " + r
                    + ": prev had " + queue.mOrderedBroadcasts.size());
            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                    "Enqueueing broadcast " + r.intent.getAction());

            boolean replaced = replacePending && queue.replaceOrderedBroadcastLocked(r);
            if (!replaced) {
                queue.enqueueOrderedBroadcastLocked(r);
                queue.scheduleBroadcastsLocked();
            }
        }

        return ActivityManager.BROADCAST_SUCCESS;
    }
        
```
注释16：给intent添加FLAG_EXCLUDE_STOPPED_PACKAGES标记，如果一个应用在安装后从来没有启动过，或者已经被用户强制停止了，那么这个应用就处于停止状态，增加FLAG表示是否要激活处于停止状态的应用，并发送广播给他们。

注释17：代码量巨长，没有贴出来。该区间内的代码主要是处理package的广播，例如删除包的时候发送PACKAGE_REMOVED广播；其他系统广播发送，例如PROXY_CHANGE_ACTION广播；还有判断当前应用是否有权利发送广播。

注释18：sticky表示Intent描述的广播是不是一个粘广播，所有相同的粘广播都保存在一个相同的list中。其中的for循环，判断当前list中是否存在一个和参数intent一样的广播，如果存在那么就用intent描述的广播来替换它，如果不存在就把该intent描述的广播添加到粘广播list中。

注释19：把目标广播接收者分为两种类型，静态和动态，系统会到PMS中找到静态注册的广播接收者，并保存在receiver中。receivers是利用PMS的queryIntentReceivers（），查询出和intent匹配的所有静态注册的接收者，此时所返回的查询结果本身已经排好序了。registeredReceivers中的子项中保存的是动态注册的接收者，是没有经过排序的。如果要并行发送广播那么registeredReceivers中的各个子项会调用queue.scheduleBroadcastsLocked()并行处理。如果要串行发送广播，那么必须把registeredReceivers表合并到receivers表中。

注释20：BroadcastRecord r = new BroadcastRecord（）。将一个intent描述的广播，以及动态注册的广播接收者封装成一个BroadcastRecord 对象r，r描述的是AMS用来发送广播的一个任务。这个任务不是立马执行的。而是被加入到一个队列中，等待被执行。(发送广播要考虑广播是否是有序的，广播接收者本身也分静态注册和动态注册，静态广播接收者一般是有序的，而动态注册的广播接收者，则分情况)，这里主要将并行的广播封装并插入BroadcastQueue内的并行处理队列。

注释21：此区间内的while循环主要用来合并动态注册和静态注册的广播接收者，合并后的广播接收者都保存在列表receivers中，并且是按照优先级排序的（剔除了并行广播后，合并之后的静态注册接收者和动态注册的接收者就按照顺序排放在一张表中）。

注释22：主要将参数intent所描述的广播，以及剩余的其他广播封装成一个BroadcastRecord对象r，r就是AMS要执行的一个广播转发任务并且加入到AMS的内部有序广播队列中该段代码，主要new一个BroadcastRecord节点，并插入BroadcastQueue内的串行处理队列，最后调用实际的广播发行方法scheduleBroadcastsLocked()。
接下来就执行scheduleBroadcastsLocked（）方法。
以下代码位于BroadcastQueue中：

```
//23发送消息
 public void scheduleBroadcastsLocked() {
        if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Schedule broadcasts ["
                + mQueueName + "]: current="
                + mBroadcastsScheduled);
	//为true说名ams的mq中已经有BROADCAST_INTENT_MSG的消息
        if (mBroadcastsScheduled) {
            return;
        }
        //否则就发送消息
        mHandler.sendMessage(mHandler.obtainMessage(BROADCAST_INTENT_MSG, this));
        mBroadcastsScheduled = true;
    }
```
注释23：mBroadcastsScheduled是AMS的一个成员变量，用来记录AMS是否已经向其所在线程的消息队列发过一条BROADCAST_INTENT_MSG的消息。如果为true说明AMS所在消息队列已经有一个BROADCAST_INTENT_MSG的消息，反之则调用mHandler发送该条消息到AMS所在线程消息队列。最后将mBroadcastsScheduled 设置为true，表明该广播已经成功发送出去了（ps：这个阶段广播只是被发送到了AMS所在线程的消息队列，并没有真正到广播接收者手中，也就是表明当广播被发送到AMS的时候系统会认为广播已经发送成功）。
既然是handler发送的消息，那么处理消息的肯定在handleMessage中处理的。
以下代码位于BroadcastQueue中：

```
    @Override
        public void handleMessage(Message msg) {
            switch (msg.what) {
            //24 处理消息
                case BROADCAST_INTENT_MSG: {
                    if (DEBUG_BROADCAST) Slog.v(
                            TAG_BROADCAST, "Received BROADCAST_INTENT_MSG");
                    processNextBroadcast(true);
                } break;
                case BROADCAST_TIMEOUT_MSG: {
                    synchronized (mService) {
                        broadcastTimeoutLocked(true);
                    }
                } break;
                case SCHEDULE_TEMP_WHITELIST_MSG: {
                    DeviceIdleController.LocalService dic = mService.mLocalDeviceIdleController;
                    if (dic != null) {
                        dic.addPowerSaveTempWhitelistAppDirect(UserHandle.getAppId(msg.arg1),
                                msg.arg2, true, (String)msg.obj);
                    }
                } break;
            }
        }
```
注释24：调用processNextBroadcast来处理类型为BROADCAST_INTENT_MSG的消息

以下代码位于BroadcastQueue中：

```
final void processNextBroadcast(boolean fromMsg) {
......
 if (fromMsg) {
 //fromMsg表示是否为BROADCAST_INTENT_MSG类型的消息
                mBroadcastsScheduled = false;
            }

           //25处理mParallelBroadcasts中的广播转发任务
            while (mParallelBroadcasts.size() > 0) {
                r = mParallelBroadcasts.remove(0);
                r.dispatchTime = SystemClock.uptimeMillis();
                r.dispatchClockTime = System.currentTimeMillis();
                final int N = r.receivers.size();
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST, "Processing parallel broadcast ["
                        + mQueueName + "] " + r);
                for (int i=0; i<N; i++) {
                    Object target = r.receivers.get(i);
                    if (DEBUG_BROADCAST)  Slog.v(TAG_BROADCAST,
                            "Delivering non-ordered on [" + mQueueName + "] to registered "
                            + target + ": " + r);
                     //将无序广播调度队列mParallelBroadcasts中的广播发送给它的目标广播接收者
                    deliverToRegisteredReceiverLocked(r, (BroadcastFilter)target, false);
                }
......
}
```
注释25：while循环处理保存在mParallelBroadcasts中的广播转发任务，mParallelBroadcasts是无序广播队列。

```
//26处理AMS有序调度队列morderedbroadcast

  if (mPendingBroadcast != null) {
                if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                        "processNextBroadcast [" + mQueueName + "]: waiting for "
                        + mPendingBroadcast.curApp);

                boolean isDead;
                synchronized (mService.mPidsSelfLocked) {
                    ProcessRecord proc = mService.mPidsSelfLocked.get(mPendingBroadcast.curApp.pid);
                    isDead = proc == null || proc.crashing;
                }
              //如果正在启动 ams就wait	

                if (!isDead) {
                 
                    return;
                } else {
                	// 否则就准备想它发送广播
                    Slog.w(TAG, "pending app  ["
                            + mQueueName + "]" + mPendingBroadcast.curApp
                            + " died before responding to broadcast");
                    mPendingBroadcast.state = BroadcastRecord.IDLE;
                    mPendingBroadcast.nextReceiver = mPendingBroadcastRecvIndex;
                    mPendingBroadcast = null;
                }
            }
            //27
            
           do {
                if (mOrderedBroadcasts.size() == 0) {
                
			//广播已经处理完了
                   
                    mService.scheduleAppGcsLocked();
                    if (looped) {
                       
                        mService.updateOomAdjLocked();
                    }
                    return;
                }
	//广播未处理完取出来存放在r中
                r = mOrderedBroadcasts.get(0);
                boolean forceReceive = false;

                int numReceivers = (r.receivers != null) ? r.receivers.size() : 0;
                if (mService.mProcessesReady && r.dispatchTime > 0) {
                    long now = SystemClock.uptimeMillis();
                    if ((numReceivers > 0) &&
                            (now > r.dispatchTime + (2*mTimeoutPeriod*numReceivers))) {
                        Slog.w(TAG, "Hung broadcast ["
                                + mQueueName + "] discarded after timeout failure:"
                                + " now=" + now
                                + " dispatchTime=" + r.dispatchTime
                                + " startTime=" + r.receiverTime
                                + " intent=" + r.intent
                                + " numReceivers=" + numReceivers
                                + " nextReceiver=" + r.nextReceiver
                                + " state=" + r.state);
                        broadcastTimeoutLocked(false); // forcibly finish this broadcast
                        forceReceive = true;
                        r.state = BroadcastRecord.IDLE;
                    }
                }

                if (r.state != BroadcastRecord.IDLE) {
                    if (DEBUG_BROADCAST) Slog.d(TAG_BROADCAST,
                            "processNextBroadcast("
                            + mQueueName + ") called when not idle (state="
                            + r.state + ")");
                    return;
                }

                if (r.receivers == null || r.nextReceiver >= numReceivers
                        || r.resultAbort || forceReceive) {
                
                    if (r.resultTo != null) {
                        try {
                            if (DEBUG_BROADCAST) Slog.i(TAG_BROADCAST,
                                    "Finishing broadcast [" + mQueueName + "] "
                                    + r.intent.getAction() + " app=" + r.callerApp);
                            performReceiveLocked(r.callerApp, r.resultTo,
                                new Intent(r.intent), r.resultCode,
                                r.resultData, r.resultExtras, false, false, r.userId);
                      mBroadcastHistory.
                            r.resultTo = null;
                        } catch (RemoteException e) {
                            r.resultTo = null;
                            Slog.w(TAG, "Failure ["
                                    + mQueueName + "] sending broadcast result of "
                                    + r.intent, e);
                        }
                    }

                    if (DEBUG_BROADCAST) Slog.v(TAG_BROADCAST, "Cancelling BROADCAST_TIMEOUT_MSG");
                    cancelBroadcastTimeoutLocked();

                    if (DEBUG_BROADCAST_LIGHT) Slog.v(TAG_BROADCAST,
                            "Finished with ordered broadcast " + r);

                    
                    addBroadcastToHistoryLocked(r);
                    mOrderedBroadcasts.remove(0);
                    r = null;
                    looped = true;
                    continue;
                }
            } while (r == null);
               ......
```
注释26：有序广播中，目标广播有可能是静态的。要是静态注册的广播进程还未启动起来。那么AMS就将其进程启动起来，然后发送广播给广播接收者。

注释27：在有序调度队列mOrderedBroadcasts中找到下一个需要处理的广播转发任务。while循环当r！=null的时候跳出循环，这时候下一个需要处理的广播就保存在r中，这时候AMS就对其进行处理 ，同样调用到方法deliverToRegisteredReceiverLocked(r, filter, r.ordered);
接着程序运行到deliverToRegisteredReceiverLocked（）。
以下代码位于BroadcastQueue中：
```
......
 private void deliverToRegisteredReceiverLocked(BroadcastRecord r,
            BroadcastFilter filter, boolean ordered) {
    try {
                if (DEBUG_BROADCAST_LIGHT) Slog.i(TAG_BROADCAST,
                        "Delivering to " + filter + " : " + r);
		//28 将broadcast 的对象r 描述的广播转发给broadcastFilter处理
                performReceiveLocked(filter.receiverList.app, filter.receiverList.receiver,
                        new Intent(r.intent), r.resultCode, r.resultData,
                        r.resultExtras, r.ordered, r.initialSticky, r.userId);
                if (ordered) {
                    r.state = BroadcastRecord.CALL_DONE_RECEIVE;
                }
            } catch (RemoteException e) {
                Slog.w(TAG, "Failure sending broadcast " + r.intent, e);
                if (ordered) {
                    r.receiver = null;
                    r.curFilter = null;
                    filter.receiverList.curBroadcast = null;
                    if (filter.receiverList.app != null) {
                        filter.receiverList.app.curReceiver = null;
                    }
                }
            }
......
```
注释28：deliverToRegisteredReceiverLocked方法中系统在检查广播发送者和接收者的权限之后，最后将BroadcastRecord对象r描述的广播交给BroadcastFilter处理。接着调用performReceiveLocked（）方法。

以下代码位于BroadcastQueue中：

```
   private static void performReceiveLocked(ProcessRecord app, IIntentReceiver receiver,
            Intent intent, int resultCode, String data, Bundle extras,
            boolean ordered, boolean sticky, int sendingUser) throws RemoteException {
       
        if (app != null) {
            if (app.thread != null) {
              //29调用到ApplicationThreadProxy中的scheduleRegisteredReceiver方法
                app.thread.scheduleRegisteredReceiver(receiver, intent, resultCode,
                        data, extras, ordered, sticky, sendingUser, app.repProcState);
            } else {
               
                throw new RemoteException("app.thread must not be null");
            }
        } else {
       
            receiver.performReceive(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
        }
    
```
注释29：app.thread对应的就是ApplicationThreadProxy，该类是ApplicationThreadNative的内部类。
以下代码位于ApplicationThreadProxy中：

```
  public final void scheduleReceiver(Intent intent, ActivityInfo info,
            CompatibilityInfo compatInfo, int resultCode, String resultData,
            Bundle map, boolean sync, int sendingUser, int processState) throws RemoteException {
        Parcel data = Parcel.obtain();
        data.writeInterfaceToken(IApplicationThread.descriptor);
        intent.writeToParcel(data, 0);
        info.writeToParcel(data, 0);
        compatInfo.writeToParcel(data, 0);
        data.writeInt(resultCode);
        data.writeString(resultData);
        data.writeBundle(map);
        data.writeInt(sync ? 1 : 0);
        data.writeInt(sendingUser);
        data.writeInt(processState);
        //30远程跨进程发送数据
        mRemote.transact(SCHEDULE_RECEIVER_TRANSACTION, data, null,
                IBinder.FLAG_ONEWAY);
        data.recycle();
    }
```
注释30：远程Binder对象发送Parcel  类型数据到ApplicationThread。接着调用到ApplicationThread中的scheduleRegisteredReceiver（）。ApplicationThread是ActivityThread的内部类。
以下代码位于ApplicationThread中：

```
public void scheduleRegisteredReceiver(IIntentReceiver receiver, Intent intent,
                int resultCode, String dataStr, Bundle extras, boolean ordered,
                boolean sticky, int sendingUser, int processState) throws RemoteException {
            updateProcessState(processState, false);
	//receiver指向一个InnerReceiver 对象，每个InnerReceiver对象封装了一个广播接收者
	
            receiver.performReceive(intent, resultCode, dataStr, extras, ordered,
                    sticky, sendingUser);
        }
```
接着调用InnerReceiver 中的performReceive（）方法。
以下代码位于InnerReceiver类中，InnerReceiver又位于类loadedApk中：

```
 public void performReceive(Intent intent, int resultCode, String data,
                    Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
                LoadedApk.ReceiverDispatcher rd = mDispatcher.get();
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = intent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Receiving broadcast " + intent.getAction() + " seq=" + seq
                            + " to " + (rd != null ? rd.mReceiver : null));
                }
                if (rd != null) {
                //31 调用rd的performReceive
                    rd.performReceive(intent, resultCode, data, extras,
                            ordered, sticky, sendingUser);
                } else {
                   
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing broadcast to unregistered receiver");
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    try {
                        if (extras != null) {
                            extras.setAllowFds(false);
                        }
                        mgr.finishReceiver(this, resultCode, data, extras, false, intent.getFlags());
                    } catch (RemoteException e) {
                        Slog.w(ActivityThread.TAG, "Couldn't finish broadcast to unregistered receiver");
                    }
                }
            }
        }
```
注释31：接着调用到rd的performReceive（）。rd是ReceiverDispatcher的对象。还记得ReceiverDispatcher么？在注册广播的时候每个Client端的ReceiverDispatcher都对应一个ReceiverList，接着调用到ReceiverDispatcher的performReceive（），ReceiverDispatcher是loadedApk的内部类。
以下代码位于ReceiverDispatcher中：

```
 public void performReceive(Intent intent, int resultCode, String data,
                Bundle extras, boolean ordered, boolean sticky, int sendingUser) {
            if (ActivityThread.DEBUG_BROADCAST) {
                int seq = intent.getIntExtra("seq", -1);
                Slog.i(ActivityThread.TAG, "Enqueueing broadcast " + intent.getAction() + " seq=" + seq
                        + " to " + mReceiver);
            }
            Args args = new Args(intent, resultCode, data, extras, ordered,
                    sticky, sendingUser);
                    
	//32 mActivityThread 是一个handler 对象指向ActivityThread中的handler对象
            if (!mActivityThread.post(args)) {
                if (mRegistered && ordered) {
                    IActivityManager mgr = ActivityManagerNative.getDefault();
                    if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                            "Finishing sync broadcast to " + mReceiver);
                    args.sendFinished(mgr);
                }
            }
        }
```
注释32：调用mActivityThread.post(args)方法，mActivityThread是一个Handler对象，该对象指向ActivityThread中的mH；因此此刻切换到广播接收者所在进程的主线程来执行args，args是一个Runnable对象。
以下代码位于Args 中：

```
  final class Args extends BroadcastReceiver.PendingResult implements Runnable {
  ......
       public void run() {
                final BroadcastReceiver receiver = mReceiver;
                final boolean ordered = mOrdered;
                
                if (ActivityThread.DEBUG_BROADCAST) {
                    int seq = mCurIntent.getIntExtra("seq", -1);
                    Slog.i(ActivityThread.TAG, "Dispatching broadcast " + mCurIntent.getAction()
                            + " seq=" + seq + " to " + mReceiver);
                    Slog.i(ActivityThread.TAG, "  mRegistered=" + mRegistered
                            + " mOrderedHint=" + ordered);
                }
                
                final IActivityManager mgr = ActivityManagerNative.getDefault();
                final Intent intent = mCurIntent;
                mCurIntent = null;
                
                if (receiver == null || mForgotten) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing null broadcast to " + mReceiver);
                        sendFinished(mgr);
                    }
                    return;
                }

                Trace.traceBegin(Trace.TRACE_TAG_ACTIVITY_MANAGER, "broadcastReceiveReg");
                try {
                    ClassLoader cl =  mReceiver.getClass().getClassLoader();
                    intent.setExtrasClassLoader(cl);
                    setExtrasClassLoader(cl);
                    receiver.setPendingResult(this);
		//33 receiver指向的是我们自己继承的broadcastreceiver因此调用到我们写的onReceive方法
                    receiver.onReceive(mContext, intent);
                } catch (Exception e) {
                    if (mRegistered && ordered) {
                        if (ActivityThread.DEBUG_BROADCAST) Slog.i(ActivityThread.TAG,
                                "Finishing failed broadcast to " + mReceiver);
		//34通知ams 转发出来的广播已经处理完成
                        sendFinished(mgr);
                    }
                    if (mInstrumentation == null ||
                            !mInstrumentation.onException(mReceiver, e)) {
                        Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
                        throw new RuntimeException(
                            "Error receiving broadcast " + intent
                            + " in " + mReceiver, e);
                    }
                }
                
                if (receiver.getPendingResult() != null) {
                    finish();
                }
                Trace.traceEnd(Trace.TRACE_TAG_ACTIVITY_MANAGER);
            }
  ......
  
  }
```
注释33：此刻的receiver指向的是我们自己定义的BroadcastReceiver的子类例如上文的MyReceiver，此时调用的  receiver.onReceive（）就是我们在MyReceiver中重写的onReceive（）。

注释34：通知AMS，转发出来的广播以执行完毕。

以上就是关于Android的注册和发送广播，到最后执行的到广播接收者onReceive（）方法的所有流程。

总结：BroadcastReceiver其核心机制是基于Bind的IPC通信，一个广播接收者要想接收到广播，必须注册到AMS中，在AMS中有ReceiverList保存注册的广播接收者。发送广播到AMS时候，AMS会查找相应的BroadcastFilter，匹配就会调用相应的远程Binder代理对象通知Client端，最终由Client端的主线程来执行MyReceiver里面的onReceiver（）方法。发送广播的又分为有序和无序，无序一定是动态注册的，而有序可能是动态注册的也可能是静态注册的。静态注册的广播，那怕是没有启动，如果AMS递送相关广播，则会先启动该进程然后递送。








