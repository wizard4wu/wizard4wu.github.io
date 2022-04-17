---

title: Spring/Spring-Boot源码阅读环境搭建
key: AAA-2022-04-2-spring-source-code-read-environment
tags: [Spring]
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

## 1. 初衷

最近突然有正式看源码的想法了，之前都是听别人讲，有时候也在项目里面看看，可是有时候发现被依赖的包的代码搜索不到，只能搜索类名，于是便有了下载源码的想法。下载源码并不是说要一个方法一个方法去看，我计划是从某个注解或者某种机制原理下手，比如@RequestMapping是怎么工作的。

## 2. 准备

由于spring和spring-boot高版本jdk 1.8是支持不了的，需要更高版本的jdk，可以[下载对应版本的JDK](https://www.oracle.com/java/technologies/downloads/archive/)

同时，高版本是需要Gradle来构建工程，可以自己下载也可以不下载。

你可能会遇到如下错误`A build scan was not published as you have not authenticated with server 'ge.spring.io'.`

```
找到build.gradle文件然后将替换成如下：
repositories {
  mavenCentral()
  maven { url "https://repo.spring.io/libs-spring-framework-build" }
	maven{ url 'http://maven.aliyun.com/nexus/content/groups/public/'}
}
添加一个远程的仓库源 方便下载。
```

```
dependencies {
    compile(project(":spring-context"))
}
```

## 3. 搭建的环境

你可以使用我已经搭建好的，Spring使用的是5.0版本，SpringBoot使用的是2.0版本。

### 3.1Spring

[项目链接](https://github.com/Encyclopedias/my-spring)

git clone git@github.com:Encyclopedias/my-spring.git 获取我的项目，我也是fork Spring官网的项目

下载完项目后使用`git checkout -b spring-5 origin/5.0.x`切换到5.0版本

![image-20220417113613402]({{"/assets/picture/2022-04/spring.png" | absolute_url}})

在5.0版本中有我新建的模块spring-helloworld,然后找到该模块下的`HelloWorldDemo`的类运行，如果输出正常表明你的Spring环境是正常的；然后还有一个新建的模块spring-research,找到该模块下`EmbedTomcatApplicationStarter`的类运行，该类运行后会启动一个内嵌的Tomcat，有助于你去研究Spring，因为这时候你可以通过请求的方式一步步debug。

运行的时候需要注意一些工作目录，因为Spring项目下有很多模块，所以你要选择当前的模块去运行，不然在寻找路径会出现问题
![image-20220417120900702]({{"/assets/picture/2022-04/workmodule.png" | absolute_url}})

### 3.2 SpringBoot

[项目链接](https://github.com/Encyclopedias/my-springboot)

git clone [git@github.com](mailto:git@github.com):Encyclopedias/my-springboot.git 获取项目

下载完项目后使用`git checkout -b springboot-2 origin/2.0.x`切换到2.0版本

![image-20220417114055767]({{"/assets/picture/2022-04/springboot.png" | absolute_url}})

和Spring很相似，该项目下也有两个我新增的项目：`springboot-demo` 和`springboot-research`，前者模块中有个`DemoApplication`类，运行该类可测试SpringBoot的Spring环境； 后者模块中有个`AppApplication`类，运行会启动一个内嵌的Tomcat可供研究整个springboot；

<font color=red>Note:</font>gradle生成的路径和maven项目生成class文件路径不同，前者是build/class， 后者是target/class

## 4. 总结

虽然文章很简短，但是期间遇到了很多问题，也花了很多时间去解决了。 希望大家少走弯路这样就可以节省很所时间了。






