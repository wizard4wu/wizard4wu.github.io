---
title: GitHub构建Maven依赖仓库
key: AAA-2022-09-10-GithubwithMaven
tags: [Maven]
comment: true
footer: true
show_edit_on_github: false
pageview: true
lightbox: true
aside:
toc: true
show_subscribe: false
lightbox: true
---

吐槽一句：博客氛围太差了，一篇博客到处发，而且自己还没有实践，直接带到沟里了。

### 1.基础构建

+ Github新建仓库

![image-20220909201755617]({{"/assets/picture/2022-09/image-20220909201755617.png" | absolute_url}})
+ 本地初始化git目录 初始化命令 `git init`

![image-20220909202238579]({{"/assets/picture/2022-09/image-20220909202238579.png" | absolute_url}})


+ 将远程的仓库和本地git目录建立连接 `git remote add origin git@github.com:Encyclopedias/MavenRepo.git`
+ 新建项目，更改maven的项目的pom文件，如下：

```java
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <groupId>org.util.maven</groupId>
    <artifactId>maven-repo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>8</maven.compiler.source>
        <maven.compiler.target>8</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>
</project>
```
+ 添加测试类和方法

```java
public class Util {
    public void helloWorld(){
        System.out.println("HelloWorld");
    }
}
```

### 2.构建本地maven依赖

+ 添加maven安装依赖的本地目录

```java
在上述的POM文件里面添加如下配置：
     <distributionManagement>
            <repository>
                <id>maven-repo</id>
                <url>file:${basedir}/repo</url>  <--该路径会在项目下生成一个repo的目录，该目录内会有jar包-->
            </repository>
    </distributionManagement>
```

+ 执行`mvn clean deploy`命令后该项目下会生成一个repo的目录，同时在你的本地仓库也会生成一个目录

我的项目结构：

![image-20220909202238579]({{"/assets/picture/2022-09/image-20220909212106188.png" | absolute_url}})

本地仓库生成的目录如下：
![image-20220909202238579]({{"/assets/picture/2022-09/image-20220909211938748.png" | absolute_url}})
此时你可以在别的项目中添加如下依赖即可访问：

```java
<dependency>
   <groupId>org.util.maven</groupId>
   <artifactId>maven-repo</artifactId>
   <version>1.0-SNAPSHOT</version>
</dependency>
```

### 3.构建远程maven依赖

+ 使用git提交项目到远程 由于上面我们已经将本地项目和远程做了关联，因此此处只管提交即可。

![image-20220909214055955]({{"/assets/picture/2022-09/image-20220909214055955.png" | absolute_url}}
)

+ 在需要引用的项目的POM文件中配置

```java
 <repositories>
        <repository>
            <id>maven-repo</id>
            <url>https://raw.githubusercontent.com/Encyclopedias/MavenRepo/main/repo</url>
        </repository>
    </repositories>
```

+ 基于该项目建一个子maven项目，然后在子的maven项目中配置，然后刷新，这时候依赖才会被下载

```java
<dependency>
   <groupId>org.util.maven</groupId>
   <artifactId>maven-repo</artifactId>
</dependency>
```



<font color=red size = 10>遇到的问题：</font>
刷新后远程的依赖可以通过连接访问，但是该依赖在项目中下载不下来

通常情况下，如果你在pom文件中添加了github的repository的话，会在maven的仓库会显示，下述是我自己添加的两个GitHub远程的maven仓库
![image-20220909220622882]({{"/assets/picture/2022-09/image-20220909220622882.png" | absolute_url}}
)

如果项目下载不下来并且上述也没有你配置的远程的github的仓库url的话，很有可能你的settings.xml文件出现了问题。

我的就是因为settings的文件配置出错了。
右击项目->maven->open ‘settings.xml’

```
  <mirrors>
        <mirror>
            <id>alimaven</id>
            <mirrorOf>central</mirrorOf>
            <name>aliyun maven</name>
            <url>http://maven.aliyun.com/nexus/content/repositories/central/</url>
        </mirror>
        <mirror>
            <id>nexus-163</id>
            <mirrorOf>*</mirrorOf>
            <name>Nexus 163</name>
            <url>http://mirrors.163.com/maven/repository/maven-public/</url>
        </mirror>
    </mirrors>
我当时下载依赖的时候发现使用nexus-163拼接了我的依赖包的路径的，而不是github的url，后来发现使用mirrorOf的标签的原因上述配置的是*，这就会导致都是通过所有的都需要通过nexus-163的url去仓库下载依赖，那我的是github的依赖，当然没有了啊。
```



