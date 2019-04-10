# FlymeTV task 部署

## 安装 launcher

{% hint style="info" %}
如果 launcher 已安装可以跳过本步骤
{% endhint %}

下载 commons launcher 

```bash
wget http://maven.meizu.com:8081/artifactory/Internet-release-local/com/meizu/rpm/commons-launcher-1.1/1.0/commons-launcher-1.1-1.0.rpm
```

执行 rpm 安装 

```bash
rpm -i commons-launcher-1.1-1.0.rpm
```

 安装完毕，发现 /data/ 下已建好 launcher 目录，里面有个 commons-launcher 的子目录，继续进入，有个 app 的子目录

## 下载 task 包

我们把 task 部署到 /data/commons-launcher/app 目录下，执行命令进入该目录

```bash
cd /data/launcher/commons-launcher/app
```

把 task 包下载回来，执行如下命令

```bash
wget http://maven.meizu.com:8081/artifactory/libs-release-local/com/meizu/flymetv-video-task/1.0.13/flymetv-video-task-1.0.13.zip
```

解压缩该 zip 包

```bash
unzip flymetv-video-task-1.0.13.zip
```

 解压缩完成后，app 目录下多了一个 lib 目录，这就是 task 的所有 jar 包了；另外还有一个 conf 目录，这里保存的是 log4j 的配置文件 log4j.xml

{% hint style="info" %}
为了达成这个效果，在工程的 src 目录下应该存在 conf 目录及 log4j.xml 文件
{% endhint %}

## 配置 launcher

接下来需要配置一下 launcher.xml，执行如下命令

```bash
vim /data/launcher/commons-launcher/bin/launcher.xml
```

 找到

```markup
<sysproperty key="spring.context.config" 
    value="classpath:task/app-task-spring-context.xml" />
```

默认的 spring 配置文件名是 task/app-task-spring-context.xml，如果你的配置文件不是这个名字，就修改为实际的名字。由于 FlymeTV 的 task 是默认命名，这里我们不做任何修改。

{% hint style="info" %}
为了方便运维和测试同学部署，建议 task/consumer 的配置文件名就用默认的命名
{% endhint %}

## 拉起launcher

执行如下命令

```bash
/data/launcher/commons-launcher/bin/start.sh
```

 bin 下有个 nohup.out，查看这个文件来验证 task 已启动

## 配置日志输出路径

默认的 launcher.xml，貌似无法按规范输出日志，这里提供一个正确的配置文件

```markup
<project name="Demo Launcher" default="task" basedir=".">

        <!-- Set the application home to the parent directory of this directory -->
        <property name="app.home" location="${basedir}/../app" />

        <property name="app.base.dir" location="${basedir}/../app"/>
        <property name="app.conf.dir" value="${app.base.dir}/conf" />
        <property name="app.lib.dir" value="${app.base.dir}/lib" />
        <property name="app.log.dir" value="/data/log/task"/>

        <path id="base.class.path">
                <fileset dir="${basedir}/../lib" includes="*.jar" />
                <pathelement path="${app.conf.dir}" />
                <pathelement path="${app.lib.dir}" />
                <fileset dir="${app.lib.dir}" includes="*.jar" />
        </path>

        <jvmargset id="base.jvm.args">
                <jvmarg value="-Xrunjdwp:transport=dt_socket,address=61009,server=y,suspend=n" />
        </jvmargset>

        <target name="task">
                <mkdir dir="${app.log.dir}" />
                <launch classname="com.meizu.framework.task.StandardTask" output="${app.log.dir}/ant.log">
                        <jvmargset refid="base.jvm.args" />
                        <classpath refid="base.class.path" />
                        <syspropertyset>
                                <sysproperty key="spring.context.config" value="classpath:task/app-task-spring-context.xml" />
                                <sysproperty key="log.dir" file="${app.log.dir}"/>
                        </syspropertyset>
                </launch>
        </target>

</project>
```

## 后记

安装好 launcher 后，bin 下的 task.sh 无法执行，报错

```bash
 -bash: ./task.sh: /bin/sh^M: bad interpreter: No such file or directory
```

 网上找了一下资料发现是因为文件格式为 dos 导致的，在 vi 里使用 `set ff=unix` 将文件格式修改为 unix 即可

