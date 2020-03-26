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

### 找出问题点

通过排除法，发现确定是 https 的问题，因为运维同事把 https 域名改成内网的 ip 地址，运行是正常的。通过这些现象表明，可以知道是 https 协议相关的原因。

遗留问题点：如何才能在 https 协议下正常使用 RestClient 远程调用接口。从查看的资料上看，添加对应的 cert 文件这一举措还没尝试。

## 问题2：前端传 long 类型数字到后端会丢失精度

产生这个问题的原因很简单，就是因为对于前端而言（Javascript），long 类型的数字类型最大长度只有 52 位（也就是 2^53）。所以团队协商，将所有的 long 的请求模型的字段全部改成字符串。这样前端就很好处理了。但是对于某些后端来讲就很麻烦了，因为某些后端语言是强类型静态类型编程语言，所以后端接收了字符串，还是得一个个转成 long 类型操作。所以得想个一劳永逸的方法。

对于 C# 而言，这个问题就很好解决。以 .netcore3.1 为例，api 接收参数和响应参数都是用的内置的序列化器（System.Text.Json.JsonSerialize）。所以我们直接全局注册一个读写 string 为 long 的 Converter 即可。

问题到此还没结束，后来又引入了一个问题，那就是将字符串数组转换成 long 数组。原来的 `JsonConverter<long>` 就不行了。开始很容易想到直接再写个针对数组的转换器就好了，但后来发现对里面的一些基本用法还不熟，纠结了 2 小时，最后看到官方的 [DictionaryConverter](https://docs.microsoft.com/zh-cn/dotnet/standard/serialization/system-text-json-converters-how-to?view=netcore-3.0#support-dictionary-with-non-string-key) 例子才知道怎么用了。

```c#
public class ArrayLongToStringConverter : JsonConverter<long[]>
    {
        public override long[] Read(ref Utf8JsonReader reader, Type typeToConvert, JsonSerializerOptions options)
        {
            if (reader.TokenType != JsonTokenType.StartArray)
                throw new JsonException();
            List<long> values = new List<long>();
            while (reader.Read())
            {
                if (reader.TokenType == JsonTokenType.EndArray)
                    return values.ToArray();

                string value = reader.GetString();
                long.TryParse(value, out long lValue);
                values.Add(lValue);
            }

            return values.ToArray();
        }

        public override void Write(Utf8JsonWriter writer, long[] values, JsonSerializerOptions options)
        {
            writer.WriteStartArray();

            var stringValues = Array.ConvertAll(values, p => p.ToString());
            JsonSerializer.Serialize(stringValues, options);

            writer.WriteEndArray();
        }
    }
```

