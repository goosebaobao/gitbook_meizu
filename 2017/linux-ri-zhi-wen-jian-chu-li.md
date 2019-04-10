# Linux 日志文件处理

## 目标

通过分析 error.log 日志，统计出有哪些异常及其数量

## 分析

首先要看下异常日志的格式

```text
2017-07-19 01:03:41,399 ERROR [qtp738355611-33898] [com.meizu.apkfilemanage.web.AppInfoController] - appUpdate:
java.lang.NumberFormatException: !hex 215
        at org.eclipse.jetty.util.TypeUtil.convertHexDigit(TypeUtil.java:375)
        at org.eclipse.jetty.util.UrlEncoded.decodeUtf8To(UrlEncoded.java:545)
        at org.eclipse.jetty.util.UrlEncoded.decodeTo(UrlEncoded.java:601)
        at org.eclipse.jetty.server.Request.extractParameters(Request.java:298)
        at org.eclipse.jetty.server.Request.getParameter(Request.java:708)
        at com.meizu.apkfilemanage.common.util.BuildBeanUtil.buildAppUpdate(BuildBeanUtil.java:905)
        at com.meizu.apkfilemanage.web.AppInfoController.appUpdate(AppInfoController.java:204)
        at sun.reflect.GeneratedMethodAccessor132.invoke(Unknown Source)
```

经过分析，发现每次打印异常的格式如下

* 第一行
  * 时间，日志级别，线程信息，类，异常消息（业务）
* 第二行及后续
  * 异常堆栈信息，其中第二行是异常的类名及异常消息

那么我们的思路是

* 找到有第三个域为 ERROR 的那一行
* 该行的下一行就是我们需要统计分析的那一行

## 命令

```bash
zcat googleinstaller-error.log.20170719.gz | awk -v line=0 '{if (line==1) print($0); if ($3=="ERROR") {line=1;} else {line=0;}}' | sort | uniq -c | sort -rn
```

命令执行结果如下

```bash
54 redis.clients.jedis.exceptions.JedisConnectionException: Could not get a resource from the pool
54 org.mybatis.spring.MyBatisSystemException: nested exception is org.apache.ibatis.exceptions.PersistenceException:
28 org.springframework.http.converter.HttpMessageNotWritableException: Could not write JSON: org.eclipse.jetty.io.EofException; nested exception is com.google.gson.JsonIOException: org.eclipse.jetty.io.EofException
12 org.springframework.dao.DeadlockLoserDataAccessException:
 8 org.springframework.transaction.CannotCreateTransactionException: Could not open JDBC Connection for transaction; nested exception is java.sql.SQLException: An attempt by a client to checkout a Connection has timed out.
 7 java.lang.IllegalArgumentException: fromIndex(150) > toIndex(127)
 6 redis.clients.jedis.exceptions.JedisConnectionException: java.net.SocketTimeoutException: Read timed out
 4 org.springframework.dao.DataIntegrityViolationException:
 4 java.lang.IllegalStateException: Optional long parameter 'timestamp' is present but cannot be translated into a null value due to being declared as a primitive type. Consider declaring it as object wrapper for the corresponding primitive type.
 3 org.eclipse.jetty.io.EofException
 3 java.lang.NullPointerException
 3 java.lang.IllegalArgumentException: fromIndex(150) > toIndex(131)
 3 com.alibaba.fastjson.JSONException: syntax error, pos 720
 1 org.springframework.dao.CannotAcquireLockException:
 1 java.lang.NumberFormatException: !hex 71
 1 java.lang.NumberFormatException: !hex 34
 1 java.lang.NumberFormatException: !hex 215
 1 java.lang.NumberFormatException: !hex 15
 1 com.alibaba.fastjson.JSONException: unclosed string : �
 1 com.alibaba.fastjson.JSONException: unclosed string : ̂
 1 com.alibaba.fastjson.JSONException: syntax error, unexpect token error
 1 com.alibaba.fastjson.JSONException: not match ':' - ,
 1 com.alibaba.fastjson.JSONException: error parse false
```

### **zcat**

和 cat 命令类似，不过作用的对象是压缩文件

### **\|**

这是 linux 管道符，他的作用是把管道符左边命令的输出当作管道符右边命令的输入，示例

```bash
ps aux | grep 'java'
```

### **awk**

这个命令很复杂，这里解释一下前面统计异常数量的命令

```bash
awk -v line=0 '{if (line==1) print($0); if($3=="ERROR") {line=1;} else {line=0;}}'
```

我们知道，awk 命令会对输入的文本逐行进行处理，类似以下的处理过程

```python
while 当前行不为空
    处理 当前行
    移动到 下一行
```

那么上面的命令就可以这样理解

```python
# 指示当前行是否包含异常信息 
line = 0
while 当前行不为空
    if line == 1 
        输出当前行
    if $3 == 'ERROR'
        line = 1
    else
        line = 0
```

这段代码的作用就是找到含有 ERROR 的行，并将其下一行打印输出

### **sort**

以行为单位对文本进行排序

### **uniq**

删除重复的行，参数

* -c 在输出行前面加上每行在输入文件中出现的次数。
* -d 仅显示重复行。
* -u 仅显示不重复的行

## 一些常用的 awk 命令

分析 nginx 的日志文件

### 统计各个接口的 pv

```bash
awk '{if($14=="/proxy") {print $15} else {print $14}}' xxx_access.log|sort|uniq -c|sort -rn|head -n 20
```

### 统计慢接口

#### 响应时间超过 1 秒的接口数

```bash
awk '{if($9>1)print $0}'|wc -l
```

#### 如果想要输出具体是哪些接口，可以结合上面命令

```bash
awk '{if($9>1)print $0}'|awk '{if($14=="/proxy") {print $15} else {print $14}}' |sort|uniq -c|sort -rn|head -n 20
```

#### 如果要分时段统计

```bash
awk '{if($9>1)print $1}'|awk -F ':' '{print $1}'|uniq -c
```

### 统计响应码

```bash
awk '{print $2}'| sort | uniq -c | sort -rn
```

