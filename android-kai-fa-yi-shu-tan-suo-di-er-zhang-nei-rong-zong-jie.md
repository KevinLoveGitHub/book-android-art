---
title: 《Android 开发艺术探索》- 第二章内容总结
date: '2018-01-26T00:09:35.000Z'
tags:
  - 读书笔记
  - toc: true
---

# 第二章

## 多进程

### 创建多个进程阿斯顿发顺丰

Android 中使用多进程的方式只有一种，就是给四大组件在 `AndroidManifest.xml` 配置 `android:process` 属性：

{% code-tabs %}
{% code-tabs-item title="AndroidManifest.xml" %}
```markup
<activity android:name=".FirstActivity"
    android:process=":kevin">
</activity>
<activity android:name=".SecondActivity"
    android:process="org.lovedev.chapter_2.kevin">
</activity>
```
{% endcode-tabs-item %}
{% endcode-tabs %}

* 不配置 `android:process` 属性，默认进程名就是包名`org.lovedev.chapter_2`
* FirstActivity 的进程名是 `org.lovedev.chapter_2:kevin`，以 `:` 开头的进程名配置，属于当前应用的私有进程，其他应用的组件不可以和他跑在同一个进程当中
* SecondActivity 的进程名是 `org.lovedev.chapter_2.kevin`，其他应用可以通过 ShareUID 的方式和它跑在同一个进程中

启动两个 Activity 后，查看运行的进程：

![](http://ojxp1924f.bkt.clouddn.com/1516897403.png%20)

### 运行机制

Android 为每个进程都会分配一个独立的虚拟机，不同的虚拟机在内存分配上有不同的地址空间，这就导致了不同的虚拟机访问同一对象会产生多个副本。

举例说明，创建一个 Constant 类：

```java
public class Constant {
    public static int index = 1;
}
```

在 MainActivity 中修改 index 的值为 2，在 FirstActivity 中再次访问 index 的值，发现该值还是 1，并且打印 Constant 对象的 hashCode 值也是不同的。

![](http://ojxp1924f.bkt.clouddn.com/1516898310.png%20)

多进程还会造成以下问题：

* 静态成员和单例模式失效
* 线程同步机制失效
* SharedPreferences 可靠性下降，因为 SharedPreferences 底层是通过读/写 xml 文件实现的，并发写很可能会有问题，所以多进程中 SharedPreferences 同时执行写操作会导致数据丢失 
* Application 多次创建，虚拟机创建进程的过程就是启动一个应用的过程，运行在不同进程中的组件也具有不同的 Application，下图是启动不同进程 Activity 时，Application 中的打印信息

  ![](http://ojxp1924f.bkt.clouddn.com/1516899072.png%20)

## 序列化

当程序需要 Intent 或 Binder 传递数据时需要使用 Parcelable 或 Serializable 接口完成序列化过程，有时还需要把对象持久化到存储设备或通过网络传输给其他客户端，此时也需要 Serializable 来完成对象的持久化

### Serializable 接口

Serializable 接口是 Java 提供的序列化接口，为对象提供标准的序列化和反序列化操作。实现起来特别简单，只需要实现该接口就可以了，下面提供一个典型的持久化对象的例子：

```java
public void serializable(View view) {
    User user = new User();
    user.name = "kevin";
    user.age = 25;

    String filePath = Environment.getExternalStorageDirectory().getAbsolutePath() + "/user.txt";
    File file = new File(filePath);
    try {
        ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream(file));
        out.writeObject(user);
        out.close();
    } catch (IOException e) {
        e.printStackTrace();
    }

    try {
        ObjectInputStream input = new ObjectInputStream(new FileInputStream(file));
        User u = (User) input.readObject();
        input.close();
        Log.d(TAG, "serializable: " + u.name);
    } catch (IOException | ClassNotFoundException e) {
        e.printStackTrace();
    }
}
```

此外还需要注意的是实现 Serializable 接口的对象可以声明一个 serialVersionUID，有时即使不声明也不会影响对象的序列化和反序列化，这个 serialVersionUID 是用来辅助序列化和反序列化的，只有序列化后数据中的 serialVersionUID 和反序列化对象中的 serialVersionUID 相同才能反序列化成功，相当于类的唯一标识。需要注意的是静态变量属于类不属于该类的对象以及 `transient` 关键字标记的成员变量不参与序列化过程。

### Parcelable 接口

Parcelable 接口是 Android 提供的序列化接口，在 Andorid Studio 中有很多插件可以自动实现 Parcelable 接口，不需要手动写。

serializable 使用简单，但是开销很大，序列化和反序列化都需要 I/O 操作。Parcelable 使用麻烦，但有现成的插件，而且更适用于 Android 平台。Parcelable 主要用于内存序列化上，对于序列化对象到存储设备或网络传输，使用 Parcelable 稍显麻烦，这两种情况建议使用 Serializable。

## Binder

Binder 是 Android 的一个实现了 IBinder 接口的类，它是 Android 中特有的，Linux 中不在。从 IPC 的角度来看，它是 Android 中进程通讯的一种方式。可以把 Binder 理解为一个虚拟的物理设备，它的设备驱动是 `/dev/binder`；从 Android Framwork 层来说，Binder 是 ServiceManager 连接各种 Service 以及 Manager（ActivityManager、WindowService等）的桥梁；从应用层来说，Binder 是客户端和服务端进行通信的媒介。下面分析通过 `IBookManager.aidl` 生成的类：

```java
public interface IBookManager extends android.os.IInterface {
    /**
     * Local-side IPC implementation stub class.
     */
    public static abstract class Stub extends android.os.Binder implements org.lovedev.chapter_2.IBookManager {

        // Binder 的唯一标识，一般用当前的全类名表示
        private static final java.lang.String DESCRIPTOR = "org.lovedev.chapter_2.IBookManager";

        /**
         * Construct the stub at attach it to the interface.
         */
        public Stub() {
            this.attachInterface(this, DESCRIPTOR);
        }

        /**
         * 将服务器端的 Binder 对象转换为客户端所需要的 AIDL 接口类型对象
         * 该转换是区分进程的，如果客户端和服务端在同一个进程中，该方法就返回服务器的 Stub 对象本身
         * 否则返回的是系统封装后的 Stub.proxy 对象
         */
        public static org.lovedev.chapter_2.IBookManager asInterface(android.os.IBinder obj) {
            if ((obj == null)) {
                return null;
            }
            android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
            if (((iin != null) && (iin instanceof org.lovedev.chapter_2.IBookManager))) {
                return ((org.lovedev.chapter_2.IBookManager) iin);
            }
            return new org.lovedev.chapter_2.IBookManager.Stub.Proxy(obj);
        }

        /**
         * 返回当前 Binder 对象自身
         */
        @Override
        public android.os.IBinder asBinder() {
            return this;
        }

        /**
         * 该方法运行在 Binder 线程池中，当客户端对该 Binder 对象发起跨进程请求时，远程请求会通过系统底层封装后交由该方法进行处理
         * 服务端可以通过 code 参数判断所请求的方法是那个，然后从 data 中取出该方法所需参数，然后执行目标方法
         * 当目标方法执行完毕之后，就向 reply 中写入返回值（如果该方法有返回值的话）
         * 注意：如果该方法返回的 false，客户端的请求就会失败，可以利用该特性来做权限验证
         */
        @Override
        public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException {
            switch (code) {
                case INTERFACE_TRANSACTION: {
                    reply.writeString(DESCRIPTOR);
                    return true;
                }
                case TRANSACTION_getBookList: {
                    data.enforceInterface(DESCRIPTOR);
                    java.util.List<org.lovedev.chapter_2.Book> _result = this.getBookList();
                    reply.writeNoException();
                    reply.writeTypedList(_result);
                    return true;
                }
                case TRANSACTION_addBook: {
                    data.enforceInterface(DESCRIPTOR);
                    org.lovedev.chapter_2.Book _arg0;
                    if ((0 != data.readInt())) {
                        _arg0 = org.lovedev.chapter_2.Book.CREATOR.createFromParcel(data);
                    } else {
                        _arg0 = null;
                    }
                    this.addBook(_arg0);
                    reply.writeNoException();
                    return true;
                }
            }
            return super.onTransact(code, data, reply, flags);
        }

        private static class Proxy implements org.lovedev.chapter_2.IBookManager {
            private android.os.IBinder mRemote;

            Proxy(android.os.IBinder remote) {
                mRemote = remote;
            }

            @Override
            public android.os.IBinder asBinder() {
                return mRemote;
            }

            public java.lang.String getInterfaceDescriptor() {
                return DESCRIPTOR;
            }

            /**
             * 当客户端调用该方法时，首先创建该方法所需的 _data 和 _reply，然后把该方法的参数写入到 _data 中
             * 接着调用 transact 方法发起 RPC（远程调用）请求，同时当前线程挂起
             * 然后服务端的 onTransact 方法会被调用，直到 RPC 过程返回后，当前线程继续执行，从 _reply 中取出 RPC 过程的返回结果
             * 最后返回 _reply 中的数据
             */
            @Override
            public java.util.List<org.lovedev.chapter_2.Book> getBookList() throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                java.util.List<org.lovedev.chapter_2.Book> _result;
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
                    _reply.readException();
                    _result = _reply.createTypedArrayList(org.lovedev.chapter_2.Book.CREATOR);
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
                return _result;
            }

            @Override
            public void addBook(org.lovedev.chapter_2.Book book) throws android.os.RemoteException {
                android.os.Parcel _data = android.os.Parcel.obtain();
                android.os.Parcel _reply = android.os.Parcel.obtain();
                try {
                    _data.writeInterfaceToken(DESCRIPTOR);
                    if ((book != null)) {
                        _data.writeInt(1);
                        book.writeToParcel(_data, 0);
                    } else {
                        _data.writeInt(0);
                    }
                    mRemote.transact(Stub.TRANSACTION_addBook, _data, _reply, 0);
                    _reply.readException();
                } finally {
                    _reply.recycle();
                    _data.recycle();
                }
            }
        }

        // 为两个方法声明整型 ID 标识，用于区分客户端所请求的到底是那个方法
        static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
        static final int TRANSACTION_addBook = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    }

    public java.util.List<org.lovedev.chapter_2.Book> getBookList() throws android.os.RemoteException;

    public void addBook(org.lovedev.chapter_2.Book book) throws android.os.RemoteException;
}
```

通过上面的分析已经了解了 Binder 的工作机制，但还有两点需要注意一下：

* 客户端发起请求时，由于当前线程会被挂起，直至服务端进程返回数据，所以如果一个远程方法很耗时，就不能在 UI 线程中发起此远程请求
* 由于服务端的 Binder 方法运行在 Binder 线程池中，所以 Binder 方法不管是否耗时都应该采用同步方式去实现，因为它已经运行在一个线程中了。

## 进程间通讯

### 文件共享

对于数据同步要求不高的进程间通讯可以通过文件共享的方式完成。 Windows 中对文件加了排斥锁将会导致其他线程无法访问，包括读写，但是 Android 是基于 Linux 系统，所以对文件的并发读写完全没有限制。在使用文件共享这种方式的时候，记得处理好并发读写的情况

Android 中的 SharedPreferences 是一个轻量级的储存方案，它通过键值对来存储数据，其实底层实现上是采用 XML 文件来存储键值对，应用的 SharedPreferences 文件路径是 `/data/data/package_name/shared_prefs`，其中 `package_name` 是应用的包名。本质上讲 SharedPreferences 也是文件的一种，但是 Android 系统对于它的读写都有一定的缓存策略，即在内存中有一份 SharedPreferences 文件的缓存。因此在多进程环境下，系统对它的读写就变得不可靠，面对高并发的读写时，SharedPreferences 有很大几率会丢失数据。不建议在进程间通讯使用 SharedPreferences。

### Messager

可以使用 Messenger 在进程间传递 Message 对象消息，它的底层实现是 AIDL。

服务端进程需要创建一个 Service 处理客户端的连接请求，同时创建 Handler 对象，并作为参数创建一个 Messenger 对象，同时在 `onBind` 函数中返回该 Messenger 对象底层的 Binder ：

```java
public class MessengerService extends Service {

    private static final String TAG = "MessengerService";

    public MessengerService() {
    }

    // 2. 自定义 Handler 对象接收并处理消息
    private static final class MessengerHandler extends Handler {
        @Override
        public void handleMessage(Message msg) {
            Log.i(TAG, "MessengerService received message: " + msg.getData());
        }
    }

    // 1. 创建 Messenger 对象
    private final Messenger mMessenger = new Messenger(new MessengerHandler());

    @Override
    public IBinder onBind(Intent intent) {
        return mMessenger.getBinder();
    }
}
```

同时在 `AndroidManifest.xml` 中注册该 Service：

```markup
<service
    android:name=".MessengerService"
    android:enabled="true"
    android:exported="true"
    android:process=":remote">
    <intent-filter>
        <action android:name="org.lovedev.messenger"/>
        <category android:name="android.intent.category.DEFAULT"/>
    </intent-filter>
</service>
```

客户端需要根据服务端绑定 Service 时的 action 进行绑定：

```java
public class MessengerActivity extends AppCompatActivity {


    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_messenger);
        Intent service = new Intent();
        service.setAction("org.lovedev.messenger");
        // Android 5.0 以上必须执行该函数，否则会报 Service Intent must be explicit 错误
        service.setPackage(getPackageName());
        bindService(service, mConnection, Context.BIND_AUTO_CREATE);
    }

    private Messenger mMessenger;
    private ServiceConnection mConnection = new ServiceConnection() {
        @Override
        public void onServiceConnected(ComponentName name, IBinder service) {
            mMessenger = new Messenger(service);
        }

        @Override
        public void onServiceDisconnected(ComponentName name) {

        }
    };



    public void send(View view) {
        Message message = Message.obtain(null, 1);
        Bundle data = new Bundle();
        data.putString("data", "MessengerActivity's data");
        message.setData(data);
        try {
            mMessenger.send(message);
        } catch (RemoteException e) {
            e.printStackTrace();
        }
    }

    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(mConnection);
    }
}
```

在 Messenger 中传递数据时，必须将数据放入 Message 中传递，Messenger 和 Message 对象都实现了 Parcelable 接口，因此可以跨进程传输。Message 参数中有一个 `object` 字段在同一进程间通信非常实用，但是进程间通信，只有系统提供的实现了 Parcelable 接口的对象才可以通过它进行通信。

### AIDL

AIDL 文件支持的数据类型：

* 基本数据类型
* String 和 CharSequence
* List：只支持 ArrayList，且内部元素必须是被 AIDL 所支持
* Map：只支持 HashMap，key 和 value 必须被 AIDL 所支持
* 实现 Parcelable 接口的对象
* AIDL

自定义的 Parcelable 对象和 AIDL 对象必须显示 import 进来，在使用自定义 Parcelable 对象时，必须新建一个与该对象同名的 AIDL 文件，并且声明它为 Parcelable 类型，例如 `Book.aidl` 文件：

```text
package org.lovedev.chapter_2;
parcelable Book;
```

为了方便 AIDL 开发，一般情况下都把所有 AIDL 相关文件放入同一个包中，客户端和服务端需要保证 AIDL 文件包结构的一致性，否则会运行报错，因为客户端需要反序列化服务端中和 AIDL 接口相关的类，如果类的完整路径不一致，就无法反序列化成功，客户端在使用的时候也需要把实现 Parcelable 接口的对象拷贝到项目中，保持和服务端包名一致。

如果需要在服务端注册 listener，不能使用普通的 `List` 对象保存 listener，因为 AIDL 在传递对象的时候是通过反序列化获取对象的，所以前后两个对象不相等。此时就需要用 `RemoteCallbackList` 来保存客户端的 listener，`RemoteCallbackList` 内部维护一个 Map 对象，以 IBinder 对象作为 Key，listener 被封装为一个 `Callback` 对象。在需要删除 listener 的时候，遍历并找出该对象具有的 IBinder 所有 listener 中与之相同的进行删除。 `RemoteCallbackList` 在使用的时候需要注意 `beginBroadcast()` 和 `finishBroadcast()` 成对出现，`RemoteCallbackList` 内部实现了线程同步，使用的时候无须关心线程同步问题，而且当客户端进程终止的时候 `RemoteCallbackList` 会自动解除该客户端的所有监听。

处理 Binder 对象意外死亡有两种方式：

* 创建一个 `IBinder.DeathRecipient` 对象，并在 `binderDied` 函数中处理意外死亡后的逻辑，该函数在 Binder 线程被回调
* 在 `ServiceConnection` 的 `onServiceDisconnected` 函数中处理意外死亡，该函数在 UI 线程被回调

客户端调用服务端的远程函数时，客户端的调用线程会被挂起，`onServiceConnected` 和 `onServiceDisconnected` 都执行在 UI 线程，如果远程方法是耗时操作，客户端调用的时候又在 UI 线程，避免在这两个函数中调用远程函数。很有可能会导致 ANR。被调用的远程方法运行在 Binder 线程池中，服务端的函数本身可以执行耗时操作，没有必要再创建线程执行。

### ContentProvider

### Socket

上面两种使用方式比较常规多见，不再记录

### Binder 连接池

如果服务端程序存在很多个服务给客户端调用，如果用常规创建服务的方式的话，在应用详情中查看服务端程序就会存在很多服务，这是很不友好的。此时需要使用 Binder 连接池，创建 Binder 连接池和创建一般的服务流程是一样的，只不过提供了根据 Flag 查询并返回对应服务的接口

### IPC 方式总结

| 方式 | 优点 | 缺点 | 适用场景 |
| :---: | :---: | :---: | :---: |
| Bundle | 简单易用 | 只能传输 bundle 支持的类型 | 四大组件间的通讯 |
| 文件 | 简单易用 | 不适合高并发通讯，不能做到即时通讯 | 无并发，不需要即时通讯 |
| AIDL | 一对多并发通讯 即时通讯 | 需要自行处理线程同步 | 一对多通讯且有 RPC 需求 |
| Messenger | 一对多串行通讯 实时通讯 | 不支持高并发，不支持 RPC， 利用 bundle 传输， | 低并发一对多即时通讯， 无 RPC 需求，或者不需要返回值的 RPC 需求 |
| ContentProvider | 一对多并发数据共享 | 理解为受约束的 AIDL，提供数据源的 CRUD 操作 | 一对多进程间的数据共享 |
| Socket | 功能强大，通过网络传输字节流，支持一对多实时通讯 | 实现复杂，不支持直接的 RPC | 使用网络交换数据 |

### 服务启动方式

![](http://ojxp1924f.bkt.clouddn.com/1526021949.png%20)

| 方式 | 区别 |
| :---: | :---: |
| startService | 执行后台任务，不进行通信。停止服务使用stopService |
| bindService | 进行通信。停止服务使用unbindService |
| startService&bindService | 使用stopService与unbindService |

### 注意事项

* 如果 `onBind` 返回的是 null，则不会调用 `onServiceConnected` 和 `onServiceDisconnected` 函数
* 调用unBinder接口时，service先调用onUnBind，然后调用onDestroy，但是最后没有调用 `onServiceDisconnected` 接口，这个接口只有异常出错才会调用

