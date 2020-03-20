## 问题1：RestSharp 调用 https 报 504 gateway timeout 错误

两台华为鲲鹏服务器，三个应用程序，暂且叫为 app1，app2，app3。app1 内部调用 app2，app2 内部调用 app3，app3 访问 mysql。然后这三个 app 程序分别部署到那两台鲲鹏服务器，用 nginx 做轮训式负载。内部采用的是 RestSharp 库调用接口。api 地址是内网 IP 地址。

目前这种模式已经上线正常运行了。

接着运维同事把同样的程序复制出来放入同样的服务器的不同文件夹下，暂且名为 demo 文件夹。修改对应的 app 的远程调用地址为 https + 域名访问。其他 nginx 配置一样。运行发现三个 app 程序都能成功访问 swager ui 页面。但是 app1，app2 接口调用失败，报 **504 gateway timeout**。app3 能正常访问接口。

### 排查问题

从现象表明，这种很有可能是远程调用地址由内往改成 https + 域名导致的。开始怀疑是 RestSharp 对 https 有限制，要设置证书或是忽略证书。所以我尝试了以下代码来忽略 https

```c#
ServicePointManager.ServerCertificateValidationCallback = ServicePointManager.ServerCertificateValidationCallback = new RemoteCertificateValidationCallback( delegate { return true;} );
```

后来发现还是如此。

查网上的答案，基本上无外乎是 nginx 的设置，说是 fastcgi 缓冲大小，以及设置长一些 timeout 数值。

后来我又尝试使用微软自带的 HttpClient 来调用接口，结果发现还是如此。

### 采用排除法

目前打算采用排除法

1. 先让运维配合，关闭 nginx 代理，看在没有 nginx 的情况是否可以调用成功。
2. 如果不行，把远程调用接口地址改成内网地址。

如果这两步都不行，然后再想想还有其他什么可能没有。