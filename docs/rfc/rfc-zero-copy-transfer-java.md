# RFC: Using Zero-Copy Transfer in Java for High-Throughput Applications

**Type:** RFC
**Status:** IN_PROGRESS
**Created:** 2026-02-17T10:22:37.291Z
**Last Updated:** 2026-02-17T10:22:37.291Z

---

<h1>RFC: Using Zero-Copy Transfer in Java for High-Throughput Applications</h1>

<h2>Summary</h2>
<p>This RFC proposes implementing zero-copy transfer techniques in our Java-based file serving application to significantly reduce CPU overhead and improve throughput. By leveraging the <code>transferTo()</code> method in Java NIO, we can transfer data directly from file buffers to socket buffers in the Linux kernel, bypassing unnecessary data copying through user space.</p>

<h2>Status</h2>
<p><strong>Proposed</strong> - Awaiting team review and sign-off</p>

<h2>Motivation</h2>
<p>Our current file serving implementation suffers from high CPU utilization when handling large file transfers. Profiling has revealed that a significant portion of CPU time is spent copying data between kernel space and user space, then back to kernel space for network transmission. This overhead becomes particularly pronounced under high load conditions.</p>

<h3>Performance Issues</h3>
<ul>
<li>CPU utilization reaches 85% during peak file transfer loads</li>
<li>Current throughput: ~500 MB/s per server instance</li>
<li>Memory bandwidth becomes a bottleneck at scale</li>
<li>Increased garbage collection pressure from temporary buffers</li>
</ul>

<h2>Background: Traditional vs Zero-Copy Transfer</h2>

<h3>Traditional File Transfer Flow</h3>
<p>In a traditional file transfer scenario, data makes multiple trips between kernel and user space:</p>

<pre><code class="language-java">// Traditional approach - multiple data copies
FileInputStream fis = new FileInputStream("largefile.dat");
byte[] buffer = new byte[8192];
int bytesRead;

while ((bytesRead = fis.read(buffer)) != -1) {
    outputStream.write(buffer, 0, bytesRead);
}
fis.close();
</code></pre>

<p>This approach involves 4 data copies:</p>
<ol>
<li>Disk → Kernel buffer (DMA)</li>
<li>Kernel buffer → User space buffer (CPU copy)</li>
<li>User space buffer → Socket buffer (CPU copy)</li>
<li>Socket buffer → NIC (DMA)</li>
</ol>

<h3>Zero-Copy Transfer Architecture</h3>
<p>Zero-copy transfer eliminates the intermediate copies through user space by using the <code>sendfile()</code> system call on Linux:</p>

<div data-type="mermaid">
<pre>sequenceDiagram
    participant App as Application
    participant Kernel as Kernel Space
    participant DMA as DMA Engine
    participant Disk as Disk
    participant NIC as Network Card
    
    App->>Kernel: transferTo() / sendfile()
    Kernel->>DMA: Request file data
    DMA->>Disk: Read file
    Disk-->>DMA: File data
    DMA->>Kernel: File buffer
    Note over Kernel: Zero-copy transfer
    Kernel->>DMA: Send to socket
    DMA->>NIC: Network transmission
    NIC-->>App: Transfer complete
</pre>
</div>

<h2>Proposed Solution</h2>

<h3>Implementation with FileChannel.transferTo()</h3>
<p>Java NIO provides the <code>FileChannel.transferTo()</code> method which maps directly to the <code>sendfile()</code> system call on Linux:</p>

<pre><code class="language-java">import java.io.FileInputStream;
import java.io.IOException;
import java.nio.channels.FileChannel;
import java.nio.channels.SocketChannel;
import java.net.InetSocketAddress;

public class ZeroCopyFileServer {
    private static final int PORT = 8080;
    private static final String FILE_PATH = "/data/largefile.dat";
    
    public void transferFile(SocketChannel socketChannel) throws IOException {
        // Open file channel for reading
        try (FileInputStream fis = new FileInputStream(FILE_PATH);
             FileChannel fileChannel = fis.getChannel()) {
            
            long position = 0;
            long bytesTransferred = 0;
            long fileSize = fileChannel.size();
            
            // Transfer file using zero-copy
            while (bytesTransferred < fileSize) {
                long count = fileChannel.transferTo(
                    position,           // Starting position in file
                    fileSize - position, // Number of bytes to transfer
                    socketChannel       // Target channel
                );
                
                if (count > 0) {
                    position += count;
                    bytesTransferred += count;
                } else {
                    break; // Transfer complete or error
                }
            }
            
            System.out.println("Transferred " + bytesTransferred + " bytes");
        }
    }
}
</code></pre>

<h3>Complete Server Implementation</h3>
<pre><code class="language-java">import java.io.IOException;
import java.net.InetSocketAddress;
import java.nio.channels.ServerSocketChannel;
import java.nio.channels.SocketChannel;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

public class HighThroughputFileServer {
    private final ExecutorService executorService;
    private final int port;
    private volatile boolean running = true;
    
    public HighThroughputFileServer(int port, int threadPoolSize) {
        this.port = port;
        this.executorService = Executors.newFixedThreadPool(threadPoolSize);
    }
    
    public void start() throws IOException {
        try (ServerSocketChannel serverChannel = ServerSocketChannel.open()) {
            serverChannel.bind(new InetSocketAddress(port));
            System.out.println("Zero-copy file server started on port " + port);
            
            while (running) {
                SocketChannel clientChannel = serverChannel.accept();
                executorService.submit(() -> handleClient(clientChannel));
            }
        }
    }
    
    private void handleClient(SocketChannel clientChannel) {
        try {
            ZeroCopyFileServer fileServer = new ZeroCopyFileServer();
            fileServer.transferFile(clientChannel);
        } catch (IOException e) {
            System.err.println("Error handling client: " + e.getMessage());
        } finally {
            try {
                clientChannel.close();
            } catch (IOException e) {
                // Log error
            }
        }
    }
    
    public void shutdown() {
        running = false;
        executorService.shutdown();
    }
}
</code></pre>

<h2>System Architecture Overview</h2>
<p>The following diagram illustrates how zero-copy fits into our overall architecture:</p>

<div data-type="excalidraw">
<pre>{
  "type": "excalidraw",
  "version": 2,
  "source": "https://excalidraw.com",
  "elements": [
    {
      "type": "rectangle",
      "version": 100,
      "id": "client-box",
      "x": 100,
      "y": 100,
      "width": 150,
      "height": 80,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#a5d8ff",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 3 },
      "seed": 1234,
      "text": "Client\nApplication"
    },
    {
      "type": "rectangle",
      "version": 101,
      "id": "server-box",
      "x": 400,
      "y": 100,
      "width": 150,
      "height": 80,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#b2f2bb",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 3 },
      "seed": 1235,
      "text": "Java NIO\nServer"
    },
    {
      "type": "rectangle",
      "version": 102,
      "id": "kernel-box",
      "x": 400,
      "y": 250,
      "width": 150,
      "height": 80,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#ffd43b",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 3 },
      "seed": 1236,
      "text": "Linux Kernel\nsendfile()"
    },
    {
      "type": "rectangle",
      "version": 103,
      "id": "disk-box",
      "x": 700,
      "y": 250,
      "width": 150,
      "height": 80,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "#ffec99",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 3 },
      "seed": 1237,
      "text": "File System\n(Disk)"
    },
    {
      "type": "arrow",
      "version": 200,
      "id": "arrow-1",
      "x": 250,
      "y": 140,
      "width": 140,
      "height": 0,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 2 },
      "seed": 1238,
      "points": [[0, 0], [140, 0]],
      "lastCommittedPoint": [140, 0],
      "startBinding": null,
      "endBinding": null,
      "startArrowhead": null,
      "endArrowhead": "arrow"
    },
    {
      "type": "arrow",
      "version": 201,
      "id": "arrow-2",
      "x": 475,
      "y": 180,
      "width": 0,
      "height": 60,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 2 },
      "seed": 1239,
      "points": [[0, 0], [0, 60]],
      "lastCommittedPoint": [0, 60],
      "startBinding": null,
      "endBinding": null,
      "startArrowhead": null,
      "endArrowhead": "arrow"
    },
    {
      "type": "arrow",
      "version": 202,
      "id": "arrow-3",
      "x": 550,
      "y": 290,
      "width": 140,
      "height": 0,
      "angle": 0,
      "strokeColor": "#1e1e1e",
      "backgroundColor": "transparent",
      "fillStyle": "solid",
      "strokeWidth": 2,
      "roughness": 0,
      "opacity": 100,
      "groupIds": [],
      "roundness": { "type": 2 },
      "seed": 1240,
      "points": [[0, 0], [140, 0]],
      "lastCommittedPoint": [140, 0],
      "startBinding": null,
      "endBinding": null,
      "startArrowhead": null,
      "endArrowhead": "arrow"
    }
  ],
  "appState": {
    "gridSize": 20,
    "viewBackgroundColor": "#ffffff"
  },
  "files": {}
}
</pre>
</div>

<h2>Performance Benchmarks</h2>

<h3>Test Environment</h3>
<ul>
<li><strong>Hardware:</strong> Intel Xeon E5-2680 v4, 64GB RAM, NVMe SSD</li>
<li><strong>OS:</strong> Ubuntu 22.04 LTS (Kernel 5.15)</li>
<li><strong>Java:</strong> OpenJDK 17</li>
<li><strong>Network:</strong> 10 Gbps Ethernet</li>
<li><strong>File Size:</strong> 1 GB</li>
</ul>

<h3>Results</h3>
<table>
<thead>
<tr>
<th>Method</th>
<th>Throughput</th>
<th>CPU Usage</th>
<th>Memory Usage</th>
<th>Context Switches</th>
</tr>
</thead>
<tbody>
<tr>
<td>Traditional (8KB buffer)</td>
<td>520 MB/s</td>
<td>85%</td>
<td>450 MB</td>
<td>~45,000/sec</td>
</tr>
<tr>
<td>Traditional (64KB buffer)</td>
<td>680 MB/s</td>
<td>78%</td>
<td>520 MB</td>
<td>~38,000/sec</td>
</tr>
<tr>
<td><strong>Zero-Copy (transferTo)</strong></td>
<td><strong>1,850 MB/s</strong></td>
<td><strong>23%</strong></td>
<td><strong>85 MB</strong></td>
<td><strong>~8,500/sec</strong></td>
</tr>
</tbody>
</table>

<blockquote>
<p><strong>Key Finding:</strong> Zero-copy transfer provides <strong>~2.7x throughput improvement</strong> while reducing CPU usage by <strong>73%</strong> and memory usage by <strong>84%</strong>.</p>
</blockquote>

<h2>Implementation Plan</h2>

<h3>Phase 1: Prototype (Week 1-2)</h3>
<ol>
<li>Implement zero-copy transfer in isolated module</li>
<li>Create benchmarking suite</li>
<li>Validate performance improvements in test environment</li>
</ol>

<h3>Phase 2: Integration (Week 3-4)</h3>
<ol>
<li>Integrate with existing file serving infrastructure</li>
<li>Add monitoring and metrics</li>
<li>Implement fallback mechanism for non-supported platforms</li>
</ol>

<h3>Phase 3: Testing (Week 5)</h3>
<ol>
<li>Load testing with production-like traffic</li>
<li>Verify correctness of transfers</li>
<li>Performance regression testing</li>
</ol>

<h3>Phase 4: Deployment (Week 6-7)</h3>
<ol>
<li>Canary deployment to 5% of traffic</li>
<li>Monitor metrics and error rates</li>
<li>Gradual rollout to 100%</li>
</ol>

<h2>Platform Compatibility</h2>

<pre><code class="language-java">public class ZeroCopyHelper {
    private static final boolean ZERO_COPY_SUPPORTED;
    
    static {
        String osName = System.getProperty("os.name").toLowerCase();
        // Zero-copy works best on Linux with kernel 2.6+
        ZERO_COPY_SUPPORTED = osName.contains("linux");
    }
    
    public static boolean isZeroCopySupported() {
        return ZERO_COPY_SUPPORTED;
    }
    
    public static void transferFile(FileChannel source, 
                                     SocketChannel target) throws IOException {
        if (isZeroCopySupported()) {
            // Use zero-copy transfer
            long position = 0;
            long size = source.size();
            while (position < size) {
                long transferred = source.transferTo(position, size - position, target);
                position += transferred;
            }
        } else {
            // Fallback to traditional buffer-based transfer
            ByteBuffer buffer = ByteBuffer.allocate(65536);
            while (source.read(buffer) > 0) {
                buffer.flip();
                target.write(buffer);
                buffer.clear();
            }
        }
    }
}
</code></pre>

<h2>Monitoring and Observability</h2>

<pre><code class="language-java">public class TransferMetrics {
    private final AtomicLong totalBytesTransferred = new AtomicLong(0);
    private final AtomicLong totalTransfers = new AtomicLong(0);
    private final AtomicLong zeroCopyTransfers = new AtomicLong(0);
    
    public void recordTransfer(long bytes, boolean usedZeroCopy) {
        totalBytesTransferred.addAndGet(bytes);
        totalTransfers.incrementAndGet();
        if (usedZeroCopy) {
            zeroCopyTransfers.incrementAndGet();
        }
    }
    
    public Map&lt;String, Object&gt; getMetrics() {
        long total = totalTransfers.get();
        return Map.of(
            "total_bytes_transferred", totalBytesTransferred.get(),
            "total_transfers", total,
            "zero_copy_transfers", zeroCopyTransfers.get(),
            "zero_copy_percentage", total > 0 ? (zeroCopyTransfers.get() * 100.0 / total) : 0
        );
    }
}
</code></pre>

<h2>Risks and Drawbacks</h2>

<h3>Technical Risks</h3>
<ul>
<li><strong>Platform Dependency:</strong> Zero-copy relies on OS support (primarily Linux). Need fallback for Windows/macOS.</li>
<li><strong>File Modifications:</strong> Files being transferred should not be modified during transfer.</li>
<li><strong>SSL/TLS:</strong> Zero-copy doesn't work with encrypted connections (data must be encrypted in user space).</li>
</ul>

<h3>Mitigation Strategies</h3>
<ul>
<li>Implement comprehensive platform detection and fallback mechanisms</li>
<li>Use file locking or versioning to prevent modifications during transfer</li>
<li>For HTTPS traffic, evaluate alternatives like kernel TLS (kTLS) in Linux 4.13+</li>
</ul>

<h2>Alternatives Considered</h2>

<h3>1. Memory-Mapped Files (mmap)</h3>
<pre><code class="language-java">MappedByteBuffer buffer = fileChannel.map(FileChannel.MapMode.READ_ONLY, 0, size);
// Pros: Can be faster for random access
// Cons: Limited by virtual address space, complexity with large files
</code></pre>

<h3>2. Direct Buffers with NIO</h3>
<pre><code class="language-java">ByteBuffer directBuffer = ByteBuffer.allocateDirect(1024 * 1024);
// Pros: Some reduction in copying, more control
// Cons: Still involves user-space copies, GC pressure
</code></pre>

<h3>3. Native Libraries (JNI)</h3>
<p>Direct JNI calls to sendfile(). Rejected due to maintenance overhead and platform portability concerns.</p>

<h2>Unresolved Questions</h2>
<ul>
<li>Should we implement kTLS support for encrypted connections in Phase 2 or defer to Phase 3?</li>
<li>What should be the threshold file size for enabling zero-copy? (Currently considering 64KB+)</li>
<li>How do we handle partial writes in high-concurrency scenarios?</li>
<li>Should we expose zero-copy statistics via JMX for runtime monitoring?</li>
</ul>

<h2>References</h2>
<ul>
<li><a href="https://man7.org/linux/man-pages/man2/sendfile.2.html">Linux sendfile(2) man page</a></li>
<li><a href="https://docs.oracle.com/en/java/javase/17/docs/api/java.base/java/nio/channels/FileChannel.html#transferTo(long,long,java.nio.channels.WritableByteChannel)">Java FileChannel.transferTo() documentation</a></li>
<li>"Efficient data transfer through zero copy" - IBM developerWorks</li>
<li>"The C10K problem" by Dan Kegel</li>
</ul>

<h2>Sign-offs Required</h2>
<p>This proposal requires sign-off from:</p>
<ul>
<li>Backend Architecture Team Lead</li>
<li>Performance Engineering Team</li>
<li>DevOps Team (for deployment strategy)</li>
<li>Security Team (for security implications review)</li>
</ul>

<hr>

<p><em>Document created as an example RFC showcasing ArchDoc's rich editing capabilities including code highlighting, diagrams, tables, and structured technical content.</em></p>
