# BroadcastReceiver源码分析
		BroadcastReceiver源码分析
1，简介
BroadcastReceiver，直译是“广播接收者”，在Android 系统中，广播主要是在组件之间进行消息传递。组件与组件之间可以是同一个进程，也可以是不同进程。既然是可以跨进程的，那么可以想像底层是通过Binder实现的，事实也却是如此。
2，为什么要有广播
既然是BroadcastReceiver基于Binder的，那么用纯Binder进行通信就行了，为什么还要创造出BroadcastReceiver？
我们知道AIDL也是基于Binder的，AIDL中采取的是proxy-stub模式分别在Client端对transact（），在Server端对onTransact（）进行封装。Client端拿到Server端的远程代理对象，才调用到Server端的方法，这需要在程序中事先写好Server端，以便提供远程代理对象。
如果程序事先不提供远程代理对象，而是由Client动态注册呢？这时候广播就登场了，在广播机制中，广播发送者不需要知道广播接收者的存在，这样就降低了发送者和接收者之间的耦合度，提高了程序的扩展性和维护性。
 
3，两种Receiver的注册
上图可以看出，在Android广播机制中，AMS扮演着路由器的角色，一根光纤进来，分出不同的线路发送给不同的接收者。因此广播的注册就是把广播接收者注册到AMS的过程（相当于，把PC网线查到路由器，才能接收到光纤的网络）。
Android系统中的Receiver分为动态和静态。动态receiver必须在运行期动态注册，其实际的注册动作由ContextImpl对象完成。静态receiver则是在AndroidManifest.xml中声明的。相比动态注册，静态稍显麻烦，因为在广播递送之时，静态receiver所从属的进程可能还没有启动呢，这就需要先启动新的进程，然后接收AMS发送消息。
3.1静态注册实例
首先编写一个接收器用来接受AMS发送来的广播
public class MyReceiver extends BroadcastReceiver {

	private static final String TAG = "MyReceiver";  
	
	@Override
	public void onReceive(Context context, Intent intent) {
		// TODO Auto-generated method stub 
                	doSomething();
	}

}
然后在AndroidManifest.xml中配置广播接收
<receiver android:name=".MyReceiver">
	<intent-filter>
		<action android:name="android.intent.action.MY_BROADCAST"/>
		<category android:name="android.intent.category.DEFAULT" />
	</intent-filter>
</receiver>

3.2动态注册实例
MyReceiver receiver = new MyReceiver();
IntentFilter filter = new IntentFilter();
filter.addAction("android.intent.action.MY_BROADCAST");
registerReceiver(receiver, filter);
4，BroadcastReceiver之源码分析
4.1，动态注册过程源码分析
拿在Activity中动态注册广播来说，在注册方法之前其实省略了Context，也就是实际上调用的是Context. registerReceiver()。Context是一个抽象类，它是Client和AMS,WMS等系统服务进行通信的接口，Activity和Service都是继承它的子类。Context的实现类是ContextImpl，也就是说调用到的是ContextImpl中的registerReceiver（）。




