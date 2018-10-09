---
title: GZIPInputStream and the available() method
---

# Introduction

Recently, I faced an issue proxying the download of a large GZip-compressed file. Because this proxy needs access to the data, the content had to be uncompressed and subsequently compressed for transmission back to the client.
The issue was that the downloaded file was never complete, and that the degree of completeness varied with every request. After a little investigation, I discovered that [java.util.zip.GZIPInputStream](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/zip/GZIPInputStream.html), which was being used for the decompression, appeared to be at fault.

<br>

## java.util.zip.GZIPInputStream
[GZIPInputStream](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/zip/GZIPInputStream.html) is a [FilterInputStream](https://docs.oracle.com/javase/8/docs/api/index.html?java/io/FilterInputStream.html) that decompresses the data as it's read.
It essentially wraps an existing InputStream, and provides the inflating behavior.

    InputStream compressedStream; // the actual data stream
	
	// The GZIPInputStream works directly with the actual data stream
    GZIPInputStream gzipIn = new GZIPInputStream(compressedStream);
	byte[] buffer = new byte[4096];
	while(EOF != (n = gzipIn.read(buffer))) {
		// handle the uncompressed data
	}

This is fairly straight-forward, and it works exactly as expected most of the time.

However, there are cases where [GZIPInputStream](https://docs.oracle.com/javase/8/docs/api/index.html?java/util/zip/GZIPInputStream.html) erroneously infers the end of the underlying InputStream, and it does so using the [InputStream#available()](https://docs.oracle.com/javase/8/docs/api/java/io/InputStream.html#available--) method. This method is documented to return

    "an estimate of the number of bytes that can be read (or skipped over) from this input stream without blocking or 0 when it reaches the end of the input stream."


Looking at the *GZIPInputStream* source, one finds the following:

    // If there are more bytes available in "in" or
    // the leftover in the "inf" is > 26 bytes:
    // this.trailer(8) + next.header.min(10) + next.trailer(8)
    // try concatenated case
    if (this.in.available() > 0 || n > 26) {
		

Here, we see that the implementation is looking for the case where multiple files are concatenated into a single GZip-compressed file (see [GZip Advanced Usage](http://www.gnu.org/software/gzip/manual/html_node/Advanced-usage.html) for details).
It's using the *available()* method to determine if there may be another GZip file in the stream (because the InputStream#available() documentation states that a _0_ result means the end of the stream has been reached). So far, so good.


<br>

## java.io.InputStream#available()

Apparently though, sometimes, *some* InputStream implementations return _0_ before the actual end of the stream (EOF), and GZIPInputStream interprets this to mean that there is no additional content, and treats the file as complete.
Because this _0_ result from the underlying InputStream is dependent on the contents of its buffer, it's an intermittent result; this causes the variable-length results from multiple attempts to download the same file.

This appears to be the result of a poor contract. Again, the documentation for *available()* states that it returns

    "an estimate of the number of bytes that can be read (or skipped over) from this input stream without blocking or 0 when it reaches the end of the input stream."

What if the stream *still* has content, but the number of bytes that can be read *without blocking* is actually __0__? Perhaps, the contract should require that the *available()* method return __-1__ if the stream is indeed empty.
Alas, this is not the case. So, what can be done? An OpenJDK bug [JDK-8081450](https://bugs.openjdk.java.net/browse/JDK-8081450) acknowledges this issue, but it remains unresolved since 2015. And what about the Oracle JDK, which has this same issue?


<br>


## Polymorphism to the Rescue

Thankfully, we can overcome this issue in our own code, thanks to interfaces and polymorphism.

We can create an InputStream implementation that *mostly* just delegates to another InputStream, but __never returns zero__ from the *available()* method.

	class GZIPInputStreamHelper extends InputStream {
    
	    private InputStream delegate;
    
	    GZIPInputStreamHelper(InputStream delegate) {
	      this.delegate = delegate;
	    }
    
	    /**
	     * @return The delegate's available() result if it's > 0, otherwise 1.
	     */
	    @Override
	    public int available() throws IOException {
	      int available = delegate.available();
	      if (available <= 1) {
	        available = 1;
	      }
	      return available;
	    }
		
        /////////
    	// The following are just the delegation implementation methods, but they are included for completeness
		/////////
		
	    @Override
	    public int read() throws IOException {
	      return delegate.read();
	    }
    
	    @Override
	    public int read(byte[] b) throws IOException {
	      return delegate.read(b);
	    }
    
	    @Override
	    public int read(byte[] b, int off, int len) throws IOException {
	      return delegate.read(b, off, len);
	    }
    
	    @Override
	    public long skip(long n) throws IOException {
	      return delegate.skip(n);
	    }
    
	    @Override
	    public void close() throws IOException {
	      delegate.close();
	    }
    
	    @Override
	    public synchronized void mark(int readlimit) {
	      if (markSupported()) {
	        delegate.mark(readlimit);
	      }
	    }
    
	    @Override
	    public synchronized void reset() throws IOException {
	      if (markSupported()) {
	        delegate.reset();
	      }
	    }
    
	    @Override
	    public boolean markSupported() {
	      return delegate.markSupported();
	    }
	  }


Modifying the earlier example by injecting this new InputStream implementation as a layer between the actual InputStream and the GZIPInputStream, resolves the issue.

    InputStream compressedStream; // the actual data stream
    
	// Wrap the actual input stream with the new implementation, and pass __that__ to the GZIPInputStream
    GZIPInputStream gzipIn = new GZIPInputStream(new GZIPInputStreamHelper(compressedStream));
	
	byte[] buffer = new byte[4096];
	while(EOF != (n = gzipIn.read(buffer))) {
		// handle the uncompressed data
	}
    

By never returning zero from the *available()* method, GZIPInputStream will always try to read from the stream until it reaches the *actual* end of the stream (i.e., EOF).

Voila! No more intermittently incomplete results!

<br>


# Summary

First of all, without access to the GZIPInputStream source code, it would have been much more difficult (if not impossible) for me to figure out what was causing this issue. I probably would have tossed the GZIPInputStream aside, and written a complete replacement.
So, +1 for the sharing of source code :-) "ipsa scientia potestas est"

This problem and it's resolution highlights the importance of interface contracts; both the definition thereof, and the adherence thereto. Interface contracts should be clear to consumers and implementers alike, and certainly not contradictory in their descriptions.
In this case, the InputStream#available() contract includes a subtle contradiction. A stream for which there are no bytes to be read *without blocking*, but which still has data, is in a "catch 22"; the stream is __not empty__, but there really are __zero__ bytes available for reading __without blocking__. The result of a method cannot be allowed to have two conflicting meanings.

The resolution also highlights to importance and power of employing interfaces in APIs and frameworks. Because the GZIPInputStream implementation depends only on InputStream, and not some specific implementation, it was easy to replace that underlying InputStream with a custom implementation. I could have alternatively extended the InputStream class, implementing the necessary methods, but I didn't want to change any of the behavior of the actual InputStream implementation __except__ for the *available()* method. Interfaces and polymorphism are our friends.

Finally, I spent some time trying to figure this out. Hopefully, someone else will benefit from this information without duplicating my effort.

<br><br><br><br>


