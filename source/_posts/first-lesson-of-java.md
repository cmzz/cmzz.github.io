---
title: 这回是个 Java 小白
date: 2022-04-29 00:40:00
tags: 
- Java
---

Java 是个有20多年历史的编程语言，语言的生态和应用场景极其完善。尤其是在企业环境中有着非常官方的应用。

此前一直没有使用过 Java，对 java 并不是很，而现在马上需要给客户交付 Java 语言的支付 SDK，临时来报佛脚，边看边学边写SDK，过程着顺便做一些记录。

## 名词
不过对于新手来说，首选面对的就是能排一长排的以 Java 为开头命名的名语，让人很是困惑：

- `Java EE` Enterprise Edition。这个版本以前称为 J2EE。企业版本帮助开发和部署可移植、健壮、可伸缩且安全的服务器端 Java 应用程序。Java EE 是在 Java SE 的基础上构建的，它提供 Web 服务、组件模型、管理和通信 API，可以用来实现企业级的面向服务体系结构（service-oriented architecture，SOA）和 Web 2.0 应用程序。
- `Java SE` Standard Edition。它允许开发和部署在桌面、服务器、嵌入式环境和实时环境中使用的 Java 应用程序。Java SE 包含了支持 Java Web 服务开发的类，并为 Java Platform，Enterprise Edition（Java EE）提供基础。
- `Java ME` Micro Edition。Java ME 为在移动设备和嵌入式设备（比如手机、PDA、电视机顶盒和打印机）上运行的应用程序提供一个健壮且灵活的环境。Java ME 包括灵活的用户界面、健壮的安全模型、许多内置的网络协议以及对可以动态下载的连网和离线应用程序的丰富支持。基于 Java ME 规范的应用程序只需编写一次，就可以用于许多设备，而且可以利用每个设备的本机功能。
- `jre` 是Java的运行环境。面向Java程序的使用者，而不是开发者 。如果你仅下载并安装了JRE，那么你的系统只能运行Java程序。JRE是运行Java程序所必须环境的集合，包含JVM标准实现及Java核心类库。它包括Java虚拟机、Java平台核心类和支持文件。 它不包含开发工具(编译器、调试器等)。
- `jdk` JDK(Java Development Kit)又称J2SDK(Java2 Software Development Kit)，是Java开发工具包， 它提供了Java的开发环境(提供了编译器javac等工具，用于将java文件编译为class文件)和运行环境(提供了JVM和Runtime辅助包，用于解析class文件使其得到运行)。如果你下载并安装了JDK，那么你不仅可以开发Java程序，也同时拥有了运行Java程序的平台。 JDK是整个Java的核心，包括了Java运行环境(JRE)，一堆Java工具tools.jar和Java标准类库 (rt.jar)。


![jre与jdk对比图](https://tva1.sinaimg.cn/large/e6c9d24egy1h1r10a12o9j20fk0b2t9r.jpg)


除了上面的，还有一堆的令人困惑版本号：Java 1.x Java 1x。

> 首先1996年发布了最初版本Java1.0，此后为Java1.1、J2SE1.2、J2SE1.3、J2SE1.4、采用 1.X的命名方式，直到 2004 年的 JavaOne 会议后版本数提升为 5.0，这一新版本为Java SE5.0，在 2006 年 Sun 公司终结了已经有 8 年历史的 J2SE、J2EE、J2ME 的命名方式启用了今天的 Java SE、Java EE、Java ME  命名方式，而此后的版本为 Java SE6、Java SE7、Java SE8、Java SE9、Java SE10、Java SE11、Java SE12、JAVA SE18

> 而JDK则在 Java1.0 到 Java9 对应每一个版本号 ：JDK1.0、JDK1.2 ... JDK1.8、JDK1.9，Java10 以后JDK对应名称为：JDk10、JDK11、JDK12、JDK18

所以
- `Java 8` 指 Java SE 8.0 的版本
- `Java 18` 指 Java SE 18 的版本 
- `JDK 1.8` 则指 Java 8 对应的 JDK 版本

## 版本
Java现在的最新版本是 Java 18，但在市场上最受欢迎的以及最为普遍的不是  `Java 8`，这又带来了环境与版本的问题。

但好在安装 Java 之后提供一个工具来管理当前系统上的java版本。命令是 `/usr/libexec/java_home` 。

在终端运行 `/usr/libexec/java_home -V` 可以看到系统中安装的所有版本的 Java  程序，并可以切换到对应的版本： `/usr/libexec/java_home -v <version>`

如：`/usr/libexec/java_home -v 1.8`，可以切换到 JDK 1.8。之后可通过 `java -version` 来查看当前的版本。

## 环境变量
在 java 中有几个重要的环境变量需要在安装完 Java 之后进行配置：
- `JAVA_HOME` 通常它指的是JDK的目录。如果需要JDK的话，大部分程序会默认去环境变量中取JAVA\_HOME这个变量。
- `JRE_HOME` 同样，这也是一个约定的变量，通常指JRE目录。其实大部分Java程序不需要JDK，而是只需要其子集JRE，所以很多程序也会去取这个变量来用。

在程序中，也可以通过 Java 提供的 API 来获取：


    find String env = System.getenv("PATH");
    System.out.println(env);


因此，我们可以将设置版本以及选设置环境变量放在 `~/.bash_profile` 或 `~/.zshrc` 文件中：


    export JAVA_HOME=/usr/libexec/java_home -v 1.8.0_331
    export PATH=${JAVA_HOME}/bin:$PATH


##后缀
在 Java 程序中，有几种不同类型的文件后缀名，对应到程序的源码、编译等不同的阶段。

- `.java` 是 Java 程序的源代码
- `.class` 是 .java 源代码编译的的字节码文件，真正可以被 jvm 执行
- `.jar` 将一组 .class 文件打包而来，本质是一个 zip 格式的压缩文件

##Maven
Apache Maven，是一个软件（特别是Java软件）项目管理及自动构建工具，由Apache软件基金会所提供。

Maven项目使用项目对象模型（Project Object Model，POM）来配置。

项目对象模型存储在名为 pom.xml 的文件中。
