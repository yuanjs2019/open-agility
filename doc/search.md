#### 一、Apache Common官方文档

1. Apache Common [BCEL](http://commons.apache.org/bcel/):  BCEL库提供了一系列用于分析、创建、修改Java Class文件的[API](https://so.csdn.net/so/search?q=API&spm=1001.2101.3001.7020)。主要用来将xml文档转为class文件。编译后的class被称为translet，可以在后续用于对XML文件的转换。该库已经内置在了JDK中。 使用参考: [红队漏洞研究-fastjsonBasicDataSource链分析](https://blog.csdn.net/gamma_lab/article/details/123294137)
2. ★ Apache Common [BeanUtils](http://commons.apache.org/beanutils/):  用于操作JAVA bean的工具包。里面提供了各种各样的工具类，让我们可以很方便的`对bean对象的属性进行各种操作`。主要是熟悉commons-beanutils库里面`MethodUtils、ConstructorUtils、PropertyUtils、BeanUtils、ConvertUtils`的使用。 参考: [commons-beanutils的使用](https://www.jianshu.com/p/27c0ea663d83)
3. Apache Common [BSF](http://commons.apache.org/bsf/): 一个支持在Java应用程序内调用脚步语言(Script)。参考：[Apache项目Research之BSF](https://blog.csdn.net/maqujun/article/details/83195119)
4. ★ Apache Common [Chain](http://commons.apache.org/chain/): 主要被使用在"责任链"的场景中。参考：[使用Apache Commons Chain](https://www.iteye.com/blog/phil-xzh-321536)
5. Apache Common [CLI](http://commons.apache.org/cli/) : 它们派生自Java API，并提供API来解析传递给程序的命令行参数/选项。 此API还可以打印与可用选项相关的帮助。参考：[Commons CLI - 快速指南](https://iowiki.com/commons_cli/commons_cli_quick_guide.html)
6. ★ Apache Common [Codec](http://commons.apache.org/codec/)：加密与编码。参考: [Apache Commons Codec ](https://zhuanlan.zhihu.com/p/388504402) /  [Apache Commons](https://www.zhihu.com/column/c_1395490348211298304)
7. ★★ **Apache Common [Collections4](http://commons.apache.org/collections/)**: 对 java.util.Collection 的扩展。参考: [apache commons类库的详解](https://blog.csdn.net/leaderway/article/details/52387925)
8. Apache Common [Compress](http://commons.apache.org/compress/): 文件解压缩库。用于处理 ar，cpio，Unix 转储，tar，zip，gzip，XZ，Pack200，bzip2、7z，arj，lzma，snappy，DEFLATE，lz4，Brotli，Zstandard，DEFLATE64 和 Z 文件的 API 。[Apache Commons Compress ](https://zhuanlan.zhihu.com/p/389762356)
9. Apache Common  [Configuration2](http://commons.apache.org/configuration/)：管理配置文件的读写。[简单的使用Configuration读取和修改配置文件](https://blog.csdn.net/weixin_45492007/article/details/118070473) 、[ 二、Apache Commons Configuration](https://blog.csdn.net/f641385712/article/details/104350308)
10. Apache Common  [CSV](http://commons.apache.org/csv/)： 实现创建和读取CSV文件。[Apache Commons CSV 教程](https://blog.csdn.net/neweastsun/article/details/85143572)
11. Apache Common  [Daemon](http://commons.apache.org/daemon/)：将一个普通的java应用程序作为linux或windows的后台服务，以daemon方式运行。[Apache Commons Daemon 使用简介](https://blog.csdn.net/catherine160/article/details/21444119)
12. Apache Common  [DBCP](http://commons.apache.org/dbcp/)：提供数据库连接池服务。[Apache commons 与开源的数据源连接池DBCP](https://blog.csdn.net/qq_44861675/article/details/107871172)
13. Apache Common  [DbUtils](http://commons.apache.org/dbutils/)：是一组非常小的类，旨在使JDBC调用处理更容易而不会泄漏资源，并具有更干净的代码。[Apache Commons DBUtils指南](https://blog.csdn.net/allway2/article/details/123974410)
14. Apache Common  [Digester](http://commons.apache.org/digester/)：是一个 XML-Java对象的映射工具，用于解析 XML配置文件.   [Apache Commons Digester 一 （基础内容、核心API](https://www.cnblogs.com/chenpi/p/6930730.html)
15. Apache Common  [Discovery](http://commons.apache.org/discovery/)：提供工具来定位资源 (包括类) ，通过使用各种模式来映射服务/引用名称和资源名称。 [使用Apache Commons Discovery查找可插拔接口实现类(Pluggable interfaces)](https://www.iteye.com/blog/terrencexu-715982)
16. Apache Common  [EL](http://commons.apache.org/el/)：提供在JSP2.0规范中定义的EL表达式的解释器
17. Apache Common  [Email](http://commons.apache.org/email/)：发送邮件。[使用Apache commons email发送邮件](https://www.cnblogs.com/conti/p/13145164.html)
18. Apache Common  [Exec](http://commons.apache.org/exec/)：提供一些常用的方法用来执行外部进程。[Apache commons-exec的使用](https://blog.csdn.net/u011943534/article/details/120938888)
19. Apache Common  [FileUpload](http://commons.apache.org/fileupload/)：
20. Apache Common  [Functor](http://commons.apache.org/functor/)
21. Apache Common  [Imaging (previously called Sanselan)](http://commons.apache.org/imaging/)
22. ★★Apache Common  [IO](http://commons.apache.org/io/)
23. Apache Common  [JCI](http://commons.apache.org/jci/)
24. Apache Common  [JCS](http://commons.apache.org/jcs/)
25. Apache Common  [Jelly](http://commons.apache.org/jelly/)
26. Apache Common  [Jexl](http://commons.apache.org/jexl/)
27. Apache Common  [JXPath](http://commons.apache.org/jxpath/)
28. ★★ Apache Common  [Lang](http://commons.apache.org/lang/)
29. Apache Common  [Launcher](http://commons.apache.org/launcher/)
30. Apache Common  [Logging](http://commons.apache.org/logging/)
31. ★★ Apache Common  [Math](http://commons.apache.org/math/)
32. Apache Common  [Modeler](http://commons.apache.org/modeler/)
33. Apache Common  [Net](http://commons.apache.org/net/)
34. Apache Common  [OGNL](http://commons.apache.org/ognl/)
35. ★★ Apache Common  [Pool](http://commons.apache.org/pool/)
36. Apache Common  [Primitives](http://commons.apache.org/primitives/)
37. ★★ Apache Common  [Proxy](http://commons.apache.org/proxy/)
38. Apache Common  [SCXML](http://commons.apache.org/scxml/)
39. Apache Common  [Validator](http://commons.apache.org/validator/)
40. Apache Common  [VFS](http://commons.apache.org/vfs/)
41. Apache Common  [Weaver](http://commons.apache.org/weaver/)

#### 二、Guava文档

[Google Guava官方教程（中文版）](https://wizardforcel.gitbooks.io/guava-tutorial/content/1.html)

[Gson - 教程](https://iowiki.com/gson/gson_index.html)

#### 三、vertx

多语言事件驱动应用框架 。  [Documentation | Eclipse Vert.x (vertx.io)](https://vertx.io/docs/)

#### 四、并发编程网

[并发编程网](http://ifeve.com/)

#### 五、javatuples

提供tuple支持。[javatuples - Main](https://www.javatuples.org/)

#### 六、函数式库 Vavr

[Vavr User Guide](https://docs.vavr.io/)

#### 七、Reactor 3

[Reactor 3 Reference Guide ](https://projectreactor.io/docs/core/release/reference/)

[Reactor 3 参考文档 ](http://htmlpreview.github.io/?https://github.com/get-set/reactor-core/blob/master-zh/src/docs/index.html)

#### 八、Genson

[Genson ](http://genson.io/#)