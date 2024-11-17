---
title: Java 实现文件下载
description: Java 使用HttpClient和OKHttp实现文件下载，以及同步异步处理方法、设置请求头、超时设置。
date: 2024-11-16
image: java.jpg
categories:
    - Java
    - Java网络编程
---

# Java 实现文件下载

## 使用HttpClient下载

### 阻塞方式下载
```java
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpClient;
import java.nio.file.Paths;
import java.nio.file.Path;

public class Main {
    public static void main(String[] args) {
        downloadAFile("https://pc-package.wpscdn.cn/wps/download/W.P.S.20.2735.exe", "W.P.S.20.2735.exe");
    }

    public static void downloadAFile(String url, String fileName) {
        HttpClient client = HttpClient.newHttpClient();
        //创建一个HttpRequest请求
        HttpRequest request = HttpRequest.newBuilder(URI.create(url)).build();
        Path filePath = Paths.get(fileName);
        try {
            //发送Http请求
            HttpResponse<Path> response = client.send(request, HttpResponse.BodyHandlers.ofFile(filePath));
            //判断是否成功下载
            if (response.statusCode() == 200) {
                System.out.printf("文件%s已经成功保存!", fileName);
            }
        } catch (IOException | InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

}

```

**在上述代码中，实现了一个名为``downloadAFile()``函数。使用 ``HttpClient.send()`` 方法发送Http请求并将响应体写入文件，并获得``HttpResponse<Path>``类型的``response``对象，使用该对象获得请求的状态码``statusCode``。**

### 异步方式下载
``` java
import java.io.IOException;
import java.net.URI;
import java.net.http.HttpRequest;
import java.net.http.HttpResponse;
import java.net.http.HttpClient;
import java.nio.file.Paths;
import java.nio.file.Path;
import java.util.Scanner;

public class Main {
    public static void main(String[] args) {
        downloadAFile("https://pc-package.wpscdn.cn/wps/download/W.P.S.20.2735.exe", "W.P.S.20.2735.exe");
        System.out.println("下载已经开始，按回车键结束...");
        Scanner scanner = new Scanner(System.in);
        scanner.nextLine();
    }

    public static void downloadAFile(String url, String fileName) {
        HttpClient client = HttpClient.newHttpClient();
        //创建一个HttpRequest请求
        HttpRequest request = HttpRequest.newBuilder(URI.create(url)).build();
        Path filePath = Paths.get(fileName);
        //异步下载
        client.sendAsync(request, HttpResponse.BodyHandlers.ofFile(filePath)).thenAccept(Main::downloadCallBack).exceptionally(e -> {
            System.out.println("下载失败！" + e.getMessage());
            return null;
        });
    }

    private static void downloadCallBack(HttpResponse<Path> pathHttpResponse) {
        if (pathHttpResponse.statusCode() == 200) {
            System.out.println("下载成功！文件已经保存至：" + pathHttpResponse.body());
        } else {
            System.out.printf("下载失败！状态码为%d。\n", pathHttpResponse.statusCode());
        }
    }
}

```
**这里使用``client.sendAsync()``异步下载文件，thenAccept()来指定下载完成时的回调函数，``exceptionally()``来处理异常。注意，由于这里是异步下载的，不能使用``try{}catch{}``来处理异常，异常不能被捕获。**

## 使用OKHttp下载

### 导入OkHttp依赖
在pom.xml的``<dependencies></dependencies>``中添加以下代码：
```xml
<!-- https://mvnrepository.com/artifact/com.squareup.okhttp3/okhttp -->
        <dependency>
            <groupId>com.squareup.okhttp3</groupId>
            <artifactId>okhttp</artifactId>
            <version>3.14.9</version>
        </dependency>
```
#### OKHttp获取文本文件
Download.java
```java
package org.example;
import okhttp3.*;

import java.io.IOException;

public class Download{
    public final OkHttpClient client = new OkHttpClient();
    public void run() throws IOException {
        Request request = new Request.Builder().url("http://www.baidu.com").build();
        try(Response response = client.newCall(request).execute()){
            if (response.body() != null) {
                System.out.println(response.body().string());
            }
        }catch (Exception e){
            System.out.println(e.getMessage());
        }
    }
}
```

Main.java
```java
package org.example;

import java.io.IOException;

public class Main {
    public static void main(String[] args) {
        Download download = new Download();
        try{
            download.run();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }
}
```

#### OKHttp异步请求
Download.java
```java
package org.example;

import okhttp3.*;

import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.InputStream;
import java.util.Objects;

public class Download {
    private final String url;
    private final String fileName;
    private final String filePath;
    private final OkHttpClient client = new OkHttpClient();

    public Download(String url, String fileName, String filePath) {
        this.url = url;
        this.fileName = fileName;
        this.filePath = filePath;
    }

    public void run() {
        Request request = new Request.Builder().url(url).build();
        Call call = client.newCall(request);
        call.enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                e.printStackTrace();
            }

            @Override
            //响应时的回调函数
            public void onResponse(Call call, Response response) throws IOException {
                if (!response.isSuccessful()) {
                    throw new IOException("Unexpected code " + response);
                }

                try (InputStream inputStream = Objects.requireNonNull(response.body()).byteStream();
                     //从响应体获取字节流
                     FileOutputStream outputStream = new FileOutputStream(new File(filePath, fileName))) {
                    //创建文件输出流
                    byte[] buffer = new byte[2048]; //创建缓存区，大小为2048KB
                    int len;     //用于还剩下多少字节没有被读取
                    while ((len = inputStream.read(buffer)) != -1) {
                        //还没有被读取完
                        outputStream.write(buffer, 0, len);  //将缓冲区数据写入文件
                    }

                    System.out.printf("文件%s已下载至%s%n", fileName, filePath);

                } catch (IOException e) {
                    e.printStackTrace();
                }
            }
        });
    }
}
```
Main.java
```java
package org.example;

public class Main {
    public static void main(String[] args) {
        String url = "https://down-tencent.huorong.cn/sysdiag-all-x64-6.0.4.0-2024.11.16.1.exe";
        String filename = "sysdiag-all-x64-6.0.4.0-2024.11.16.1.exe";
        Download download = new Download(url, filename, "D:/");
        download.run();
        
    }
}
```
**上述代码中，使用``Request.Builder().url(url).build()``创建一个Request对象，使用``Call``类的``call``对象发起Http Get请求，使用``enqueue()``方法处理响应和错误。**
在处理响应时，首先将``response.body()``响应体通过``byteStream()``方法转化为字节流，然后于FileInputStream一起使用写入文件。


#### OKHttp同步请求
无需使用``call.enqueue()``方法，最后使用断言``assertThat(response.code(), equalTo(200));``判断状态码是否等于200即可。

#### 自定义请求头
```java
    Request request = new Request.Builder().url(url).addHeader("Content-Length","application/x-msdownload").build();
```

### 允许重定向
```java
    private final OkHttpClient client = new OkHttpClient().newBuilder().followRedirects(true).build();
```

### 超时设置
```java
    private final OkHttpClient client = new OkHttpClient().newBuilder().readTimeout(3, TimeUnit.SECONDS).build();
```
### 取消请求
可以使用``Call.cancel()``终止请求。


