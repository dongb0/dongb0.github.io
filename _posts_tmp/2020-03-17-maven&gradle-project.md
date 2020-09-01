

有小半年没用 Java 了，最近实训需要写 Java 代码，碰到了一点问题，统一记录。

1. Eclipse No Java Virual Machine Found.

因为 Java 自动升级之后文件夹名称改变了，但是Eclipse的配置文件没有改，所以找不到 JVM 的路径。

把 Eclipse 安装目录下的 `` 文件的配置改为 安装 /// 的路径

2. Java 执行 .class 文件未找到主类

带上完整的包名，不要加.class后缀

普通 Java 项目文件目录结构：

Maven quickstart 项目目录结构：生成什么？

Gradle Java Application项目目录结构：跟 Maven 都生成了什么？

用了插件之后可以生成什么？


3. 如何妥善使用 Java Test，或者说一般项目里Test路径应该怎么用


maven和gradle创建Hello world项目的运行与普通Java project的对比

无控制台输出？

Idea

