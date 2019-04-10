# 用 HttpServletRequestWrapper 定制客户端请求参数

## 介绍

* `HttpServletRequestWrapper` 实现了 `HttpServletRequest` 接口，可以让开发人员很方便的改造发送给 Servlet 的请求
* 应用了装饰模式
* 一般要和 Filter 配合应用

## 应用场景

需要修改客户端请求参数的场合，例如

* 将不支持的语言参数修改为默认语言
* 将加密的 DeviceId 解密，并解析出其中的 imei 和 sn，同时在客户端请求里添加这 2 个参数
  * `deviceId = hex(rc4(imei + '_' + sn))`

## 示例

就以上面所说的解密 DeviceId 为例

### web.xml 配置

添加一个解析 DeviceId 的 Filter

```markup
<!-- 解析加密的 deviceId 得到 imei 和 sn -->
<filter>
    <filter-name>deviceIdParseFilter</filter-name>
    <filter-class>com.meizu.browser.app.common.DeviceIdParseFilter</filter-class>
    <init-param>
        <param-name>encoding</param-name>
        <param-value>UTF-8</param-value>
    </init-param>
    <init-param>
        <param-name>forceEncoding</param-name>
        <param-value>true</param-value>
    </init-param>
</filter>
<filter-mapping>
    <filter-name>deviceIdParseFilter</filter-name>
    <url-pattern>*.do</url-pattern>
</filter-mapping>
```

### Filter 代码

```java
public class DeviceIdParseFilter implements Filter {

    private static final String KEY = "1234567890";
    private static final Logger log = Logger.getLogger(DeviceIdParseFilter.class);
    private static final String[] DEFAULT_RESULT = {"",""};


    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }


    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
            throws IOException, ServletException {
        String deviceId = request.getParameter("deviceId");
        if (deviceId != null && deviceId.length() > 0) {
            String[] result = parseDeviceId(deviceId);
            DeviceIdParseRequest req = new DeviceIdParseRequest((HttpServletRequest) request,
                    result[0], result[1]);
            chain.doFilter(req, response);
        } else {
            chain.doFilter(request, response);
        }
    }


    /**
     * <pre>
     * 从 deviceId 里解析出 imei 和 sn
     * 
     * imei = result[0]
     * sn   = result[1]
     * </pre>
     * 
     * @param deviceId
     * @return
     */
    private static final String[] parseDeviceId(String deviceId) {
        try {
            String src = Rc4Util.decrypt(deviceId, KEY);
            if (src.indexOf('_') >= 0) {
                return src.split("_");
            }
        } catch (Exception e) {
            log.error(e, e);
        }
        return DEFAULT_RESULT;
    }


    @Override
    public void destroy() {

    }

}
```

这个 Filter 会将包含 deviceId 参数的请求进行如下处理

1. 将 deviceId 的值用 RC4 进行解密
2. 从解密出来的 deviceId 里解析出 imei 和 sn
3. 将请求改造成 DeviceIdParseRequest，这就是我们的 HttpServletRequestWrapper 

而不包含 deviceId 参数的请求不做任何处理

### HttpServletRequestWrapper 代码

```java
public class DeviceIdParseRequest extends HttpServletRequestWrapper {

    private String imei;
    private String sn;


    /**
     * @param request
     */
    public DeviceIdParseRequest(HttpServletRequest request) {
        super(request);
        this.imei = "";
        this.sn = "";
    }


    /**
     * @param request
     * @param imei
     * @param sn
     */
    public DeviceIdParseRequest(HttpServletRequest request, String imei, String sn) {
        super(request);
        this.imei = imei;
        this.sn = sn;
    }


    @Override
    public String getParameter(String name) {
        if ("imei".equals(name)) {
            return imei;
        } else if ("sn".equals(name)) {
            return sn;
        } else {
            return super.getParameter(name);
        }
    }


    @Override
    public String[] getParameterValues(String name) {
        if ("imei".equals(name)) {
            return new String[] { imei };
        } else if ("sn".equals(name)) {
            return new String[] { sn };
        } else {
            return super.getParameterValues(name);
        }
    }

}
```

这里针对 imei 和 sn 进行了特殊处理，返回的不是客户端提交的参数，而是在 Filter 里通过解析 deviceId 得到的 imei 和 sn

需要注意的是

* 如果用 `request.getParameter()` 获取客户端请求参数的值，那么只需要重写该方法就行了
* 如果用 SpringMVC 的 `@RequestParam` 注解来获取请求参数的值，那么需要重写 `getParameterValues` 方法：因为 SpringMVC 是用这个方法来获取参数值的

### 运行结果

#### **用于测试的 controller**

```java
@Controller
@RequestMapping("/api/")
public class TestController {

    @ResponseBody
    @RequestMapping("test.do")
    public Result test(String deviceId, String imei, String sn) {
        Map<String, String> map = new HashMap<>();
        map.put("deviceId", deviceId);
        map.put("imei", imei);
        map.put("sn", sn);
        return new Result(map);
    }

}
```

这个测试类把接收到的参数直接返回

#### **请求参数不包含 deviceId**

请求 url

```text
http://browser.in.meizu.com/api/test.do?reqno=123456&imei=imei&sn=1001&model=mx6&os=flyme6&ver=1.0.0&locale=en_US
```

返回结果

```javascript
{
    "code": "200",
    "message": "",
    "redirect": "",
    "value": {
        "sn": "1001",
        "imei": "imei",
        "deviceId": null
    }
}
```

#### **请求参数包含 deviceId**

请求 url

```text
http://browser.in.meizu.com/api/test.do?reqno=123456&sn=1001&model=mx6&os=flyme6&ver=1.0.0&locale=en_US&deviceId=7cfbf5cbd70bcf1c006d7d0aa77688518444497a2b45683ea41ce690e92d6d38
```

返回结果

```javascript
{
    "code": "200",
    "message": "",
    "redirect": "",
    "value": {
        "sn": "111",
        "imei": "org.testng.annotations.Test;",
        "deviceId": "7cfbf5cbd70bcf1c006d7d0aa77688518444497a2b45683ea41ce690e92d6d38"
    }
}
```

可以看到

* 请求参数里不存在的 imei 能获取到值
* 请求参数里存在的 sn 值被修改了

## 附录

### 关于 RC4 加密算法

deviceId 的加解密用到了 RC4 算法，下面是相应的代码

```java
/**
 * @Auther zhangyan
 * 
 * File Created at 2018年1月23日
 * 
 */
package com.meizu.browser.utils;

import org.apache.commons.codec.DecoderException;
import org.apache.commons.codec.binary.Hex;
import org.bouncycastle.crypto.engines.RC4Engine;
import org.bouncycastle.crypto.params.KeyParameter;

/**
 * rc4 加解密
 * 
 * @author zhangyan
 * @date 2018年1月23日
 */
public class Rc4Util {

    /**
     * 将明文用 rc4 加密并返回 16 进制编码格式的密文
     * 
     * @param src
     * @param key
     * @return
     */
    public static final String encrypt(String src, String key) {
        RC4Engine rc4 = new RC4Engine();
        rc4.init(true, new KeyParameter(key.getBytes()));
        byte[] in = src.getBytes();
        int len = in.length;
        byte[] out = new byte[len];
        rc4.processBytes(in, 0, len, out, 0);
        return Hex.encodeHexString(out);
    }


    /**
     * 将 16 进制编码格式的密文解密
     * 
     * @param src
     * @param key
     * @return
     */
    public static final String decrypt(String src, String key) {
        RC4Engine rc4 = new RC4Engine();
        rc4.init(false, new KeyParameter(key.getBytes()));
        byte[] in;
        try {
            in = Hex.decodeHex(src.toCharArray());
        } catch (DecoderException e) {
            throw new RuntimeException(e);
        }
        int len = in.length;
        byte[] out = new byte[len];
        rc4.processBytes(in, 0, len, out, 0);
        return new String(out);
    }

}
```

