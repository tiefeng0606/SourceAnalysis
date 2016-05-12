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
源码解析见[BroadcastReceiver源码解析（二）](http://blog.csdn.net/tiefeng0606/article/details/51381221)







