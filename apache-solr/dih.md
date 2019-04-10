# DIH

## 概念和术语

* DataSource
  * 数据源，定义了数据的位置，对于数据库来说，是一个数据源名称（DSN）；对于一个 http 的数据源，是一个 URL
* Entity
  * 实体，是发送给 solr 进行索引的数据。对于关系型数据库来说，实体是表或者视图
* Processor
  * 处理机，用于处理实体，即从数据源抽取、转换数据，添加到 solr 进行索引。可自定义处理机
* Transformer
  * 转换器，实体里的字段可能做默认的转换。可能会修改字段，创建新字段，从某个字段生成多个文档。DIH 内建了多个 Transformer，进行诸如日期修改，html 清洗等功能。可自定义转换器

## 配置

### 配置 solrconfig.xml

solrconfig.xml 在core\(本例中创建了一个 core，名为 testcore\)的 conf 目录下，如下

```bash
[root@dev conf]# pwd
/data/solr-5.3.1/server/solr/testcore/conf
[root@dev conf]# ll
total 124
-rw-r--r-- 1 root root  4041 Sep 17  2015 currency.xml
-rw-r--r-- 1 root root  1386 Sep 17  2015 elevate.xml
drwxr-xr-x 2 root root  4096 Sep 17  2015 lang
-rw-r--r-- 1 root root 29216 Feb  3 16:04 managed-schema
-rw-r--r-- 1 root root   329 Sep 17  2015 params.json
-rw-r--r-- 1 root root   894 Sep 17  2015 protwords.txt
-rw-r--r-- 1 root root 63110 Sep 17  2015 solrconfig.xml
-rw-r--r-- 1 root root   795 Sep 17  2015 stopwords.txt
-rw-r--r-- 1 root root  1148 Sep 17  2015 synonyms.txt
```

DIH 必须在 solrconfig.xml 里注册。示例

```markup
<requestHandler name="/dataimport" 
                class="org.apache.solr.handler.dataimport.DataImportHandler">
    <lst name="defaults">
        <str name="config">/path/to/my/DIHconfigfile.xml</str>
    </lst>
</requestHandler>
```

这个配置指明了另一个配置文件，即 `/path/to/my/DIHconfigfile.xml`，这个才是 DIH 的配置文件，该文件指明了 datasource，如何抽取数据，抽取哪些数据，如何处理数据

可以指定多个 DIH 配置文件。（意味着注册多个 DIH?）

此时，重启 solr，testcore 可能会加载失败，异常如下

```java
org.apache.solr.common.SolrException: Error loading class 'org.apache.solr.handler.dataimport.DataImportHandler'
    at org.apache.solr.core.SolrCore.<init>(SolrCore.java:820)
    at org.apache.solr.core.SolrCore.<init>(SolrCore.java:659)
    at org.apache.solr.core.CoreContainer.create(CoreContainer.java:723)
    at org.apache.solr.core.CoreContainer$1.call(CoreContainer.java:443)
    at org.apache.solr.core.CoreContainer$1.call(CoreContainer.java:434)
    at java.util.concurrent.FutureTask$Sync.innerRun(FutureTask.java:334)
    at java.util.concurrent.FutureTask.run(FutureTask.java:166)
    at org.apache.solr.common.util.ExecutorUtil$MDCAwareThreadPoolExecutor$1.run(ExecutorUtil.java:210)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1145)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:615)
    at java.lang.Thread.run(Thread.java:722)
Caused by: org.apache.solr.common.SolrException: Error loading class 'org.apache.solr.handler.dataimport.DataImportHandler'
    at org.apache.solr.core.SolrResourceLoader.findClass(SolrResourceLoader.java:491)
    at org.apache.solr.core.SolrResourceLoader.findClass(SolrResourceLoader.java:422)
    at org.apache.solr.core.SolrCore.createInstance(SolrCore.java:567)
    at org.apache.solr.core.PluginBag.createPlugin(PluginBag.java:122)
    at org.apache.solr.core.PluginBag.init(PluginBag.java:217)
    at org.apache.solr.core.RequestHandlers.initHandlersFromConfig(RequestHandlers.java:130)
    at org.apache.solr.core.SolrCore.<init>(SolrCore.java:773)
    ... 10 more
Caused by: java.lang.ClassNotFoundException: org.apache.solr.handler.dataimport.DataImportHandler
    at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:423)
    at java.net.FactoryURLClassLoader.loadClass(URLClassLoader.java:789)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:356)
    at java.lang.Class.forName0(Native Method)
    at java.lang.Class.forName(Class.java:266)
    at org.apache.solr.core.SolrResourceLoader.findClass(SolrResourceLoader.java:475)
    ... 16 more
```

需要在 solrconfig.xml 里添加jar包，如下

```markup
<lib dir="${solr.install.dir:../../../..}/dist/" regex="solr-dataimporthandler-\d.*\.jar" />
```

此时就能成功完成导入了吗？答案是否定滴。。。异常信息如下

```java
Full Import failed:java.lang.RuntimeException: java.lang.RuntimeException: org.apache.solr.handler.dataimport.DataImportHandlerException: Could not load driver: com.mysql.jdbc.Driver Processing Document # 1
    at org.apache.solr.handler.dataimport.DocBuilder.execute(DocBuilder.java:270)
    at org.apache.solr.handler.dataimport.DataImporter.doFullImport(DataImporter.java:416)
    at org.apache.solr.handler.dataimport.DataImporter.runCmd(DataImporter.java:480)
    at org.apache.solr.handler.dataimport.DataImporter$1.run(DataImporter.java:461)
Caused by: java.lang.RuntimeException: org.apache.solr.handler.dataimport.DataImportHandlerException: Could not load driver: com.mysql.jdbc.Driver Processing Document # 1
    at org.apache.solr.handler.dataimport.DocBuilder.buildDocument(DocBuilder.java:416)
    at org.apache.solr.handler.dataimport.DocBuilder.doFullDump(DocBuilder.java:329)
    at org.apache.solr.handler.dataimport.DocBuilder.execute(DocBuilder.java:232)
    ... 3 more
Caused by: org.apache.solr.handler.dataimport.DataImportHandlerException: Could not load driver: com.mysql.jdbc.Driver Processing Document # 1
    at org.apache.solr.handler.dataimport.DataImportHandlerException.wrapAndThrow(DataImportHandlerException.java:70)
    at org.apache.solr.handler.dataimport.JdbcDataSource.createConnectionFactory(JdbcDataSource.java:158)
    at org.apache.solr.handler.dataimport.JdbcDataSource.init(JdbcDataSource.java:78)
    at org.apache.solr.handler.dataimport.DataImporter.getDataSourceInstance(DataImporter.java:389)
    at org.apache.solr.handler.dataimport.ContextImpl.getDataSource(ContextImpl.java:102)
    at org.apache.solr.handler.dataimport.SqlEntityProcessor.init(SqlEntityProcessor.java:52)
    at org.apache.solr.handler.dataimport.EntityProcessorWrapper.init(EntityProcessorWrapper.java:74)
    at org.apache.solr.handler.dataimport.DocBuilder.buildDocument(DocBuilder.java:433)
    at org.apache.solr.handler.dataimport.DocBuilder.buildDocument(DocBuilder.java:414)
    ... 5 more
Caused by: java.lang.ClassNotFoundException: Unable to load com.mysql.jdbc.Driver or org.apache.solr.handler.dataimport.com.mysql.jdbc.Driver
    at org.apache.solr.handler.dataimport.DocBuilder.loadClass(DocBuilder.java:933)
    at org.apache.solr.handler.dataimport.JdbcDataSource.createConnectionFactory(JdbcDataSource.java:156)
    ... 12 more
Caused by: org.apache.solr.common.SolrException: Error loading class 'com.mysql.jdbc.Driver'
    at org.apache.solr.core.SolrResourceLoader.findClass(SolrResourceLoader.java:491)
    at org.apache.solr.core.SolrResourceLoader.findClass(SolrResourceLoader.java:422)
    at org.apache.solr.handler.dataimport.DocBuilder.loadClass(DocBuilder.java:923)
    ... 13 more
Caused by: java.lang.ClassNotFoundException: com.mysql.jdbc.Driver
    at java.net.URLClassLoader$1.run(URLClassLoader.java:366)
    at java.net.URLClassLoader$1.run(URLClassLoader.java:355)
    at java.security.AccessController.doPrivileged(Native Method)
    at java.net.URLClassLoader.findClass(URLClassLoader.java:354)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:423)
    at java.net.FactoryURLClassLoader.loadClass(URLClassLoader.java:789)
    at java.lang.ClassLoader.loadClass(ClassLoader.java:356)
    at java.lang.Class.forName0(Native Method)
    at java.lang.Class.forName(Class.java:266)
    at org.apache.solr.core.SolrResourceLoader.findClass(SolrResourceLoader.java:475)
    ... 15 more
```

看上去是 mysql 驱动找不到，那么首先执行如下命令下载 mysql 驱动

```bash
cd /data/solr-5.3.1/dist/
wget http://maven.meizu.com:8081/artifactory/repo1-cache/mysql/mysql-connector-java/6.0.2/mysql-connector-java-6.0.2.jar
```

然后修改 solrconfig.xml

```markup
<lib dir="${solr.install.dir:../../../..}/dist/" regex="mysql-connector-java-\d.*\.jar" />
```

### DIH 配置

示例如下

```markup
<dataConfig>

    <!-- dataSource元素，指定了jdbc驱动，连接串，登录信息 -->
    <!-- 还可以指定是否自动提交事务，批处理的尺寸，只读等信息 -->
    <dataSource driver="org.hsqldb.jdbcDriver" 
                url="jdbc:hsqldb:./example-DIH/hsqldb/ex" 
                user="sa" />

    <!-- document元素，包含多个entity元素。entity元素可以嵌套使用 -->
    <document>

        <!-- entity元素可以包括一个或多个field元素，用于映射数据源的字段和solr的字段 -->
        <entity name="item" 
                query="select * from item" 
                deltaQuery="select id from item where last_modified &gt; '${dataimporter.last_index_time}'">
            <field column="NAME" name="name" />

            <!-- 这个内嵌的entity表明了一个item和它的属性之间的1对多关系 -->
            <!-- 注意变量 ${item.ID}，是item的字段“ID”的值 (item是entity的名字) -->
            <entity name="feature" 
                    query="select DESCRIPTION from FEATURE where ITEM_ID='${item.ID}'" 
                    deltaQuery="select ITEM_ID from FEATURE where last_modified &gt; '${dataimporter.last_index_time}'" 
                    parentDeltaQuery="select ID from item where ID=${feature.ITEM_ID}">
                <field name="features" column="DESCRIPTION" />
            </entity>

            <entity name="item_category" 
                    query="select CATEGORY_ID from item_category where ITEM_ID='${item.ID}'" 
                    deltaQuery="select ITEM_ID, CATEGORY_ID from item_category where last_modified &gt; '${dataimporter.last_index_time}'" 
                    parentDeltaQuery="select ID from item where ID=${item_category.ITEM_ID}">
                <entity name="category" 
                        query="select DESCRIPTION from category where ID = '${item_category.CATEGORY_ID}'" 
                        deltaQuery="select ID from category where last_modified &gt; '${dataimporter.last_index_time}'" 
                        parentDeltaQuery="select ITEM_ID, CATEGORY_ID from item_category where CATEGORY_ID=${category.ID}">
                    <field column="description" name="cat" />
                </entity>
            </entity>
        </entity>
    </document>
</dataConfig>
```

## DIH命令

```bash
命令url：http//hostport/solr/collection_name/dataimport?command=${command}
```

### abort

中止一个正在执行的操作

### delta-import

增量导入和变更检测，支持 clean,commit,optimize,debug 等参数，参数说明见 full-import

### full-import

全量导入。该命令会立即返回，操作会异步地执行，同时 status 命令会显示 busy；该命令不会阻塞 solr 的查询。当全量导入开始后，它会将开始时间存储到 `conf/dataimport.properties` 文件，这个时间会用于增量导入。

该命令可以附带如下的参数

* clean
  * 默认为 true。是否在索引前清理索引
* commit
  * 默认为 true。操作完成后是否提交
* debug
  * 默认为 false。是否以调试模式执行。注意：调试模式下 commit 默认为 false。
* entity
  * 配置文件里定义的实体名称，可以指定多个实体名称。如果不指定则导入所有实体。
* optimize
  * 默认为 true。操作完成后是否优化

### **reload-config**

修改了配置文件而不想重启 solr，执行这个命令

### **status**

返回统计信息和状态信息，如文档创建数，删除数，运行的查询，返回的行数等等

## property writer

配置文件 dataConfig 节点的子节点，用于增量导入里的日期格式、地区等设置，该节点是可选的。

```markup
<propertyWriter 
    dateFormat="yyyy-MM-dd HH:mm:ss" 
    type="SimplePropertiesWriter"
    directory="data" 
    filename="my_dih.properties" 
    locale="en_US" />
```

属性说明

* dateFormat
* type

  指定了实现类，非 cloud 模式，使用 SimplePropertiesWriter，cloud 模式，使用 ZKPropertiesWriter；如果未指定，会根据是否开启了 cloud 模式自动选择

* directory

  仅用于 SimplePropertiesWriter，配置文件所在的目录，如果未指定，默认为 `conf`

* filename

  仅用于 SimplePropertiesWriter，配置文件名，如果未指定，默认为 requestHandler（见 solrconfig.xml）的名字，以 `.properties` 为后缀名

* locale

  时区

## Data Source

继承 `org.apache.solr.handler.dataimport.DataSource` 来自定义 Data Source

## Entity Processors

实体处理机，抽取，转换数据，并添加到 solr 进行索引。

### SQL Entity Processor

* query

  必须，查询数据的 sql

* deltaQuery

  delta-import 命令使用的 sql。该查询返回更新过的记录的主键，deltaImportQuery 通过变量  `${dataimporter.delta.<column-name>}` 访问这些主键

* parentDeltaQuery

  用于 delta-import 的 sql

* deletedPkQuery

  用于 delta-import 的 sql

* deltaImportQuery

  用于 delta-import 的 sql，如果未指定，DIH 会尝试构造 import 查询？

  命名空间 `${dataimporter.delta.<column-name>}` 可被用于此查询，例如 `select * from tbl where id=${dataimporter.delta.id}`

## Transformers

处理文档的字段。transformer 可以创建新字段，或者修改已存在的，需要在 entity 元素里指定 transformer

```markup
<entity name="abcde" transformer="org.apache.solr....,my.own.transformer,..." />
```

