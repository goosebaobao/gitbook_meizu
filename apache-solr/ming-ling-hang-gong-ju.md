# 命令行工具

## bin/solr 命令

```bash
# bin/solr -help

Usage: solr COMMAND OPTIONS
       where COMMAND is one of: start, stop, restart, status, healthcheck, create, create_core, create_collection, delete

  Standalone server example (start Solr running in the background on port 8984):

    ./solr start -p 8984

  SolrCloud example (start Solr running in SolrCloud mode using localhost:2181 to connect to ZooKeeper, with 1g max heap size and remote Java debug options enabled):

    ./solr start -c -m 1g -z localhost:2181 -a "-Xdebug -Xrunjdwp:transport=dt_socket,server=y,suspend=n,address=1044"

Pass -help after any COMMAND to see command-specific usage information,
  such as:    ./solr start -help or ./solr stop -help
```

COMMAND

* start
* stop
* restart
* status
* healthcheck
* create
* create\_core
* create\_collection
* delete

### solr start

```bash
# bin/solr start -help

Usage: solr start [-f] [-c] [-h hostname] [-p port] [-d directory] [-z zkHost] [-m memory] [-e example] [-s solr.solr.home] [-a "additional-options"] [-V]

  -f            Start Solr in foreground; default starts Solr in the background
                  and sends stdout / stderr to solr-PORT-console.log

  -c or -cloud  Start Solr in SolrCloud mode; if -z not supplied, an embedded ZooKeeper
                  instance is started on Solr port+1000, such as 9983 if Solr is bound to 8983

  -h <host>     Specify the hostname for this Solr instance

  -p <port>     Specify the port to start the Solr HTTP listener on; default is 8983
                  The specified port (SOLR_PORT) will also be used to determine the stop port
                  STOP_PORT=($SOLR_PORT-1000) and JMX RMI listen port RMI_PORT=(1$SOLR_PORT).
                  For instance, if you set -p 8985, then the STOP_PORT=7985 and RMI_PORT=18985

  -d <dir>      Specify the Solr server directory; defaults to server

  -z <zkHost>   ZooKeeper connection string; only used when running in SolrCloud mode using -c
                   To launch an embedded ZooKeeper instance, don't pass this parameter.

  -m <memory>   Sets the min (-Xms) and max (-Xmx) heap size for the JVM, such as: -m 4g
                  results in: -Xms4g -Xmx4g; by default, this script sets the heap size to 512m

  -s <dir>      Sets the solr.solr.home system property; Solr will create core directories under
                  this directory. This allows you to run multiple Solr instances on the same host
                  while reusing the same server directory set using the -d parameter. If set, the
                  specified directory should contain a solr.xml file, unless solr.xml exists in ZooKeeper.
                  This parameter is ignored when running examples (-e), as the solr.solr.home depends
                  on which example is run. The default value is server/solr.

  -e <example>  Name of the example to run; available examples:
      cloud:         SolrCloud example
      techproducts:  Comprehensive example illustrating many of Solr's core capabilities
      dih:           Data Import Handler
      schemaless:    Schema-less example

  -a            Additional parameters to pass to the JVM when starting Solr, such as to setup
                  Java debug options. For example, to enable a Java debugger to attach to the Solr JVM
                  you could pass: -a "-agentlib:jdwp=transport=dt_socket,server=y,suspend=n,address=18983"
                  In most cases, you should wrap the additional parameters in double quotes.

  -noprompt     Don't prompt for input; accept all defaults when running examples that accept user input

  -V            Verbose messages from this script
```

OPTIONS 

* -f：在前台启动 solr，`ctrl+c` 用来停止 solr；默认情况下，solr 在后台启动，并会将 stdout/stderr 输出到 solr-PORT-console.log，例如，如果端口为 8983，则该日志文件为 solr-8983-console.log 
* -c 或 -cloud：以 SolrCloud 模式启动 solr，如果 `-z` 未提供，一个内建的 ZooKeeper 实例会在 solr 的端口 +1000 上启动，例如，solr 端口为 8983，则该 zk 的端口为 9983 
* -h &lt;host&gt;：指定该 solr 实例的 hostname 
* -p &lt;port&gt;：指定 solr 的 http 监听端口，默认值为 8983，同时，以下端口也会相应绑定：停止端口（-1000），jmx rmi 端口（1\*\*\*\***，\*\*\*\*** 是 http 监听端口） 
* -d &lt;dir&gt;：指定 solr 服务器目录，默认目录是 server 
* -z &lt;zkHost&gt;：Zookeeper 连接串，仅当 solr 以 SolrCloud 模式运行时使用。不指定该选项，则使用内建的 Zookeeper
* -m &lt;memory&gt;：设定 jvm 堆的最小（-Xms）和最大\(-Xmx\)值为 memory，即 `-Xms=-Xmx=memory`，默认为 512m 
* -s &lt;dir&gt;：设定 `solr.solr.home` 环境变量，solr 在该目录下创建 core 目录；这样就可以在一个主机上运行多个 solr 实例。如果设置该选项，在指定的目录下应该有一个 solr.xml 文件，除非该文件在 zookeeper 上。如果使用了选项 `-e`，则该选项将被忽略。默认值是 server/solr 
* -e &lt;example&gt;：运行示例，可用的示例有
  *  cloud：SolrCloud 示例 
  * techproducts 
  * dih：Data Import Handler 
  * schemaless 
* -a：启动 solr 时需要传递给 jvm 的额外参数，这些参数多数情况下应该用双引号引起来 
* -noprompt：不提示输入 
* -V：显示详细的信息

### solr stop

```bash
# bin/solr stop -help

Usage: solr stop [-k key] [-p port] [-V]

  -k <key>      Stop key; default is solrrocks

  -p <port>     Specify the port the Solr HTTP listener is bound to

  -all          Find and stop all running Solr servers on this host

  NOTE: To see if any Solr servers are running, do: solr status
```

### solr status

在当前机器上查找正在运行的 solr 实例，并返回其基本信息

```bash
# bin/solr status

Found 1 Solr nodes:

Solr process 5437 running on port 8983
{
  "solr_home":"/data/solr-5.3.1/server/solr/",
  "version":"5.3.1 1703449 - noble - 2015-09-17 01:48:15",
  "startTime":"2016-02-03T02:37:58.973Z",
  "uptime":"0 days, 0 hours, 34 minutes, 13 seconds",
  "memory":"85.4 MB (%17.4) of 490.7 MB"}
```

### solr create

创建一个 core 或者 collection，基于 solr 的运行模式：standalone（core）或者 SolrCloud（collection） 模式。

换言之，该操作会检查 solr 的运行模式，并使用合适的操作（create\_core或者create\_collection）

```bash
# bin/solr create -help

Usage: solr create [-c name] [-d confdir] [-n configName] [-shards #] [-replicationFactor #] [-p port]

  Create a core or collection depending on whether Solr is running in standalone (core) or SolrCloud
  mode (collection). In other words, this action detects which mode Solr is running in, and then takes
  the appropriate action (either create_core or create_collection). For detailed usage instructions, do:

    bin/solr create_core -help

       or

    bin/solr create_collection -help
```

#### solr create\_core

选项 

* -c &lt;core&gt; ：要创建的 core 的名称 
* -d &lt;confdir&gt;：创建一个新的 core 时从中复制配置信息的目录。可以使用内建的配置，或者指定一个自己的配置目录 
  * 内建配置，如果不指定默认采用 data\_driven\_schema\_configs 
    * basic\_configs: 最小化的 solr 配置 
    * data\_driven\_schema\_configs: Managed schema with field-guessing support enabled 
    * sample\_techproducts\_configs: 示例配置，有多项可选的特性以用来演示 solr 的全部功能
* -p &lt;port&gt;：想要创建 core 的 solr 实例的端口，如果不指定的话，会自动查找运行中的 solr 实例并在第一个被找到的实例下创建该 core

```bash
# bin/solr create_core -help

Usage: solr create_core [-c core] [-d confdir] [-p port]

  -c <core>     Name of core to create

  -d <confdir>  Configuration directory to copy when creating the new core, built-in options are:

      basic_configs: Minimal Solr configuration
      data_driven_schema_configs: Managed schema with field-guessing support enabled
      sample_techproducts_configs: Example configuration with many optional features enabled to
         demonstrate the full power of Solr

      If not specified, default is: data_driven_schema_configs

      Alternatively, you can pass the path to your own configuration directory instead of using
      one of the built-in configurations, such as: bin/solr create_core -c mycore -d /tmp/myconfig

  -p <port>     Port of a local Solr instance where you want to create the new core
                  If not specified, the script will search the local system for a running
                  Solr instance and will use the port of the first server it finds.
```

#### solr create\_collection

选项

* -c  &lt;core&gt;：要创建的 collection 的名称 
* -d  &lt;confdir&gt;：创建一个新的 collection 时从中复制配置信息的目录。可以使用内建的配置，或者指定一个自己的配置目录
  * 内建配置，如果不指定默认采用 data\_driven\_schema\_configs 

    * basic\_configs: 最小化的 solr 配置 
    * data\_driven\_schema\_configs: Managed schema with field-guessing support enabled 
    * sample\_techproducts\_configs: 示例配置，有多项可选的特性以用来演示 solr 的全部功能

    默认情况下，脚本会将配置目录上传到 zookeeper，使用和 collection 一样的名字。如果想要重用一个已存在的目录或在 zookeeper 里创建一个可被多个 collection 共享的配置目录，使用 -n 选项 
* -n  &lt;configName&gt;：在 zookeeper 里为配置目录命名，默认会和 collection 同名 
* -shards &lt;\#&gt;：将该 collection 切割为几个 shard，默认是 1 个 
* -replicationFactor &lt;\#&gt;：每个 collection 里的文档有几个 copy，默认是 1（即没有副本 replication） 
* -p &lt;port&gt;：想要创建 collection 的 solr 实例的端口，如果不指定的话，会自动查找运行中的 solr 实例并在第一个被找到的实例下创建该 collection

```bash
# bin/solr create_collection -help

Usage: solr create_collection [-c collection] [-d confdir] [-n configName] [-shards #] [-replicationFactor #] [-p port]

  -c <collection>         Name of collection to create

  -d <confdir>            Configuration directory to copy when creating the new collection, built-in options are:

      basic_configs: Minimal Solr configuration
      data_driven_schema_configs: Managed schema with field-guessing support enabled
      sample_techproducts_configs: Example configuration with many optional features enabled to
         demonstrate the full power of Solr

      If not specified, default is: data_driven_schema_configs

      Alternatively, you can pass the path to your own configuration directory instead of using
      one of the built-in configurations, such as: bin/solr create_collection -c mycoll -d /tmp/myconfig

      By default the script will upload the specified confdir directory into ZooKeeper using the same
      name as the collection (-c) option. Alternatively, if you want to reuse an existing directory
      or create a confdir in ZooKeeper that can be shared by multiple collections, use the -n option

  -n <configName>         Name the configuration directory in ZooKeeper; by default, the configuration
                            will be uploaded to ZooKeeper using the collection name (-c), but if you want
                            to use an existing directory or override the name of the configuration in
                            ZooKeeper, then use the -c option.

  -shards <#>             Number of shards to split the collection into; default is 1

  -replicationFactor <#>  Number of copies of each document in the collection, default is 1 (no replication)

  -p <port>               Port of a local Solr instance where you want to create the new collection
                            If not specified, the script will search the local system for a running
                            Solr instance and will use the port of the first server it finds.
```

## bin/post 命令

solr 用于查找匹配搜索的文档，没有文档啥也找不到，在 solr 能做什么之前，他需要输入。

bin/post 可以提交多种类型的数据到 solr，包括 solr 自身的 xml，json 格式文件，csv，富文本文件的目录，甚至是简单的 web 爬取

```bash
# bin/post -help

Usage: post -c <collection> [OPTIONS] <files|directories|urls|-d ["...",...]>
    or post -help

   collection name defaults to DEFAULT_SOLR_COLLECTION if not specified

OPTIONS
### ====
  Solr options:
    -url <base Solr update URL> (overrides collection, host, and port)
    -host <host> (default: localhost)
    -p or -port <port> (default: 8983)
    -commit yes|no (default: yes)

  Web crawl options:
    -recursive <depth> (default: 1)
    -delay <seconds> (default: 10)

  Directory crawl options:
    -delay <seconds> (default: 0)

  stdin/args options:
    -type <content/type> (default: application/xml)

  Other options:
    -filetypes <type>[,<type>,...] (default: xml,json,csv,pdf,doc,docx,ppt,pptx,xls,xlsx,odt,odp,ods,ott,otp,ots,rtf,htm,html,txt,log)
    -params "<key>=<value>[&<key>=<value>...]" (values must be URL-encoded; these pass through to Solr update request)
    -out yes|no (default: no; yes outputs Solr response to console)


Examples:

* JSON file: bin/post -c wizbang events.json
* XML files: bin/post -c records article*.xml
* CSV file: bin/post -c signals LATEST-signals.csv
* Directory of files: bin/post -c myfiles ~/Documents
* Web crawl: bin/post -c gettingstarted http://lucene.apache.org/solr -recursive 1 -delay 1
* Standard input (stdin): echo '{commit: {}}' | bin/post -c my_collection -type application/json -out yes -d
* Data as string: bin/post -c signals -type text/csv -out yes -d $'id,value\n1,0.47'
```

## 实战

启动 solr，创建一个 core

```bash
# bin/solr create -c testcore

Setup new core instance directory:
/data/solr-5.3.1/server/solr/testcore

Creating new core 'testcore' using command:
http://localhost:8983/solr/admin/cores?action=CREATE&name=testcore&instanceDir=testcore

{
  "responseHeader":{
    "status":0,
    "QTime":1633},
  "core":"testcore"}
```

添加文档

```bash
# bin/post -c testcore example/exampledocs/*.xml
/data/java/jdk1.7.0_15/bin/java -classpath /data/solr-5.3.1/dist/solr-core-5.3.1.jar -Dauto=yes -Dc=testcore -Ddata=files org.apache.solr.util.SimplePostTool example/exampledocs/gb18030-example.xml example/exampledocs/hd.xml example/exampledocs/ipod_other.xml example/exampledocs/ipod_video.xml example/exampledocs/manufacturers.xml example/exampledocs/mem.xml example/exampledocs/money.xml example/exampledocs/monitor2.xml example/exampledocs/monitor.xml example/exampledocs/mp500.xml example/exampledocs/sd500.xml example/exampledocs/solr.xml example/exampledocs/utf8-example.xml example/exampledocs/vidcard.xml
SimplePostTool version 5.0.0
Posting files to [base] url http://localhost:8983/solr/testcore/update...
Entering auto mode. File endings considered are xml,json,csv,pdf,doc,docx,ppt,pptx,xls,xlsx,odt,odp,ods,ott,otp,ots,rtf,htm,html,txt,log
POSTing file gb18030-example.xml (application/xml) to [base]
POSTing file hd.xml (application/xml) to [base]
POSTing file ipod_other.xml (application/xml) to [base]
POSTing file ipod_video.xml (application/xml) to [base]
POSTing file manufacturers.xml (application/xml) to [base]
POSTing file mem.xml (application/xml) to [base]
POSTing file money.xml (application/xml) to [base]
POSTing file monitor2.xml (application/xml) to [base]
POSTing file monitor.xml (application/xml) to [base]
POSTing file mp500.xml (application/xml) to [base]
POSTing file sd500.xml (application/xml) to [base]
POSTing file solr.xml (application/xml) to [base]
POSTing file utf8-example.xml (application/xml) to [base]
POSTing file vidcard.xml (application/xml) to [base]
14 files indexed.
COMMITting Solr index changes to http://localhost:8983/solr/testcore/update...
Time spent: 0:00:00.638
```

