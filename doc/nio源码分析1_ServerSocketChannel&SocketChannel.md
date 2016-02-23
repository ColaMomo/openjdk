# Nio 

nio网络编程主要涉及以下几个类

1. ServerSocketChannel、SocketChannel
2. Selector、SelectionKey
3. ByteBuffer

## ServerSocketChannel

ServerSocketChannel 包含两个重要的接口：  
1. bind： 绑定服务端端口，监听请求  
2. accept： 接收客户端连接

```
java.nio.channels.ServerSocketChannel.java

public abstract class ServerSocketChannel
    extends AbstractSelectableChannel
    implements NetworkChannel
{
    protected ServerSocketChannel(SelectorProvider provider) {
        super(provider);
    }

	//创建ServerSocketChannel
    public static ServerSocketChannel open() throws IOException {
        return SelectorProvider.provider().openServerSocketChannel();
    }

	//支持的事件
    public final int validOps() {
        return SelectionKey.OP_ACCEPT;
    }

	//绑定服务端端口
    public final ServerSocketChannel bind(SocketAddress local)
        throws IOException
    {
        return bind(local, 0);
    }
    
	//绑定服务端端口
    public abstract ServerSocketChannel bind(SocketAddress local, int backlog)
        throws IOException;

    public abstract <T> ServerSocketChannel setOption(SocketOption<T> name, T value)
        throws IOException;

 	//获取对应的ServerSocket
    public abstract ServerSocket socket();

	//接收连接请求
    public abstract SocketChannel accept() throws IOException;

}
```

sun.nio.ch.ServerSocketChannelImpl.java

属性  

```
    private static NativeDispatcher nd;
    private final FileDescriptor fd;
    private int fdVal;

    private volatile long thread = 0;

    private final Object lock = new Object();
    private final Object stateLock = new Object();

    private static final int ST_UNINITIALIZED = -1;
    private static final int ST_INUSE = 0;
    private static final int ST_KILLED = 1;
    private int state = ST_UNINITIALIZED;

    private SocketAddress localAddress; // null => unbound

    ServerSocket socket;
```

#### bind

bind() 方法，用于绑定端口，监听连接 

```
    @Override
    public ServerSocketChannel bind(SocketAddress local, int backlog) throws IOException {
        synchronized (lock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if (isBound())
                throw new AlreadyBoundException();
            InetSocketAddress isa = (local == null) ? new InetSocketAddress(0) :
                Net.checkAddress(local);
            SecurityManager sm = System.getSecurityManager();
            if (sm != null)
                sm.checkListen(isa.getPort());
            NetHooks.beforeTcpBind(fd, isa.getAddress(), isa.getPort());
            Net.bind(fd, isa.getAddress(), isa.getPort());
            Net.listen(fd, backlog < 1 ? 50 : backlog);
            synchronized (stateLock) {
                localAddress = Net.localAddress(fd);
            }
        }
        return this;
    }
```

#### accept

accept() 方法，用于接收客户端的连接

```
    public SocketChannel accept() throws IOException {
        synchronized (lock) {
			...
            SocketChannel sc = null;

            int n = 0;
            FileDescriptor newfd = new FileDescriptor();
            InetSocketAddress[] isaa = new InetSocketAddress[1];

            try {
                begin();
                if (!isOpen())
                    return null;
                thread = NativeThread.current();
                for (;;) {
                    n = accept0(this.fd, newfd, isaa);
                    if ((n == IOStatus.INTERRUPTED) && isOpen())
                        continue;
                    break;
                }
            } finally {
                thread = 0;
                end(n > 0);
                assert IOStatus.check(n);
            }

            if (n < 1)
                return null;

            IOUtil.configureBlocking(newfd, true);
            InetSocketAddress isa = isaa[0];
            sc = new SocketChannelImpl(provider(), newfd, isa);
            SecurityManager sm = System.getSecurityManager();
            if (sm != null) {
                try {
                    sm.checkAccept(isa.getAddress().getHostAddress(),
                                   isa.getPort());
                } catch (SecurityException x) {
                    sc.close();
                    throw x;
                }
            }
            return sc;
        }
    }

	//本地方法接收连接
    private native int accept0(FileDescriptor ssfd, FileDescriptor newfd,
                               InetSocketAddress[] isaa)
        throws IOException;
```

socket() 方法获取ServerSocketChannel对应的ServerSocket

```
    public ServerSocket socket() {
        synchronized (stateLock) {
            if (socket == null)
                socket = ServerSocketAdaptor.create(this);
            return socket;
        }
    }
```

配置ServerSocketChannel选项相关接口：  

```
    @Override
    public <T> ServerSocketChannel setOption(SocketOption<T> name, T value)
        throws IOException
    {
        if (name == null)
            throw new NullPointerException();
        if (!supportedOptions().contains(name))
            throw new UnsupportedOperationException("'" + name + "' not supported");

        synchronized (stateLock) {
            if (!isOpen())
                throw new ClosedChannelException();

            Net.setSocketOption(fd, Net.UNSPEC, name, value);
            return this;
        }
    }

    @Override
    public <T> T getOption(SocketOption<T> name)
        throws IOException
    {
        if (name == null)
            throw new NullPointerException();
        if (!supportedOptions().contains(name))
            throw new UnsupportedOperationException("'" + name + "' not supported");

        synchronized (stateLock) {
            if (!isOpen())
                throw new ClosedChannelException();

            return (T) Net.getSocketOption(fd, Net.UNSPEC, name);
        }
    }

    private static class DefaultOptionsHolder {
        static final Set<SocketOption<?>> defaultOptions = defaultOptions();

        private static Set<SocketOption<?>> defaultOptions() {
            HashSet<SocketOption<?>> set = new HashSet<SocketOption<?>>(2);
            set.add(StandardSocketOptions.SO_RCVBUF);
            set.add(StandardSocketOptions.SO_REUSEADDR);
            return Collections.unmodifiableSet(set);
        }
    }

    @Override
    public final Set<SocketOption<?>> supportedOptions() {
        return DefaultOptionsHolder.defaultOptions;
    }
```

接口提供的事件如下：  
SelectionKey.OP_READ  
SelectionKey.OP_WRITE  
SelectionKey.OP_CONNECT  
SelectionKey.OP_ACCEPT  
底层的epoll接受的事件  
PollArrayWrapper.POLLIN  
PollArrayWrapper.POLLOUT  
PollArrayWrapper.POLLERR  
PollArrayWrapper.POLLHUP  
PollArrayWrapper.POLLNVAL  
PollArrayWrapper.POLLREMOVE​  
translateReadyOps() 方法将系统事件转换为接口事件


```
    public boolean translateReadyOps(int ops, int initialOps,
                                     SelectionKeyImpl sk) {
        int intOps = sk.nioInterestOps(); 
        int oldOps = sk.nioReadyOps();
        int newOps = initialOps;

        if ((ops & PollArrayWrapper.POLLNVAL) != 0) {
            return false;
        }

        if ((ops & (PollArrayWrapper.POLLERR
                    | PollArrayWrapper.POLLHUP)) != 0) {
            newOps = intOps;
            sk.nioReadyOps(newOps);
            return (newOps & ~oldOps) != 0;
        }

        if (((ops & PollArrayWrapper.POLLIN) != 0) &&
            ((intOps & SelectionKey.OP_ACCEPT) != 0))
                newOps |= SelectionKey.OP_ACCEPT;

        sk.nioReadyOps(newOps);
        return (newOps & ~oldOps) != 0;
    }

    public boolean translateAndUpdateReadyOps(int ops, SelectionKeyImpl sk) {
        return translateReadyOps(ops, sk.nioReadyOps(), sk);
    }

    public boolean translateAndSetReadyOps(int ops, SelectionKeyImpl sk) {
        return translateReadyOps(ops, 0, sk);
    }

    public void translateAndSetInterestOps(int ops, SelectionKeyImpl sk) {
        int newOps = 0;

        if ((ops & SelectionKey.OP_ACCEPT) != 0)
            newOps |= PollArrayWrapper.POLLIN;
        sk.selector.putEventOps(sk, newOps);
    }
```

## SocketChannel

socketChannel包含三个主要接口：

1. connect() 连接服务端
2. read() 接收并读取数据
3. write() 写入并发送数据

```
java.nio.channels.SocketChannel.java

public abstract class SocketChannel
    extends AbstractSelectableChannel
    implements ByteChannel, ScatteringByteChannel, GatheringByteChannel, NetworkChannel
{

    protected SocketChannel(SelectorProvider provider) {
        super(provider);
    }

    public static SocketChannel open() throws IOException {
        return SelectorProvider.provider().openSocketChannel();
    }

    public static SocketChannel open(SocketAddress remote)
        throws IOException
    {
        SocketChannel sc = open();
        try {
            sc.connect(remote);
        } catch (Throwable x) {
            try {
                sc.close();
            } catch (Throwable suppressed) {
                x.addSuppressed(suppressed);
            }
            throw x;
        }
        assert sc.isConnected();
        return sc;
    }

  	//SocketChannel支持的事件
    public final int validOps() {
        return (SelectionKey.OP_READ
                | SelectionKey.OP_WRITE
                | SelectionKey.OP_CONNECT);
    }

    @Override
    public abstract SocketChannel bind(SocketAddress local)
        throws IOException;

    @Override
    public abstract <T> SocketChannel setOption(SocketOption<T> name, T value)
        throws IOException;

    public abstract SocketChannel shutdownInput() throws IOException;

    public abstract SocketChannel shutdownOutput() throws IOException;

    public abstract Socket socket();

    public abstract boolean isConnected();

    public abstract boolean isConnectionPending();

    public abstract boolean connect(SocketAddress remote) throws IOException;

    public abstract boolean finishConnect() throws IOException;

    public abstract SocketAddress getRemoteAddress() throws IOException;

    public abstract int read(ByteBuffer dst) throws IOException;

    public abstract long read(ByteBuffer[] dsts, int offset, int length)
        throws IOException;

    public final long read(ByteBuffer[] dsts) throws IOException {
        return read(dsts, 0, dsts.length);
    }

    public abstract int write(ByteBuffer src) throws IOException;

    public abstract long write(ByteBuffer[] srcs, int offset, int length)
        throws IOException;

    public final long write(ByteBuffer[] srcs) throws IOException {
        return write(srcs, 0, srcs.length);
    }

}
```

属性：

```
    private static NativeDispatcher nd;
	
	//文件描述符fd
    private final FileDescriptor fd;
    private final int fdVal;

    //本地读写线程id
    private volatile long readerThread = 0;
    private volatile long writerThread = 0;

	//锁
    private final Object readLock = new Object();
    private final Object writeLock = new Object();
    private final Object stateLock = new Object();

	//状态
    private static final int ST_UNINITIALIZED = -1;
    private static final int ST_UNCONNECTED = 0;
    private static final int ST_PENDING = 1;
    private static final int ST_CONNECTED = 2;
    private static final int ST_KILLPENDING = 3;
    private static final int ST_KILLED = 4;
    private int state = ST_UNINITIALIZED;

    //本地地址&远程地址
    private SocketAddress localAddress;
    private SocketAddress remoteAddress;

    // Input/Output open
    private boolean isInputOpen = true;
    private boolean isOutputOpen = true;
    private boolean readyToConnect = false;

    //socket
    private Socket socket;
```

#### read 

read() 方法用于接收数据

```
  public int read(ByteBuffer buf) throws IOException {

        if (buf == null)
            throw new NullPointerException();

        synchronized (readLock) {
            if (!ensureReadOpen())
                return -1;
            int n = 0;
            try {
                begin();

                synchronized (stateLock) {
                    if (!isOpen()) {
                        return 0;
                    }

                    readerThread = NativeThread.current();
                }

                for (;;) {
                    n = IOUtil.read(fd, buf, -1, nd, readLock);
                    if ((n == IOStatus.INTERRUPTED) && isOpen()) {
                        continue;
                    }
                    return IOStatus.normalize(n);
                }

            } finally {
                readerCleanup();        
                end(n > 0 || (n == IOStatus.UNAVAILABLE));

                synchronized (stateLock) {
                    if ((n <= 0) && (!isInputOpen))
                        return IOStatus.EOF;
                }

                assert IOStatus.check(n);
            }
        }
    }
```

```
IOUtil.java

    static int read(FileDescriptor fd, ByteBuffer dst, long position,
                    NativeDispatcher nd, Object lock)
        throws IOException
    {
        if (dst.isReadOnly())
            throw new IllegalArgumentException("Read-only buffer");
        if (dst instanceof DirectBuffer)
            return readIntoNativeBuffer(fd, dst, position, nd, lock);

        ByteBuffer bb = Util.getTemporaryDirectBuffer(dst.remaining());
        try {
            int n = readIntoNativeBuffer(fd, bb, position, nd, lock);
            bb.flip();
            if (n > 0)
                dst.put(bb);
            return n;
        } finally {
            Util.offerFirstTemporaryDirectBuffer(bb);
        }
    }
    
    private static int readIntoNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                            long position, NativeDispatcher nd,
                                            Object lock)
        throws IOException
    {
        int pos = bb.position();
        int lim = bb.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);

        if (rem == 0)
            return 0;
        int n = 0;
        if (position != -1) {
            n = nd.pread(fd, ((DirectBuffer)bb).address() + pos,
                         rem, position, lock);
        } else {
            n = nd.read(fd, ((DirectBuffer)bb).address() + pos, rem);
        }
        if (n > 0)
            bb.position(pos + n);
        return n;
    }
```

#### write

write() 方法用于发送数据

```
    public int write(ByteBuffer buf) throws IOException {
        if (buf == null)
            throw new NullPointerException();
        synchronized (writeLock) {
            ensureWriteOpen();
            int n = 0;
            try {
                begin();
                synchronized (stateLock) {
                    if (!isOpen())
                        return 0;
                    writerThread = NativeThread.current();
                }
                for (;;) {
                    n = IOUtil.write(fd, buf, -1, nd, writeLock);
                    if ((n == IOStatus.INTERRUPTED) && isOpen())
                        continue;
                    return IOStatus.normalize(n);
                }
            } finally {
                writerCleanup();
                end(n > 0 || (n == IOStatus.UNAVAILABLE));
                synchronized (stateLock) {
                    if ((n <= 0) && (!isOutputOpen))
                        throw new AsynchronousCloseException();
                }
                assert IOStatus.check(n);
            }
        }
    }
```

```
IOUtil.java

    static int write(FileDescriptor fd, ByteBuffer src, long position,
                     NativeDispatcher nd, Object lock)
        throws IOException
    {
        if (src instanceof DirectBuffer)
            return writeFromNativeBuffer(fd, src, position, nd, lock);

        int pos = src.position();
        int lim = src.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);
        ByteBuffer bb = Util.getTemporaryDirectBuffer(rem);
        try {
            bb.put(src);
            bb.flip();
            src.position(pos);

            int n = writeFromNativeBuffer(fd, bb, position, nd, lock);
            if (n > 0) {
                src.position(pos + n);
            }
            return n;
        } finally {
            Util.offerFirstTemporaryDirectBuffer(bb);
        }
    }
```

```
    private static int writeFromNativeBuffer(FileDescriptor fd, ByteBuffer bb,
                                           long position, NativeDispatcher nd,
                                             Object lock)
        throws IOException
    {
        int pos = bb.position();
        int lim = bb.limit();
        assert (pos <= lim);
        int rem = (pos <= lim ? lim - pos : 0);

        int written = 0;
        if (rem == 0)
            return 0;
        if (position != -1) {
            written = nd.pwrite(fd,
                                ((DirectBuffer)bb).address() + pos,
                                rem, position, lock);
        } else {
            written = nd.write(fd, ((DirectBuffer)bb).address() + pos, rem);
        }
        if (written > 0)
            bb.position(pos + written);
        return written;
    }
```

#### connect

connect() 方法用于向服务端发送连接请求

```
  public boolean connect(SocketAddress sa) throws IOException {
        int localPort = 0;

        synchronized (readLock) {
            synchronized (writeLock) {
                ensureOpenAndUnconnected();
                InetSocketAddress isa = Net.checkAddress(sa);
                SecurityManager sm = System.getSecurityManager();
                if (sm != null)
                    sm.checkConnect(isa.getAddress().getHostAddress(),
                                    isa.getPort());
                synchronized (blockingLock()) {
                    int n = 0;
                    try {
                        try {
                            begin();
                            synchronized (stateLock) {
                                if (!isOpen()) {
                                    return false;
                                }
                                if (localAddress == null) {
                                    NetHooks.beforeTcpConnect(fd,
                                                           isa.getAddress(),
                                                           isa.getPort());
                                }
                                readerThread = NativeThread.current();
                            }
                            for (;;) {
                                InetAddress ia = isa.getAddress();
                                if (ia.isAnyLocalAddress())
                                    ia = InetAddress.getLocalHost();
                                n = Net.connect(fd,ia,isa.getPort());
                                if (  (n == IOStatus.INTERRUPTED)
                                      && isOpen())
                                    continue;
                                break;
                            }

                            synchronized (stateLock) {
                                if (isOpen() && (localAddress == null) ||
                                    ((InetSocketAddress)localAddress)
                                        .getAddress().isAnyLocalAddress())
                                {
                                    localAddress = Net.localAddress(fd);
                                }
                            }

                        } finally {
                            readerCleanup();
                            end((n > 0) || (n == IOStatus.UNAVAILABLE));
                            assert IOStatus.check(n);
                        }
                    } catch (IOException x) {
                        close();
                        throw x;
                    }
                    synchronized (stateLock) {
                        remoteAddress = isa;
                        if (n > 0) {
                            state = ST_CONNECTED;
                            return true;
                        }
                        if (!isBlocking())
                            state = ST_PENDING;
                        else
                            assert false;
                    }
                }
                return false;
            }
        }
    }
```
finishConnect() 方法用于完成连接。  
对于阻塞请求，connect() 方法会等待直到客户端接收连接后返回。  
对于非阻塞请求，connect() 方法在客户端发送请求连接后即返回，所以需要调用finishConnect() 方法等待服务端接收连接。  

```
    public boolean finishConnect() throws IOException {
        synchronized (readLock) {
            synchronized (writeLock) {
                synchronized (stateLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    if (state == ST_CONNECTED)
                        return true;
                    if (state != ST_PENDING)
                        throw new NoConnectionPendingException();
                }
                int n = 0;
                try {
                    try {
                        begin();
                        synchronized (blockingLock()) {
                            synchronized (stateLock) {
                                if (!isOpen()) {
                                    return false;
                                }
                                readerThread = NativeThread.current();
                            }
                            if (!isBlocking()) {
                                for (;;) {
                                    n = checkConnect(fd, false,
                                                     readyToConnect);
                                    if (  (n == IOStatus.INTERRUPTED)
                                          && isOpen())
                                        continue;
                                    break;
                                }
                            } else {
                                for (;;) {
                                    n = checkConnect(fd, true,
                                                     readyToConnect);
                                    if (n == 0) {
                                        continue;
                                    }
                                    if (  (n == IOStatus.INTERRUPTED)
                                          && isOpen())
                                        continue;
                                    break;
                                }
                            }
                        }
                    } finally {
                        synchronized (stateLock) {
                            readerThread = 0;
                            if (state == ST_KILLPENDING) {
                                kill();
                                n = 0;
                            }
                        }
                        end((n > 0) || (n == IOStatus.UNAVAILABLE));
                        assert IOStatus.check(n);
                    }
                } catch (IOException x) {
                    close();
                    throw x;
                }
                if (n > 0) {
                    synchronized (stateLock) {
                        state = ST_CONNECTED;
                    }
                    return true;
                }
                return false;
            }
        }
    }
    
    //本地方法
    private static native int checkConnect(FileDescriptor fd,
                                           boolean block, boolean ready)
        throws IOException;

```
