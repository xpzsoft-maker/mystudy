#网关日志记录——截取HttpServletResponse的OutputStream流内容

##1. ServletOutputStream
HttpServletResponse的输出流类型为ServletOutputStream，且只能被读取一次：
```java
response.getOutputStream(); //只能使用一次
```
ServletOutputStream源码如下：
```java
//
// Source code recreated from a .class file by IntelliJ IDEA
// (powered by Fernflower decompiler)
//

package javax.servlet;

import java.io.CharConversionException;
import java.io.IOException;
import java.io.OutputStream;
import java.text.MessageFormat;
import java.util.ResourceBundle;

public abstract class ServletOutputStream extends OutputStream {
    private static final String LSTRING_FILE = "javax.servlet.LocalStrings";
    private static final ResourceBundle lStrings = ResourceBundle.getBundle("javax.servlet.LocalStrings");

    protected ServletOutputStream() {
    }

    /**
    * 需要重写
    * 这是输出文本的关键函数
    */
    public void print(String s) throws IOException {
        if (s == null) {
            s = "null";
        }

        int len = s.length();
        byte[] buffer = new byte[len];

        for(int i = 0; i < len; ++i) {
            char c = s.charAt(i);
            if ((c & '\uff00') != 0) {
                String errMsg = lStrings.getString("err.not_iso8859_1");
                Object[] errArgs = new Object[]{c};
                errMsg = MessageFormat.format(errMsg, errArgs);
                throw new CharConversionException(errMsg);
            }

            buffer[i] = (byte)(c & 255);
        }

        this.write(buffer);
    }

    public void print(boolean b) throws IOException {
        String msg;
        if (b) {
            msg = lStrings.getString("value.true");
        } else {
            msg = lStrings.getString("value.false");
        }

        this.print(msg);
    }

    public void print(char c) throws IOException {
        this.print(String.valueOf(c));
    }

    public void print(int i) throws IOException {
        this.print(String.valueOf(i));
    }

    public void print(long l) throws IOException {
        this.print(String.valueOf(l));
    }

    public void print(float f) throws IOException {
        this.print(String.valueOf(f));
    }

    public void print(double d) throws IOException {
        this.print(String.valueOf(d));
    }

    public void println() throws IOException {
        this.print("\r\n");
    }

    public void println(String s) throws IOException {
        StringBuilder sb = new StringBuilder();
        sb.append(s);
        sb.append("\r\n");
        this.print(sb.toString());
    }

    public void println(boolean b) throws IOException {
        StringBuilder sb = new StringBuilder();
        if (b) {
            sb.append(lStrings.getString("value.true"));
        } else {
            sb.append(lStrings.getString("value.false"));
        }

        sb.append("\r\n");
        this.print(sb.toString());
    }

    public void println(char c) throws IOException {
        this.println(String.valueOf(c));
    }

    public void println(int i) throws IOException {
        this.println(String.valueOf(i));
    }

    public void println(long l) throws IOException {
        this.println(String.valueOf(l));
    }

    public void println(float f) throws IOException {
        this.println(String.valueOf(f));
    }

    public void println(double d) throws IOException {
        this.println(String.valueOf(d));
    }

     /**
    * 需要重写
    */
    public abstract boolean isReady();

    /**
    * 需要重写
    */
    public abstract void setWriteListener(WriteListener var1);
}

```
从源码中可以看出，ServletOutputStream的核心实现是print(String)，所有内容都将转换为字符串，然后输出。
此外，ServletOutputStream继承OutputStream，print(String)最后调用的是OutputStream的write(byte b[])，
OutputStream的源码如下：
```java
/*
 * Copyright (c) 1994, 2004, Oracle and/or its affiliates. All rights reserved.
 * ORACLE PROPRIETARY/CONFIDENTIAL. Use is subject to license terms.
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 *
 */

package java.io;

/**
 * This abstract class is the superclass of all classes representing
 * an output stream of bytes. An output stream accepts output bytes
 * and sends them to some sink.
 * <p>
 * Applications that need to define a subclass of
 * <code>OutputStream</code> must always provide at least a method
 * that writes one byte of output.
 *
 * @author  Arthur van Hoff
 * @see     java.io.BufferedOutputStream
 * @see     java.io.ByteArrayOutputStream
 * @see     java.io.DataOutputStream
 * @see     java.io.FilterOutputStream
 * @see     java.io.InputStream
 * @see     java.io.OutputStream#write(int)
 * @since   JDK1.0
 */
public abstract class OutputStream implements Closeable, Flushable {
    /**
     * Writes the specified byte to this output stream. The general
     * contract for <code>write</code> is that one byte is written
     * to the output stream. The byte to be written is the eight
     * low-order bits of the argument <code>b</code>. The 24
     * high-order bits of <code>b</code> are ignored.
     * <p>
     * Subclasses of <code>OutputStream</code> must provide an
     * implementation for this method.
     *
     * @param      b   the <code>byte</code>.
     * @exception  IOException  if an I/O error occurs. In particular,
     *             an <code>IOException</code> may be thrown if the
     *             output stream has been closed.
     */
    public abstract void write(int b) throws IOException;

    /**
     * Writes <code>b.length</code> bytes from the specified byte array
     * to this output stream. The general contract for <code>write(b)</code>
     * is that it should have exactly the same effect as the call
     * <code>write(b, 0, b.length)</code>.
     *
     * @param      b   the data.
     * @exception  IOException  if an I/O error occurs.
     * @see        java.io.OutputStream#write(byte[], int, int)
     */
    public void write(byte b[]) throws IOException {
        write(b, 0, b.length);
    }

    /**
     * Writes <code>len</code> bytes from the specified byte array
     * starting at offset <code>off</code> to this output stream.
     * The general contract for <code>write(b, off, len)</code> is that
     * some of the bytes in the array <code>b</code> are written to the
     * output stream in order; element <code>b[off]</code> is the first
     * byte written and <code>b[off+len-1]</code> is the last byte written
     * by this operation.
     * <p>
     * The <code>write</code> method of <code>OutputStream</code> calls
     * the write method of one argument on each of the bytes to be
     * written out. Subclasses are encouraged to override this method and
     * provide a more efficient implementation.
     * <p>
     * If <code>b</code> is <code>null</code>, a
     * <code>NullPointerException</code> is thrown.
     * <p>
     * If <code>off</code> is negative, or <code>len</code> is negative, or
     * <code>off+len</code> is greater than the length of the array
     * <code>b</code>, then an <tt>IndexOutOfBoundsException</tt> is thrown.
     *
     * @param      b     the data.
     * @param      off   the start offset in the data.
     * @param      len   the number of bytes to write.
     * @exception  IOException  if an I/O error occurs. In particular,
     *             an <code>IOException</code> is thrown if the output
     *             stream is closed.
     */
    /**
    * 需要重写
    */
    public void write(byte b[], int off, int len) throws IOException {
        if (b == null) {
            throw new NullPointerException();
        } else if ((off < 0) || (off > b.length) || (len < 0) ||
                   ((off + len) > b.length) || ((off + len) < 0)) {
            throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        for (int i = 0 ; i < len ; i++) {
            write(b[off + i]);
        }
    }

    /**
     * Flushes this output stream and forces any buffered output bytes
     * to be written out. The general contract of <code>flush</code> is
     * that calling it is an indication that, if any bytes previously
     * written have been buffered by the implementation of the output
     * stream, such bytes should immediately be written to their
     * intended destination.
     * <p>
     * If the intended destination of this stream is an abstraction provided by
     * the underlying operating system, for example a file, then flushing the
     * stream guarantees only that bytes previously written to the stream are
     * passed to the operating system for writing; it does not guarantee that
     * they are actually written to a physical device such as a disk drive.
     * <p>
     * The <code>flush</code> method of <code>OutputStream</code> does nothing.
     *
     * @exception  IOException  if an I/O error occurs.
     */
    /**
    * 需要重写
    */
    public void flush() throws IOException {
    }

    /**
     * Closes this output stream and releases any system resources
     * associated with this stream. The general contract of <code>close</code>
     * is that it closes the output stream. A closed stream cannot perform
     * output operations and cannot be reopened.
     * <p>
     * The <code>close</code> method of <code>OutputStream</code> does nothing.
     *
     * @exception  IOException  if an I/O error occurs.
     */
    /**
    * 需要重写
    */
    public void close() throws IOException {
    }

}

```
OutputStream实现了Closeable, Flushable两个接口，核心实现是write(byte b[], int off, int len)，而且要注意，
<b style="color: red">flush()与close()没有具体实现</b>，分析得出，要截取HttpServletResponse的输出流，需要重写ServletOutputStream，
重写ServletOutputStream的关键点在于记录print(String)与write(byte b[], int off, int len)内容。

##2. 重写ServletOutputStream
因为ServletOutputStream的flush()与close()没有具体实现，且ServletOutputStream包含虚函数，为了避免重写错误，
我们将重用filter的ServletResponse.getOutputStream()对象。具体实现如下：
```java
public static class GwServletOutputStream extends ServletOutputStream {

        /**
        * 重用对象
        */
        private ServletOutputStream out;
        /**
        * 记录输出内容
        */
        private StringBuilder stringBuilder = new StringBuilder();

        /**
        * 重用ServletResponse.getOutputStream()
        */
        GwServletOutputStream(ServletOutputStream out) { 
            this.out = out;
        }

        @Override
        public void print(String s) throws IOException {
            stringBuilder.append(s); // 在此处记录输出内容
            this.out.print(s); // 重用输出流对象的print
        }

        @Override
        public void write(@NonNull byte [] b, int off, int len) throws IOException {
            stringBuilder.append(new String(b)); // 在此处记录输出内容
            this.out.write(b, off, len); // 重用输出流对象的write
        }

        @Override
        public void flush() throws IOException {
            this.out.flush(); // 重用输出流对象的flush
        }

        @Override
        public void close() throws IOException {
            this.out.close(); // 重用输出流对象的close
        }

        @Override
        public boolean isReady() {
            return this.out.isReady(); // 重用输出流对象的isready
        }

        @Override
        public void setWriteListener(WriteListener writeListener) {
            this.out.setWriteListener(writeListener); // 重用输出流对象的setWriteListener
        }

        @Override
        public void write(int b) throws IOException {
            stringBuilder.append(new String(b)); // 在此处记录输出内容
            this.out.write(b); // 重用输出流对象的close
        }

        String getResponseMsg() {
            return stringBuilder.toString();
        }
    }
```
<b style="color: red">网关的最外层的Filter截取输出流内容，形成日志记录</b>
```java
/**
* 最外层的filter
*/
public class MostOutFilter extends Filter{
    @Override
    public void doFilter(ServletRequest req, ServletResponse res, FilterChain chain) throws IOException, ServletException {
        
        // 在启用res.getOutputStream()之前获取其对象，只能用1次
        GwServletOutputStream gwServletOutputStream = new GwServletOutputStream(res.getOutputStream());

        HttpServletResponse response = new HttpServletResponseWrapper((HttpServletResponse)res) {
            /**
            * 返回重写后的输出流
            */
            @Override
            public ServletOutputStream getOutputStream() {
                return gwServletOutputStream;
            }
        };

        chain.doFilter(req, response);

        gwServletOutputStream.getResponseMsg(); // 获取输出流的数据
    }
}
```
##3. 疑问
ServletOutputStream不能被实例化，那么Tomcat中的HttpServletResponse的getOutputStream对象的实现类是哪个？找到该类，如果其不是fianl，重写该类
比重写ServletOutputStream更安全，毕竟不知道该类中是否有自己的实现方法并被其它拦截器或者过滤器调用。待以后解决。。。
<b style="color: red">经过实测，该方法在SpringSecurity的filter机制中是安全的。</b>