---
description: by 陈健森 at 2016.9
---

# Maven 引用 lib 下的 jar 包

## 适用场景

在项目开发中，项目用 maven 管理，是一个 maven 项目。

一般情况下 jar 包都可以使用 pom.xml 来配置管理，但也有一些时候，我们项目中使用了一个内部 jar 文件，但是这个文件我们又没有开放到 maven 库中，我们会将文件当到我们项目 WEB-INF/lib 中。

如果我们不对 pom.xml 进行特殊配置的话，maven 打包是不会自动去引用和编译 lib 中的 jar 文件的，所以需要我们修改下 pom.xml 文件。

## 把某个 jar 包加入 maven 寻找的路径

```markup
<dependency> 
  <groupId>com.AXMLPrinter</groupId> 
  <artifactId>AXMLPrinter2</artifactId> 
  <version>1.0</version> 
  <scope>system</scope> 
  <systemPath>${project.basedir}/src/main/webapp/WEB-INF/lib/AXMLPrinter2.jar</systemPath> 
</dependency>
```

* `<type>`  package 类型，一般不用写，默认是 jar。还有，jar/war/pom/maven-plugin/ear 等。
* `<scope>` 依赖范围，我简记为“能见度”。
  * `compile` 默认的 scope，表示 dependency 都可以在生命周期中使用。而且，这些 dependencies 会传递到依赖的项目中。适用于所有阶段，会随着项目一起发布
  * `provided` 跟 compile 相似，但是表明了 dependency 由 JDK 或者容器提供，例如 Servlet API 和一些 Java EE APIs。这个 scope 只能作用在编译和测试时，同时没有传递性。
  * `runtime` 表示 dependency 不作用在编译时，但会作用在运行和测试时，如 JDBC 驱动，适用运行和测试阶段。
  * `test` 表示 dependency 作用在测试时，不作用在运行时。只在测试时使用，用于编译和运行测试代码。不会随项目发布。
  * `system` 跟 provided 相似，但是在系统中要以外部 JAR 包的形式提供，maven 不会在 repository 查找它。

可是，这种的不好处是，只能加入某个 jar 包而不是某个目录。

## 使用 lib 目录下面所有的 jar 包

修改 maven-compiler-plugin 的配置，如下：

```markup
<plugin> 
<groupId>org.apache.maven.plugins</groupId> 
<artifactId>maven-compiler-plugin</artifactId>
<version>3.1</version> 
<configuration> 
    <target>1.7</target>
    <encoding>UTF-8</encoding>
    <compilerArguments> 
      .....
      <extdirs>${project.basedir}/src/main/webapp/WEB-INF/lib</extdirs>
    </compilerArguments> 
</configuration>
</plugin>
```

其中 `${project.basedir}` 一定要写，不然会出现在 windows 下可以正常编译，在 Linux 服务器上就有可能出现编译找不到 jar 包的错误。

