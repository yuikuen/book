# VsCode SpringBoot 

## JDK 安装

可选择 OpenJDK 或 OracleJDK

- **OpenJDK**

1. [官网](https://www.openlogic.com/openjdk-downloads)
2. [清华大学开源软件镜像站](https://mirrors.tuna.tsinghua.edu.cn/AdoptOpenJDK/8/jdk/x64/windows/)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221120055668.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221120301777.png)

- **OracleJDK**

1. [官网](https://www.oracle.com/java/technologies/downloads/)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221133546365.png)

1）选择 msi & exe，一路安装

2）修改环境变量，我的电脑→属性→高级系统设置→高级选项→环境变量

- 系统变量处点击新建系统变量–变量名： `JAVA_HOME`，变量值：`C:\Program Files\Java\jdk1.8.0_321`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221135002345.png)

- 再次新建系统变量-变量名： `CLASSPATH`，变量值：`.;%JAVA_HOME%\lib\dt.jar;%JAVA_HOME%\lib\tools.jar;`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221135524774.png)

- 找到用户变量 PATH，点击编辑并新建两个变量值：`%JAVA_HOME%\bin` `%JAVA_HOME%\jre\bin`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221140146796.png)

3）验证，打开 cmd 下输入 `java -version 或 javac`

```cmd
$ java -version
java version "1.8.0_261"
Java(TM) SE Runtime Environment (build 1.8.0_261-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.261-b12, mixed mode)
```

## Maven 安装

1）下载程序，选择 [bin.zip](https://maven.apache.org/download.cgi) 后缀文件下载，解压到本地

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221141047035.png)

2）配置环境变量

- MAVEN_HOME

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221141811291.png)

- PATH

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221141555797.png)

- 验证 `mvn -v`

```cmd
$ mvn -v
Apache Maven 3.8.4 (9b656c72d54e5bacbed989b64718c159fe39b537)
Maven home: C:\Program Files\apache-maven-3.8.4
Java version: 1.8.0_261, vendor: Oracle Corporation, runtime: C:\Program Files\Java\jdk1.8.0_261\jre
Default locale: zh_CN, platform encoding: GBK
OS name: "windows 10", version: "10.0", arch: "amd64", family: "windows"
```

3）配置仓库

- 本地仓库

创建目录并修改配置文件，`.\apache-maven-3.8.4\conf\settings` (需要具体访问权限)

```xml
<localRepository>C:\Program Files\apache-maven-3.8.4\repository</localRepository>
```

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221143653457.png)

- 中央仓库，添加 [阿里云maven仓库](https://developer.aliyun.com/mvn/guide)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221144415645.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221145314721.png)

```xml
    <mirror>
      <id>aliyunmaven</id>
      <mirrorOf>*</mirrorOf>
      <name>阿里云公共仓库</name>
      <url>https://maven.aliyun.com/repository/public</url>
    </mirror>
```

## Git 安装

1. [官网](https://git-scm.com/)
2. [清华源](https://mirrors.tuna.tsinghua.edu.cn/github-release/git-for-windows/git/LatestRelease/)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221145515599.png)

## SpringBoot 搭建

1）安装插件

在 Visual Studio Code 中打开扩展视图(Ctrl+Shift+X)，搜索并安装

- Java Extension Pack (Java 扩展包)
- Spring Boot Extension Pack

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221145837536.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221152233092.png)

2）配置环境(JDK、Maven)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221152534220.png)

把 maven 的可执行文件路径配置、maven的 setting 路径配置、java.home 的路径配置

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221153137855.png)

```
{
    "security.workspace.trust.untrustedFiles": "open",
    "workbench.startupEditor": "newUntitledFile",
    "java.errors.incompleteClasspath.severity": "ignore",
    "workbench.colorTheme": "Visual Studio Dark",
    "java.home":"C:\\Program Files\\Java\\jdk1.8.0_261",
    "java.configuration.maven.userSettings": "C:\\Program Files\\apache-maven-3.8.4\\conf\\settings.xml",
    "maven.executable.path": "C:\\Program Files\\apache-maven-3.8.4\\bin\\mvn.cmd",
    "maven.terminal.useJavaHome": true,
    "maven.terminal.customEnv": [
        {
            "environmentVariable": "JAVA_HOME",
            "value": "C:\\Program Files\\Java\\jdk1.8.0_261"
        }
    ],
    "extensions.autoUpdate": false,
}
```

配置完成重启 VSCode

3）创建 SpringBoot 项目

使用快捷键(Ctrl+Shift+P)命令窗口，输入 Spring 选择创建 Maven 项目 `Spring Initializr:Create a Maven Project...`

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221153901836.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221154026593.png)

选择 java

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221154103719.png)

设置包名

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221154142247.png)

设置项目名

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221154156150.png)

选择 jar --> jdk 选择 1.8 --> 选择依赖

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221155142385.png)

4）修改配置文件

将 application.properties 改为 yml格式，并添加数据库配置

```yaml
spring:
  datasource:  # 配置mysql
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/mydb?characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull&allowMultiQueries=true&serverTimezone=Asia/Shanghai&useSSL=false
    username: root
    password: 123456
    
server:
  port: 8080  # 默认端口
  
mybatis:
  mapper-locations: classpath:mapper/*.xml 

```

5）运行项目，选择 DemoApplication.java 文件，右键菜单 运行(第一次运行要下载相关maven依赖，会比较慢)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221160842429.png)

**异常提示**

1）右下角会弹出一个弹框

`Java 11 or more recent is required to run. Please download and install a recent JDK.
Source: Language Support for Java™ by Red Hat`

错误是**Language Support for Java™ by Red Hat**这个插件报出来的，其原因是这个插件鼓励开发者们使用 Java 11或者更新的版本。在v0.64.1这个版本更新中，这个插件将 Java 11作为运行版本

解决方案：选择降级到v0.64.1之前，同时关闭插件自动更新

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221155953950.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221160134291.png)

## Git 克隆项目

使用快捷键(Ctrl+Shift+P)命令窗口，输入 git clone

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221161249590.png)

输入项目地址，选择工作区，待下载完进行运行测试

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221161414983.png)

![](https://yuikuen-1259273046.cos.ap-guangzhou.myqcloud.com/devops/image-20220221162027244.png)