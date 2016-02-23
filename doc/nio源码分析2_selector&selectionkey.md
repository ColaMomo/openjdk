# nio源码分析 - Selector, SelectionKey

## Selector 

selector原理流程图如下：

![selector原理](http://7xr5t9.com1.z0.glb.clouddn.com/java_nio_selector_flow.jpg)

```
java.nio.channels.Selector.java

public abstract class Selector implements Closeable {

    protected Selector() { }

    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

    public abstract boolean isOpen();

    public abstract SelectorProvider provider();

    //获取selector包含的全部SelectionKey
    public abstract Set<SelectionKey> keys();

	//获取Selector已经发生事件的SelectionKey
    public abstract Set<SelectionKey> selectedKeys();

    public abstract int selectNow() throws IOException;

    public abstract int select(long timeout)
        throws IOException;

	//轮询，等待事件发生
    public abstract int select() throws IOException;

    public abstract Selector wakeup();

    public abstract void close() throws IOException;
}

```

属性

```
    //selector上已经发生事件的SelectionKey
    protected Set<SelectionKey> selectedKeys;

    //selector所包含的全部SelectionKey
    protected HashSet<SelectionKey> keys;

    private Set<SelectionKey> publicKeys;             // Immutable
    private Set<SelectionKey> publicSelectedKeys;     // Removal allowed, but not addition
```

#### open

```
    protected SelectorImpl(SelectorProvider sp) {
        super(sp);
        keys = new HashSet<SelectionKey>();
        selectedKeys = new HashSet<SelectionKey>();
        if (Util.atBugLevel("1.4")) {
            publicKeys = keys;
            publicSelectedKeys = selectedKeys;
        } else {
            publicKeys = Collections.unmodifiableSet(keys);
            publicSelectedKeys = Util.ungrowableSet(selectedKeys);
        }
    }
```

```
	//SelectorProvider.java
    public static Selector open() throws IOException {
        return SelectorProvider.provider().openSelector();
    }

	//PollSelectorProvider.java
    public AbstractSelector openSelector() throws IOException {
        return new PollSelectorImpl(this);
    }

	//PollSelectorImpl.java
	PollSelectorImpl(SelectorProvider sp) {
		super(sp, 1, 1);
		long pipeFds = IOUtil.makePipe(false);
		fd0 = (int) (pipeFds >>> 32);
		fd1 = (int) pipeFds;
		pollWrapper = new PollArrayWrapper(INIT_CAP);
		pollWrapper.initInterrupt(fd0, fd1);
		channelArray = new SelectionKeyImpl[INIT_CAP];
	}
```

在PollSelectorImpl的构造方法中，创建了一对管道，这对管道是用于唤醒阻塞的select() 方法的

initInterrupt() 将管道fd加入select的监听fd中。 

```
//PollArrayWrapper.java

    void initInterrupt(int fd0, int fd1) {
        this.interruptFD = fd1;
        this.putDescriptor(0, fd0);
        this.putEventOps(0, Net.POLLIN);
        this.putReventOps(0, 0);
    }
```



#### register

```
//AbstractSelectableChannel.java

    public final SelectionKey register(Selector sel, int ops,
                                       Object att)
        throws ClosedChannelException
    {
        synchronized (regLock) {
            if (!isOpen())
                throw new ClosedChannelException();
            if ((ops & ~validOps()) != 0)
                throw new IllegalArgumentException();
            if (blocking)
                throw new IllegalBlockingModeException();
            SelectionKey k = findKey(sel);
            if (k != null) {
                k.interestOps(ops);
                k.attach(att);
            }
            if (k == null) {
                // New registration
                synchronized (keyLock) {
                    if (!isOpen())
                        throw new ClosedChannelException();
                    k = ((AbstractSelector)sel).register(this, ops, att);
                    addKey(k);
                }
            }
            return k;
        }
    }
```

```
//SelectorImpl.java

    protected final SelectionKey register(AbstractSelectableChannel ch,
                                          int ops,
                                          Object attachment)
    {
        if (!(ch instanceof SelChImpl))
            throw new IllegalSelectorException();
        SelectionKeyImpl k = new SelectionKeyImpl((SelChImpl)ch, this);
        k.attach(attachment);
        synchronized (publicKeys) {
            implRegister(k);
        }
        k.interestOps(ops);
        return k;
    }
```

```
//AbstractPollSelectorImpl.java

    protected void implRegister(SelectionKeyImpl ski) {
        synchronized (closeLock) {
            if (closed)
                throw new ClosedSelectorException();

            if (channelArray.length == totalChannels) {
                int newSize = pollWrapper.totalChannels * 2;
                SelectionKeyImpl temp[] = new SelectionKeyImpl[newSize];
                for (int i=channelOffset; i<totalChannels; i++)
                    temp[i] = channelArray[i];
                channelArray = temp;
                pollWrapper.grow(newSize);
            }
            channelArray[totalChannels] = ski;
            ski.setIndex(totalChannels);
            pollWrapper.addEntry(ski.channel);
            totalChannels++;
            keys.add(ski);
        }
    }
```

addEntry() 将fd加入监听数组中。 

```
//PollArrayWrapper.java

    void addEntry(SelChImpl channel) {
        this.putDescriptor(this.totalChannels, IOUtil.fdVal(channel.getFD()));
        this.putEventOps(this.totalChannels, 0);
        this.putReventOps(this.totalChannels, 0);
        ++this.totalChannels;
    }
```

#### select

select() 方法等待就绪事件的发生

```
    public int select(long timeout)
        throws IOException
    {
        if (timeout < 0)
            throw new IllegalArgumentException("Negative timeout");
        return lockAndDoSelect((timeout == 0) ? -1 : timeout);
    }

    public int select() throws IOException {
        return select(0);
    }

	//非阻塞调用
    public int selectNow() throws IOException {
        return lockAndDoSelect(0);
    }

    private int lockAndDoSelect(long timeout) throws IOException {
        synchronized (this) {
            if (!isOpen())
                throw new ClosedSelectorException();
            synchronized (publicKeys) {
                synchronized (publicSelectedKeys) {
                    return doSelect(timeout);
                }
            }
        }
    }
```

```
sun.nio.ch.PollSelectorImpl.java

	protected int doSelect(long timeout) throws IOException
	{
		if (channelArray == null)
			throw new ClosedSelectorException();
   		processDeregisterQueue();
        try {
            begin();
            pollWrapper.poll(totalChannels, 0, timeout);
        } finally {
            end();
        }
        processDeregisterQueue();
        int numKeysUpdated = updateSelectedKeys();
        if (pollWrapper.getReventOps(0) != 0) {
            pollWrapper.putReventOps(0, 0);
            synchronized (interruptLock) {
                IOUtil.drain(fd0);
                interruptTriggered = false;
            }
        }
        return numKeysUpdated;
   }

```

processDeregisterQueue方法主要是对已取消的键集合进行处理，通过调用cancel()方法将选择键加入已取消的键集合中，这个键并不会立即注销，而是在下一次select操作时进行注销，注销操作在implDereg完成

```
    void processDeregisterQueue() throws IOException {
        Set cks = cancelledKeys();
        synchronized (cks) {
            if (!cks.isEmpty()) {
                Iterator i = cks.iterator();
                while (i.hasNext()) {
                    SelectionKeyImpl ski = (SelectionKeyImpl)i.next();
                    try {
                        implDereg(ski);
                    } catch (SocketException se) {
                        IOException ioe = new IOException(
                            "Error deregistering key");
                        ioe.initCause(se);
                        throw ioe;
                    } finally {
                        i.remove();
                    }
                }
            }
        }
    }
```

```
AbstractPollSelectorImpl.java

    protected void implDereg(SelectionKeyImpl ski) throws IOException {
        int i = ski.getIndex();
        assert (i >= 0);
        if (i != totalChannels - 1) {
            SelectionKeyImpl endChannel = channelArray[totalChannels-1];
            channelArray[i] = endChannel;
            endChannel.setIndex(i);
            pollWrapper.release(i);
            PollArrayWrapper.replaceEntry(pollWrapper, totalChannels - 1,
                                          pollWrapper, i);
        } else {
            pollWrapper.release(i);
        }
        channelArray[totalChannels-1] = null;
        totalChannels--;
        pollWrapper.totalChannels--;
        ski.setIndex(-1);
        keys.remove(ski);
        selectedKeys.remove(ski);
        deregister((AbstractSelectionKey)ski);
        SelectableChannel selch = ski.channel();
        if (!selch.isOpen() && !selch.isRegistered())
            ((SelChImpl)selch).kill();
    }
```

updateSelectedKeys() 将有事件发生的SelectionKey添加到SelectedKeys集合中

```
AbstractPollSelectorImpl.java

    protected int updateSelectedKeys() {
        int numKeysUpdated = 0;
        for (int i=channelOffset; i<totalChannels; i++) {
            int rOps = pollWrapper.getReventOps(i);
            if (rOps != 0) {
                SelectionKeyImpl sk = channelArray[i];
                pollWrapper.putReventOps(i, 0);
                if (selectedKeys.contains(sk)) {
                    if (sk.channel.translateAndSetReadyOps(rOps, sk)) {
                        numKeysUpdated++;
                    }
                } else {
                    sk.channel.translateAndSetReadyOps(rOps, sk);
                    if ((sk.nioReadyOps() & sk.nioInterestOps()) != 0) {
                        selectedKeys.add(sk);
                        numKeysUpdated++;
                    }
                }
            }
        }
        return numKeysUpdated;
    }
```

具体的底层io多路复用在PollArrayWrapper中的poll() 方法中调用本地方法进行实现。

```
PollArrayWrapper.java

private int poll() throws IOException{  
            return poll0(pollWrapper.pollArrayAddress,  
                         Math.min(totalChannels, MAX_SELECTABLE_FDS),  
                         readFds, writeFds, exceptFds, timeout);  
        }  
private native int poll0(long pollAddress, int numfds,  
             int[] readFds, int[] writeFds, int[] exceptFds, long timeout);  
```

#### wakeup

wakeup() 方法用于唤醒阻塞的select调用。

具体方法是，在初始化selector的时候，会新建一对管道或socket，并将fd加入select的监听fd数组中，在调用wakeup()方法时，向管道或socket中发送数据，这样select就会监听到该事件，进而从阻塞态返回。

```
	//PollSelectorImpl.java
	public Selector wakeup() {
		synchronized (interruptLock) {
			if (!interruptTriggered) {
				pollWrapper.interrupt();
				interruptTriggered = true;
			}
		}
		return this;
	}
	
	//PollArrayWrapper.java
    public void interrupt() {
        interrupt(this.interruptFD);
    }
    
    private static native void interrupt(int fd);

```


## SelectionKey

```
package java.nio.channels;

public abstract class SelectionKey {

    protected SelectionKey() { }

    public abstract SelectableChannel channel();

    public abstract Selector selector();

    public abstract boolean isValid();

    public abstract void cancel();

    public abstract int interestOps();

    public abstract SelectionKey interestOps(int ops);

    public abstract int readyOps();

    public static final int OP_READ = 1 << 0;
    public static final int OP_WRITE = 1 << 2;
    public static final int OP_CONNECT = 1 << 3;
    public static final int OP_ACCEPT = 1 << 4;

    public final boolean isReadable() {
        return (readyOps() & OP_READ) != 0;
    }

    public final boolean isWritable() {
        return (readyOps() & OP_WRITE) != 0;
    }

    public final boolean isConnectable() {
        return (readyOps() & OP_CONNECT) != 0;
    }

    public final boolean isAcceptable() {
        return (readyOps() & OP_ACCEPT) != 0;
    }

    private volatile Object attachment = null;

    private static final AtomicReferenceFieldUpdater<SelectionKey,Object>
        attachmentUpdater = AtomicReferenceFieldUpdater.newUpdater(
            SelectionKey.class, Object.class, "attachment"
        );

    public final Object attach(Object ob) {
        return attachmentUpdater.getAndSet(this, ob);
    }

    public final Object attachment() {
        return attachment;
    }

}
```
