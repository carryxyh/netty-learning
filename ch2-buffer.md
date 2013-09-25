Netty那点事（二）-Netty中的buffer
--------
>尚未完成

上一篇文章我们概要介绍了Netty的原理及结构，下面几篇文章我们开始对Netty的各个模块进行比较详细的分析。Netty的结构最底层是Buffer机制，这部分也相对独立，我们就先从Buffer讲起。

## What：Buffer二三事

Buffer中文名又叫缓冲区，

NIO中的ByteBuffer。

关于ByteBuffer的吐槽：[http://www.javacodegeeks.com/2012/12/the-java-bytebuffer-a-crash-course.html](http://www.javacodegeeks.com/2012/12/the-java-bytebuffer-a-crash-course.html)

## How：Netty中的ChannelBuffer及实现

ChannelBuffers是所有ChannelBuffer实现类的入口，它提供了很多静态的工具方法来创建不同的Buffer，靠“顺藤摸瓜”式读代码方式，大致能把各种ChannelBuffer的实现类摸个遍。

![channel buffer in Netty][1]


### ChannelBuffer中的readerIndex和writerIndex

开始以为Netty的ChannelBuffer是对NIO ByteBuffer的一个封装，其实不是的，**它是把ByteBuffer重新实现了一遍**。

以最常用的`HeapChannelBuffer`为例，其底层也是一个byte[]，与ByteBuffer不同的是，它是可以同时进行读和写的，而不需要使用flip()进行读写切换。ChannelBuffer读写的核心代码在`AbstactChannelBuffer`里，这里通过readerIndex和writerIndex两个整数，分别指向当前读的位置和当前写的位置，并且，总是等于writerIndex - readerIndex。贴两段代码，让大家能看的更明白一点：

```java
    public void writeByte(int value) {
        setByte(writerIndex ++, value);
    }

    public byte readByte() {
        if (readerIndex == writerIndex) {
            throw new IndexOutOfBoundsException("Readable byte limit exceeded: "
                    + readerIndex);
        }
        return getByte(readerIndex ++);
    }

    public int writableBytes() {
        return capacity() - writerIndex;
    }
    
    public int readableBytes() {
        return writerIndex - readerIndex;
    }
```

我倒是觉得这样的方式非常自然，比单指针与flip()要更加好理解一些。AbstactChannelBuffer还有两个相应的mark指针`markedReaderIndex`和`markedWriterIndex`，跟NIO的原理是一样的，这里不再赘述了。

### 字节序Endianness

在创建Buffer时，我们注意到了这样一个方法：`public static ChannelBuffer buffer(ByteOrder endianness, int capacity);`，其中`ByteOrder`是什么意思呢？

这里有个很基础的概念：字节序(ByteOrder/Endianness)。它规定了多余一个字节的数字(int啊long什么的)，如何在内存中表示。BIG_ENDIAN(大端序)表示高位在前，整型数`12`会被存储为`0 0 0 12`四字节，而LITTLE_ENDIAN则正好相反。可能搞C/C++的程序员对这个会比较熟悉，而Javaer则比较陌生一点，因为Java已经把内存给管理好了。但是在网络编程方面，根据协议的不同，不同的字节序也可能会被用到。目前大部分协议还是采用大端序，可参考[RFC1700](http://tools.ietf.org/html/rfc1700)。

了解了这些知识，我们也很容易就知道为什么会有`BigEndianHeapChannelBuffer`和`LittleEndianHeapChannelBuffer`了！

### DynamicChannelBuffer

DynamicChannelBuffer是一个很方便的Buffer，之所以叫Dynamic是因为它的长度会根据内容的长度来扩充，你可以像使用ArrayList一样，无须关心其容量。实现自动扩容的核心在于`ensureWritableBytes`方法，算法很简单：在写入前做容量检查，容量不够时，新建一个容量x2的buffer，跟ArrayList的扩容是相同的。贴一段代码吧(为了代码易懂，这里我删掉了一些边界检查，只保留主逻辑)：

```java
    public void writeByte(int value) {
        ensureWritableBytes(1);
        super.writeByte(value);
    }

    public void ensureWritableBytes(int minWritableBytes) {
        if (minWritableBytes <= writableBytes()) {
            return;
        }

        int newCapacity = capacity();
        int minNewCapacity = writerIndex() + minWritableBytes;
        while (newCapacity < minNewCapacity) {
            newCapacity <<= 1;
        }

        ChannelBuffer newBuffer = factory().getBuffer(order(), newCapacity);
        newBuffer.writeBytes(buffer, 0, writerIndex());
        buffer = newBuffer;
    }
```

### CompositeChannelBuffer

`CompositeChannelBuffer`是由多个ChannelBuffer组合而成的，可以看做一个整体进行读写。这里有一个技巧：CompositeChannelBuffer并不会开辟新的内存并直接复制所有ChannelBuffer内容，而是直接保存了所有ChannelBuffer的引用，并在子ChannelBuffer进行读写。这也就是Netty官方所说的"Zero-Copy-Capable"了。来段简略版的代码吧：

```java
	public class CompositeChannelBuffer{

	    //components保存所有内部ChannelBuffer
	    private ChannelBuffer[] components;
	    //indices记录在整个CompositeChannelBuffer中，每个components的起始位置
	    private int[] indices;
	    //缓存上一次读写的componentId
	    private int lastAccessedComponentId;

	    public byte getByte(int index) {
	        //通过indices中记录的位置索引到对应第几个子Buffer
	        int componentId = componentId(index);
	        return components[componentId].getByte(index - indices[componentId]);
	    }

	}	
```

查找componentId的算法再次不作介绍了，大家自己实现起来也不会太难。值得一提的是，基于ChannelBuffer连续读写的特性，这里使用了顺序查找，并且用`lastAccessedComponentId`来进行缓存。

### WrappedChannelBuffer

![virtual buffer in Netty][2]


writerIndex

readerIndex

HeapBuffer

DirectBuffer

trivial

CompositeChannelBuffer gathering

## When & Where：TCP/IP协议与Buffer

MTU与Nagle算法

## Why：代码背后的设计思想

关于Zero-Copy-Capable，我觉得理解为什么需要"Zero-Copy-Capable Rich Byte Buffer"，比理解它是怎么实现的，可能要更花心思一点。

![virtual buffer in Netty][3]

所以，**如果说NIO的Buffer和Netty的ChannelBuffer最大的区别的话，就是前者仅仅是物理上的Buffer，而后者是物理Buffer和抽象后的逻辑Buffer的结合。**我觉得这才是关于Netty的ChannelBuffer最需要理解的。

  [1]: http://static.oschina.net/uploads/space/2013/0925/081551_v8pK_190591.png
  [2]: http://static.oschina.net/uploads/space/2013/0925/074748_oSkl_190591.png
  [3]: http://netty.io/3.7/guide/images/combine-slice-buffer.png
  

参考资料：

* Endianness [http://en.wikipedia.org/wiki/Endianness](http://en.wikipedia.org/wiki/Endianness)