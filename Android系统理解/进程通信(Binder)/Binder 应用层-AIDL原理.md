Binder系列：
[Binder Kernel层—Binder内核驱动](https://www.jianshu.com/p/fdf12cd7c28d)
[Binder Native层—服务管理(ServiceManager进程)](https://www.jianshu.com/p/fb514e3eed3a)
[Binder Native层—注册/查询服务](https://www.jianshu.com/p/a7e5bbb0ab72)
[Binder Framework层—注册和查询服务](https://www.jianshu.com/p/60be38017422)
[Binder 应用层-AIDL原理](https://www.jianshu.com/p/7ca78da80f95)

## AIDL概述
> 本篇文章讲解AIDL的原理，对于AIDL的使用详解可参考文档[Android 接口定义语言 (AIDL)](https://developer.android.google.cn/guide/components/aidl.html?hl=zh-cn#Implement)和[Android AIDL使用详解](https://www.jianshu.com/p/29999c1a93cd)

### AIDL支持的数据类型：
1. 8种基本的数据类型：int、long、float、double、char、byte、short、boolean。
2. String、CharSequence
3. List<AIDL支持的数据类型>、Map<AIDL支持的数据类型,AIDL支持的数据类型>
4. 声明为parcelable的aidl接口，先创建一个aidl接口，然后重建一个同名的java类并实现Parcelable序列化。
  ```
// aidl接口
parcelable Book;

// java类
public class Book implements Parcelable {
    private String name;
    // ...
    protected Book(Parcel in) {
        this.name = in.readString();
    }

    public static final Creator<Book> CREATOR = new Creator<Book>() {
        @Override
        public Book createFromParcel(Parcel in) {
            return new Book(in);
        }

        @Override
        public Book[] newArray(int size) {
            return new Book[size];
        }
    };

    @Override
    public int describeContents() {
        return 0;
    }

    @Override
    public void writeToParcel(Parcel parcel, int i) {
        parcel.writeString(name);
    }


    public void readFromParcel(Parcel dest) {
        name = dest.readString();
    }

}

  ```
****
### AIDL的使用流程：
1. AIDL是一种模版接口，通过该接口定义Server端对外提供的方法。AIDL接口通过syn gradle自动生成一个Service端的Binder对象`Stub`，和一个client端的代理对象`Proxy`。
  ```
// BookController.aidl
package com.czy.server;
import com.czy.server.Book;

// Declare any non-default types here with import statements

interface BookController {
      List<Book> getBookList();
     void addBookInOut(inout Book book);

     void addBookIn(in Book book);

     void addBookOut(out Book book);
}

// 自动生成一个Java类
public interface BookController extends android.os.IInterface
{
  
  /** Local-side IPC implementation stub class. */
  public static abstract class Stub extends android.os.Binder implements com.czy.server.BookController
  {
    private static final java.lang.String DESCRIPTOR = "com.czy.server.BookController";
    /** Construct the stub at attach it to the interface. */
    public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    }
    /**
     * Cast an IBinder object into an com.czy.server.BookController interface,
     * generating a proxy if needed.
     */
    public static com.czy.server.BookController asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.czy.server.BookController))) {
        return ((com.czy.server.BookController)iin);
      }
      return new com.czy.server.BookController.Stub.Proxy(obj);
    }
    @Override public android.os.IBinder asBinder()
    {
      return this;
    }
    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code)
      {
        case INTERFACE_TRANSACTION:
        {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_getBookList:
        {
          data.enforceInterface(descriptor);
          java.util.List<com.czy.server.Book> _result = this.getBookList();
          reply.writeNoException();
          reply.writeTypedList(_result);
          return true;
        }
        case TRANSACTION_addBookInOut:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          if ((0!=data.readInt())) {
            _arg0 = com.czy.server.Book.CREATOR.createFromParcel(data);
          }
          else {
            _arg0 = null;
          }
          this.addBookInOut(_arg0);
          reply.writeNoException();
          if ((_arg0!=null)) {
            reply.writeInt(1);
            _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          }
          else {
            reply.writeInt(0);
          }
          return true;
        }
        case TRANSACTION_addBookIn:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          if ((0!=data.readInt())) {
            _arg0 = com.czy.server.Book.CREATOR.createFromParcel(data);
          }
          else {
            _arg0 = null;
          }
          this.addBookIn(_arg0);
          reply.writeNoException();
          return true;
        }
        case TRANSACTION_addBookOut:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          _arg0 = new com.czy.server.Book();
          this.addBookOut(_arg0);
          reply.writeNoException();
          if ((_arg0!=null)) {
            reply.writeInt(1);
            _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          }
          else {
            reply.writeInt(0);
          }
          return true;
        }
        default:
        {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }
    private static class Proxy implements com.czy.server.BookController
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;
      }
      @Override public java.util.List<com.czy.server.Book> getBookList() throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.util.List<com.czy.server.Book> _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getBookList();
          }
          _reply.readException();
          _result = _reply.createTypedArrayList(com.czy.server.Book.CREATOR);
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
      @Override public void addBookInOut(com.czy.server.Book book) throws android.os.RemoteException
      {
        //...
      }
      @Override public void addBookIn(com.czy.server.Book book) throws android.os.RemoteException
      {
        //...
      }
      @Override public void addBookOut(com.czy.server.Book book) throws android.os.RemoteException
      {
       //...
      }
      public static com.czy.server.BookController sDefaultImpl;
    }
    static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_addBookInOut = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    static final int TRANSACTION_addBookIn = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    static final int TRANSACTION_addBookOut = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
    public static boolean setDefaultImpl(com.czy.server.BookController impl) {
      // Only one user of this interface can use this function
      // at a time. This is a heuristic to detect if two different
      // users in the same process use this function.
      if (Stub.Proxy.sDefaultImpl != null) {
        throw new IllegalStateException("setDefaultImpl() called twice");
      }
      if (impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }
    public static com.czy.server.BookController getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }
  public java.util.List<com.czy.server.Book> getBookList() throws android.os.RemoteException;
  public void addBookInOut(com.czy.server.Book book) throws android.os.RemoteException;
  public void addBookIn(com.czy.server.Book book) throws android.os.RemoteException;
  public void addBookOut(com.czy.server.Book book) throws android.os.RemoteException;
}
  ```

2. Service端通过建立一个Service，在其onBind方法中返回AIDL接口自动生成的Binder对象(Stub),并实现Server端对外提供的方法。
  ```
public class AIDLService extends Service {

     //... 
    private final BookController.Stub stub = new BookController.Stub() {
        @Override
        public List<Book> getBookList() throws RemoteException {
            return bookList;
        }
        // ... 接口方法的实现
    };


    @Nullable
    @Override
    public IBinder onBind(Intent intent) {
        return stub;
    }

}
  ```

3. Client通过启动Server端的Service，并利用ServiceConnection进行监听获取到Client端的代理对象Proxy，并通过Proxy进行对Server方法的调用。

  ```
   private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(p0: ComponentName?, p1: IBinder?) {
            bookController = BookController.Stub.asInterface(p1)
            isConnected = true
        }

        override fun onServiceDisconnected(p0: ComponentName?) {
            bookController = null
            isConnected = true
        }
    }

 val intent = Intent(this,com.aidl.server.AIDLService)
        intent.also {
            bindService(it, serviceConnection, Context.BIND_AUTO_CREATE)
        }
  ```

## AIDL的原理
AIDL是基于Binder机制进行通信，通过模版接口进行编译自动生成一个包含Server端的Binder对象(Stub)，和一个Client的代理对象(Proxy)：
  ```
public interface BookController extends android.os.IInterface
{

  /** Local-side IPC implementation stub class. */
  public static abstract class Stub extends android.os.Binder implements com.czy.server.BookController
  {
    private static final java.lang.String DESCRIPTOR = "com.czy.server.BookController";
    /** Construct the stub at attach it to the interface. */
    public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    }
    /**
     * Cast an IBinder object into an com.czy.server.BookController interface,
     * generating a proxy if needed.
     */
    public static com.czy.server.BookController asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.czy.server.BookController))) {
        return ((com.czy.server.BookController)iin);
      }
      return new com.czy.server.BookController.Stub.Proxy(obj);
    }
    @Override public android.os.IBinder asBinder()
    {
      return this;
    }
    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code)
      {
        case INTERFACE_TRANSACTION:
        {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_getBookList:
        {
          data.enforceInterface(descriptor);
          java.util.List<com.czy.server.Book> _result = this.getBookList();
          reply.writeNoException();
          reply.writeTypedList(_result);
          return true;
        }
        case TRANSACTION_addBookInOut:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          if ((0!=data.readInt())) {
            _arg0 = com.czy.server.Book.CREATOR.createFromParcel(data);
          }
          else {
            _arg0 = null;
          }
          this.addBookInOut(_arg0);
          reply.writeNoException();
          if ((_arg0!=null)) {
            reply.writeInt(1);
            _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          }
          else {
            reply.writeInt(0);
          }
          return true;
        }
        case TRANSACTION_addBookIn:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          if ((0!=data.readInt())) {
            _arg0 = com.czy.server.Book.CREATOR.createFromParcel(data);
          }
          else {
            _arg0 = null;
          }
          this.addBookIn(_arg0);
          reply.writeNoException();
          return true;
        }
        case TRANSACTION_addBookOut:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          _arg0 = new com.czy.server.Book();
          this.addBookOut(_arg0);
          reply.writeNoException();
          if ((_arg0!=null)) {
            reply.writeInt(1);
            _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          }
          else {
            reply.writeInt(0);
          }
          return true;
        }
        default:
        {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }
    private static class Proxy implements com.czy.server.BookController
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;
      }
      @Override public java.util.List<com.czy.server.Book> getBookList() throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.util.List<com.czy.server.Book> _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getBookList();
          }
          _reply.readException();
          _result = _reply.createTypedArrayList(com.czy.server.Book.CREATOR);
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
      @Override public void addBookInOut(com.czy.server.Book book) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          if ((book!=null)) {
            _data.writeInt(1);
            book.writeToParcel(_data, 0);
          }
          else {
            _data.writeInt(0);
          }
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookInOut, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookInOut(book);
            return;
          }
          _reply.readException();
          if ((0!=_reply.readInt())) {
            book.readFromParcel(_reply);
          }
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
      @Override public void addBookIn(com.czy.server.Book book) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          if ((book!=null)) {
            _data.writeInt(1);
            book.writeToParcel(_data, 0);
          }
          else {
            _data.writeInt(0);
          }
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookIn, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookIn(book);
            return;
          }
          _reply.readException();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
      @Override public void addBookOut(com.czy.server.Book book) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookOut, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookOut(book);
            return;
          }
          _reply.readException();
          if ((0!=_reply.readInt())) {
            book.readFromParcel(_reply);
          }
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
      public static com.czy.server.BookController sDefaultImpl;
    }
    static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_addBookInOut = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    static final int TRANSACTION_addBookIn = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    static final int TRANSACTION_addBookOut = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
    public static boolean setDefaultImpl(com.czy.server.BookController impl) {
      // Only one user of this interface can use this function
      // at a time. This is a heuristic to detect if two different
      // users in the same process use this function.
      if (Stub.Proxy.sDefaultImpl != null) {
        throw new IllegalStateException("setDefaultImpl() called twice");
      }
      if (impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }
    public static com.czy.server.BookController getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }

  // 接口方法
  public java.util.List<com.czy.server.Book> getBookList() throws android.os.RemoteException;
  public void addBookInOut(com.czy.server.Book book) throws android.os.RemoteException;
  public void addBookIn(com.czy.server.Book book) throws android.os.RemoteException;
  public void addBookOut(com.czy.server.Book book) throws android.os.RemoteException;
}
  ```
### 1. 首先定义Java接口
首先根据aidl文件，生成一个同名的Java接口，并继承IInterface
  ```
public interface BookController extends android.os.IInterface
{
  // 接口方法
  public java.util.List<com.czy.server.Book> getBookList() throws android.os.RemoteException;
  public void addBookInOut(com.czy.server.Book book) throws android.os.RemoteException;
  public void addBookIn(com.czy.server.Book book) throws android.os.RemoteException;
  public void addBookOut(com.czy.server.Book book) throws android.os.RemoteException;
}
  ```
### 2. Stub
生成一个代表Server端Binder的抽象类Stub，该Stub继承Binder类，并继承BookController接口，但Stub并没有实现BookController接口的方法而是留给Service实现。
  ```
  /** Local-side IPC implementation stub class. */
  public static abstract class Stub extends android.os.Binder implements com.czy.server.BookController
  {
    private static final java.lang.String DESCRIPTOR = "com.czy.server.BookController";
    /** Construct the stub at attach it to the interface. */
    public Stub()
    {
      this.attachInterface(this, DESCRIPTOR);
    }
    
    // 1. asInterface方法返回Proxy客户端的代理对象
    public static com.czy.server.BookController asInterface(android.os.IBinder obj)
    {
      if ((obj==null)) {
        return null;
      }
      android.os.IInterface iin = obj.queryLocalInterface(DESCRIPTOR);
      if (((iin!=null)&&(iin instanceof com.czy.server.BookController))) {
        return ((com.czy.server.BookController)iin);
      }
      return new com.czy.server.BookController.Stub.Proxy(obj);
    }

    @Override public android.os.IBinder asBinder()
    {
      return this;
    }

     // 处理Client端的请求
    @Override public boolean onTransact(int code, android.os.Parcel data, android.os.Parcel reply, int flags) throws android.os.RemoteException
    {
      java.lang.String descriptor = DESCRIPTOR;
      switch (code)
      {
        case INTERFACE_TRANSACTION:
        {
          reply.writeString(descriptor);
          return true;
        }
        case TRANSACTION_getBookList:
        {
          data.enforceInterface(descriptor);
          // 调用实例的getBookList()方法
          java.util.List<com.czy.server.Book> _result = this.getBookList();
          reply.writeNoException();
          // 把返回的数据回传给Client端
          reply.writeTypedList(_result);
          return true;
        }
        case TRANSACTION_addBookInOut:
        {
          data.enforceInterface(descriptor);
          com.czy.server.Book _arg0;
          if ((0!=data.readInt())) {
            _arg0 = com.czy.server.Book.CREATOR.createFromParcel(data);
          }
          else {
            _arg0 = null;
          }
          // 调用实例的addBookInOut方法
          this.addBookInOut(_arg0);
          reply.writeNoException();
          if ((_arg0!=null)) {
            reply.writeInt(1);
            _arg0.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
          }
          else {
            reply.writeInt(0);
          }
          return true;
        }
     
        default:
        {
          return super.onTransact(code, data, reply, flags);
        }
      }
    }

     // 逐渐+1的方式生存方法的对应的命令协议
    static final int TRANSACTION_getBookList = (android.os.IBinder.FIRST_CALL_TRANSACTION + 0);
    static final int TRANSACTION_addBookInOut = (android.os.IBinder.FIRST_CALL_TRANSACTION + 1);
    static final int TRANSACTION_addBookIn = (android.os.IBinder.FIRST_CALL_TRANSACTION + 2);
    static final int TRANSACTION_addBookOut = (android.os.IBinder.FIRST_CALL_TRANSACTION + 3);
    public static boolean setDefaultImpl(com.czy.server.BookController impl) {
      // Only one user of this interface can use this function
      // at a time. This is a heuristic to detect if two different
      // users in the same process use this function.
      if (Stub.Proxy.sDefaultImpl != null) {
        throw new IllegalStateException("setDefaultImpl() called twice");
      }
      if (impl != null) {
        Stub.Proxy.sDefaultImpl = impl;
        return true;
      }
      return false;
    }
    public static com.czy.server.BookController getDefaultImpl() {
      return Stub.Proxy.sDefaultImpl;
    }
  }
  ```
Stub对象的`asInterface`方法返回一个客户端的Proxy对象。`onTransact()`方法根据命令协议执行不同的方法。

### 3. Proxy
客户端代理对象Proxy,其并没有继承Binder,而是继承了接口。
  ```
    private static class Proxy implements com.czy.server.BookController
    {
      private android.os.IBinder mRemote;
      Proxy(android.os.IBinder remote)
      {
        mRemote = remote;
      }
      @Override public android.os.IBinder asBinder()
      {
        return mRemote;
      }
      public java.lang.String getInterfaceDescriptor()
      {
        return DESCRIPTOR;
      }
      @Override public java.util.List<com.czy.server.Book> getBookList() throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        java.util.List<com.czy.server.Book> _result;
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_getBookList, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            return getDefaultImpl().getBookList();
          }
          _reply.readException();
          _result = _reply.createTypedArrayList(com.czy.server.Book.CREATOR);
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
        return _result;
      }
      @Override public void addBookInOut(com.czy.server.Book book) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          if ((book!=null)) {
            _data.writeInt(1);
            book.writeToParcel(_data, 0);
          }
          else {
            _data.writeInt(0);
          }
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookInOut, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookInOut(book);
            return;
          }
          _reply.readException();
          if ((0!=_reply.readInt())) {
            book.readFromParcel(_reply);
          }
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
      @Override public void addBookIn(com.czy.server.Book book) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          if ((book!=null)) {
            _data.writeInt(1);
            book.writeToParcel(_data, 0);
          }
          else {
            _data.writeInt(0);
          }
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookIn, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookIn(book);
            return;
          }
          _reply.readException();
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
      @Override public void addBookOut(com.czy.server.Book book) throws android.os.RemoteException
      {
        android.os.Parcel _data = android.os.Parcel.obtain();
        android.os.Parcel _reply = android.os.Parcel.obtain();
        try {
          _data.writeInterfaceToken(DESCRIPTOR);
          boolean _status = mRemote.transact(Stub.TRANSACTION_addBookOut, _data, _reply, 0);
          if (!_status && getDefaultImpl() != null) {
            getDefaultImpl().addBookOut(book);
            return;
          }
          _reply.readException();
          if ((0!=_reply.readInt())) {
            book.readFromParcel(_reply);
          }
        }
        finally {
          _reply.recycle();
          _data.recycle();
        }
      }
      public static com.czy.server.BookController sDefaultImpl;
    }
  ```


