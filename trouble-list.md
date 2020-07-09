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

## 问题3：鲲鹏服务器对时间字符串转时间格式的问题

产生这个问题原因是，A Api发起一个请求，传了一个时间字符串参数给 B Api。B Api 接受这个时间字符串反序列化为强类型 `DateTime` 时报错，导致请求返回 400 错误。

产生的原因：从服务器返回的错误信息是这样的，"2020/4/28 上午 8:00:00" 不是有效的时间格式。从本地环境和测试环境上没有出现这个问题可以看出，是本身服务器环境的原因导致时间转化成上面所示的格式。

所以我们知道是鲲鹏服务器对这种时间格式的字符串是不识别的，所以我们需要强制转换一下指定时间格式即可

```c#
await client.GetAsync<object>("/student/getpaged?startTime="+time.ToString("yyyy-MM-DD HH:mm:sssss"));
```

思考：但是问题虽然解决了，但是我们之后是不是只要涉及到传时间参数是不是都得指定格式呢？有没有什么一劳永逸的方法呢？~~目前我还没有得到解决方法~~。希望可以得到大家的解决方案。

经翻看 [DateTime](https://source.dot.net/#System.Private.CoreLib/DateTimeFormat.cs) 源码得知，时间类型的 `ToString` ，其实是调用了 `DateTimeFormat.Format(DateTime dateTime, string? format, IFormatProvider? provider)` ，这个方法又调用了重载函数 `string Format(DateTime dateTime, string? format, IFormatProvider? provider, TimeSpan offset)`。在没有提供 format 和 formatProvider 的情况下，系统会创建一个默认的 `DateTimeFormatInfo` 实例化对象，然后进行时间格式化。

所以我们只要修改对应的 `DateTimeFormatInfo` 成员属性即可。代码如下

```c#
CultureInfo culture = (CultureInfo)CultureInfo.CurrentCulture.Clone();
culture.DateTimeFormat.ShortDatePattern = "yyyy-MM-dd";
CultureInfo.CurrentCulture = culture;
```

这样就可以一劳永逸了，而不用在每次传时间参数的时候指定 format。

除了上面的方式，其实也可以通过反射来在原来的 `DateTimeFormatInfo` 基础上直接修改 `ShortDatePattern` 等属性。为什么要反射？先来看 `ShortDatePattern` 属性定义：

```c#
public string ShortDatePattern { get; set; }
```

那我们是不是可以直接对它进行赋值呢

```c#
CultureInfo.CurrentCulture.DateTimeFormat.ShortDatePattern = "yyyy-MM-dd";
```

答案是不行，会报错：`System.InvalidOperationException : Instance is read-only.` 意思是说 `ShortDatePattern` 是只读属性。明明是 get,set 为什么会是只读呢？翻看[源码](https://source.dot.net/#System.Private.CoreLib/DateTimeFormatInfo.cs,845)就很明白了，因为它是通过一个字段 `_isReadOnly` 来控制属性是否能被修改。看这个类的其他共有属性，就会发现都是通过这个字段来控制的。所以通过反射修改这个字段设置为 true 从而达到在原有的示例直接修改这个属性的目的。

```c#
var t = CultureInfo.CurrentCulture.DateTimeFormat.GetType();
var isReadOnlyFieldInfo = t.GetField("_isReadOnly", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Instance);
isReadOnlyFieldInfo.SetValue(CultureInfo.CurrentCulture.DateTimeFormat, false);
CultureInfo.CurrentCulture.DateTimeFormat.ShortDatePattern = "yyyy-MM-dd";
isReadOnlyFieldInfo.SetValue(CultureInfo.CurrentCulture.DateTimeFormat, true);
```

## 问题4：时间戳于时间格式的转换

时间戳的展现形式分为两种单位，一种是秒级，还有一种是毫秒级。秒级展现形式长度一般为 10 位，毫秒级时间戳一般位 13 位。而时间戳展现的位数一般是 13 位。转换方式一般就是先转换为本地时间，然后与 1970-01-01 的时间戳对比。

```c#
public class DateTimeHelper
{
    public static long ToTimestamp(DateTime dateTime, TimestampStyle timestampStyle = TimestampStyle.Seconds)
    {
        var start = new DateTime(1970, 1, 1, 0, 0, 0).ToLocalTime();
        var span = dateTime.ToLocalTime() - start;

        if (timestampStyle == TimestampStyle.Seconds)
            return (long)span.TotalSeconds;
        else
            return (long)span.TotalMilliseconds;
    }

    public static DateTime ToDateTime(long timestamp, TimestampStyle timestampStyle = TimestampStyle.Seconds)
    {
        // Unix timestamp is seconds past epoch
        DateTime dtDateTime = new DateTime(1970, 1, 1, 0, 0, 0, 0, DateTimeKind.Utc);
        if (timestampStyle == TimestampStyle.Seconds)
            dtDateTime = dtDateTime.AddSeconds(timestamp).ToLocalTime();
        else
            dtDateTime = dtDateTime.AddMilliseconds(timestamp).ToLocalTime();
        return dtDateTime;
    }

    public enum TimestampStyle
    {
        Seconds,
        Milliseconds
    }
}
```

## 问题5: .NET Core 3.X 内置序列化中文转码问题，设置全局序列化器无效

这个问题是很普遍的问题，就是 `System.Text.Json.JsonSerializer` 序列化对象时，会把中文属性值转码。

网上很多解决方案都是说加全局序列化器：

```c#
services.AddControllers().AddJsonOptions(config =>
{
   Encoder = System.Text.Encodings.Web.JavaScriptEncoder.UnsafeRelaxedJsonEscaping;
});
```

但是运行发现这是没效果的，只能把 `JsonSerializerOptions` 传给 `JsonSerializer.Serialize(obj, options)` 才能执行正常。但是有很多地方需要手动添加 options，所以我翻找资料都无果，最后有人告知我这个功能目前 .NET Core 版本里不支持设置默认的序列化器，只能一个个传。具体详见 https://stackoverflow.com/questions/58331479/how-to-globally-set-default-options-for-system-text-json-jsonserializer ，以及这个 [issue](https://github.com/dotnet/runtime/issues/31094)。

但是我还是不死心，通过翻看[源码](https://source.dot.net/#System.Text.Json/System/Text/Json/Serialization/JsonSerializer.Write.String.cs,61)得知
JsonSerializer.Serialize 在没有传 setting 的时候是调用了 JsonSerializerOptions.s_defaultOptions 这个单例对象。所以很自然有了另一种解决方法：

```c#
// Startup.cs 
// Configure
var fieldInfo = typeof(JsonSerializerOptions).GetField("s_defaultOptions", System.Reflection.BindingFlags.NonPublic | System.Reflection.BindingFlags.Static | System.Reflection.BindingFlags.Instance);

 var options = fieldInfo.GetValue(null) as JsonSerializerOptions;
 options.Encoder = JavaScriptEncoder.UnsafeRelaxedJsonEscaping;
```

这样就不用在每个序列化方法去传入 option 了，从而实现了全局序列化器的效果。