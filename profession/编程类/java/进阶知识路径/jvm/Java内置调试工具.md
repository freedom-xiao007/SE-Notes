# Java 调试工具
***
## 总览
&ensp;&ensp;&ensp;&ensp;简单做下记录，调试工具可以分为两大类：命令行和图形化的

### 命令行工具
&ensp;&ensp;&ensp;&ensp;带*号的为较为重要的，命令都可搜索或者查看详情

- java : Java 应用的启动程序
- javac  :JDK 内置的编译工具
- javap  :反编译 class 文件的工具
- javadoc  :根据 Java 代码和标准注释,自动生成相关的API说明文档
- javah  :JNI 开发时, 根据 java 代码生成需要的 .h文件。
- extcheck  :检查某个 jar 文件和运行时扩展 jar 有没有版本冲突，很少使用
- jdb : Java Debugger ; 可以调试本地和远端程序, 属于 JPDA 中的一个 demo 实现, 供其他调试器参考。开发时很少使用
- jdeps : 探测 class 或 jar 包需要的依赖
- jar :打包工具，可以将文件和目录打包成为 .jar 文件；.jar 文件本质上就是 zip 文件,只是后缀不同。使用时按顺序对应好选项和参数即可。
- keytool: 安全证书和密钥的管理工具; （支持生成、导入、导出等操作）
- jarsigner : JAR 文件签名和验证工具
- policytool: 实际上这是一款图形界面工具, 管理本机的 Java 安全策略
- *jps/jinfo: 查看 java 进程
- *jstat :查看 JVM 内部 gc 相关信息
- *jmap :查看 heap 或类占用空间统计
- *jstack :查看线程信息
- *jcmd :执行 JVM 相关分析命令（整合命令）
- *jrunscript/jjs :执行 js 命令

### 图形化工具
- jconsole：一般
- jvisualvm：还行
- VisualGC：还算好用
- jmc：比较好用