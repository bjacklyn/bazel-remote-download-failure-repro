# bazel-remote-download-failure-repro

Steps to reproduce:
1. Build a modified version of bazel that always closes the http channel
    - Checkout bazel source at tag: 3.4.1
    - Apply the below change
    - Build bazel `bazel build //src:bazel-dev`
    - (Alternatively) I've provided the `bazel-dev` binary I built locally with this change
```
diff --git a/src/main/java/com/google/devtools/build/lib/remote/http/HttpDownloadHandler.java b/src/main/java/com/google/devtools/build/lib/remote/http/HttpDownloadHandler.java
index 37012aa62a..29396f5e5a 100644
--- a/src/main/java/com/google/devtools/build/lib/remote/http/HttpDownloadHandler.java
+++ b/src/main/java/com/google/devtools/build/lib/remote/http/HttpDownloadHandler.java
@@ -43,7 +43,7 @@ import java.util.Map.Entry;
 final class HttpDownloadHandler extends AbstractHttpHandler<HttpObject> {
 
   private OutputStream out;
-  private boolean keepAlive = HttpVersion.HTTP_1_1.isKeepAliveDefault();
+  private boolean keepAlive = false;
   private boolean downloadSucceeded;
   private HttpResponse response;
 
@@ -97,7 +97,7 @@ final class HttpDownloadHandler extends AbstractHttpHandler<HttpObject> {
       if (!downloadSucceeded) {
         out = new ByteArrayOutputStream();
       }
-      keepAlive = HttpUtil.isKeepAlive((HttpResponse) msg);
+      keepAlive = false;
     }
 
     if (msg instanceof HttpContent) {
@@ -202,7 +202,7 @@ final class HttpDownloadHandler extends AbstractHttpHandler<HttpObject> {
       }
     } finally {
       out = null;
-      keepAlive = HttpVersion.HTTP_1_1.isKeepAliveDefault();
+      keepAlive = false;
       downloadSucceeded = false;
       response = null;
     }
```
2. Start a bazel-remote http server using https://github.com/buchgr/bazel-remote
    - `mkdir data`
    - `docker run -u 1000:1000 -v $(pwd)/data:/data -p 9090:9090 -p 9091:8080 buchgr/bazel-remote-cache`

3. Clone this repo and run this build to reproduce
    - `for i in {1..5}; do ./bazel-dev shutdown && rm -rf bazel-bin/* && ./bazel-dev build --remote_http_cache=http://172.17.0.1:9091 --experimental_allow_tags_propagation --remote_download_minimal //:pkg --features=thin_lto; done`
    
Notes:
  - The `for i in {1..5}` is there because the first build always passes for me, and the subsequent builds always fail with `Linking of rule '@com_google_protobuf//:protoc' failed: I/O exception during sandboxed execution: null`
  - `--experimental_allow_tags_propagation` is passed because `tags = ["no-remote"]` does not seem to be respected on `cc_binary` / `cc_library` without it
  - `--features=thin_lto` seems to require clang so run `export CC=clang` if needed (I installed clang-11, so I used `export CC=clang-11`)
  - I am not 100% that BOTH `thin_lto` and cache misses (i.e. with `tags=["no-remote"]` or with `--modify_execution_info='CppLink.*=+no-remote'`) are needed in order to reproduce this, but that combo consistently reproduces for me
  _ I am also not 100% that the `keep_alive = false` patch is needed to reproduce. It looks like bazel inspects the headers returned by the server to decide whether to close the channel or keep it open. When I was investigating this earlier I was only able to reproduce the problem 80-90% of the time (i.e. intermittent). We use `nginx` internally as our remote cache, and I don't know when/why nginx will decide to close the connection.
  - When `--remote_download_minimal` is passed the build simply fails on the linking step, whereas when the flag is not passed bazel prints `WARNING: Reading from Remote Cache: BulkTransferException` but doesn't fail the build. I have heard complaints internally that whenever they see `Remote Cache: BulkTransferException` in their local build they stop getting remote cache hits for the remainder of the build, but I haven't repro'd that (yet?).


The bottom line is this bug seems to be caused by a race condition between releasing a channel to the channelPool, and subsequently closing the channel presumably after another thread has acquired it from the pool and tries to use it. That would explain why channelpool allows the channel to be reused (because it hasn't be closed yet at time of release/acquire), and could also explain the intermittent nature of the failure (sometimes the releasing thread closes the channel before it is acquired by another thread instead of after?)

If ^ is true then this is the code that looks suspicious to me (comments added by me). Also note that there is `failAndReset()` which follows the same pattern.
```
  private void succeedAndReset(ChannelHandlerContext ctx) {
    try {
      succeedAndResetUserPromise();    // this resolves the download promise with internally ends up calling releaseDownloadChannel(ch)
    } finally {
      reset(ctx);                      // this conditionally closes the channel if the server requests it closed
    }
  }
```
