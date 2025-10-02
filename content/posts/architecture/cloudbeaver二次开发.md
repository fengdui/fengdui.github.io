---
title: "cloudbeaver二次开发"
date: "2024-12-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---



# eclipse环境
https://github.com/dbeaver/dbeaver/wiki/Develop-in-Eclipse  
按照官网配置一路下来直接启动 出现问题换一下eclipse版本号 之前用了一个低版本号的各种问题

```
09:36:29.310 [main] DEBUG io.cloudbeaver.model.app.BaseWebApplication -- Loading configuration from C:\Users\Administrator\IdeaProjects\cloudbeaver\deploy\cloudbeaver\conf\cloudbeaver.conf  
09:36:29.312 [main] DEBUG io.cloudbeaver.server.CBServerConfigurationController -- Using configuration [C:\Users\Administrator\IdeaProjects\cloudbeaver\deploy\cloudbeaver\conf\cloudbeaver.conf]  
09:36:29.312 [main] DEBUG io.cloudbeaver.server.CBServerConfigurationController -- Read configuration [C:\Users\Administrator\IdeaProjects\cloudbeaver\deploy\cloudbeaver\conf\cloudbeaver.conf]  
09:36:29.424 [main] INFO io.cloudbeaver.server.CBPlatform -- Initialize web platform...:  
09:36:29.628 [main] DEBUG org.jkiss.dbeaver.runtime.SecurityProviderUtils -- BounceCastle bundle found. Use JCE provider BC  
09:36:30.444 [main] DEBUG org.jkiss.dbeaver.registry.BasePlatformImpl -- Initialize base platform...  
09:36:31.718 [main] DEBUG org.jkiss.dbeaver.registry.DataSourceProviderRegistry -- Total database drivers: 119 (119)  
09:36:31.816 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'yandex_clickhouse' is missing library 'ru.yandex.clickhouse:clickhouse-jdbc:RELEASE'  
09:36:31.819 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'com_clickhouse' is missing library 'drivers/clickhouse_com'  
09:36:31.819 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'db2' is missing library 'drivers/db2'  
09:36:31.821 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'db2_iseries' is missing library 'drivers/db2-jt400'  
09:36:31.822 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'jaybird' is missing library 'drivers/jaybird'  
09:36:31.822 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'duckdb_jdbc' is missing library 'drivers/duckdb'  
09:36:31.823 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'kyuubi_hive' is missing library 'drivers/kyuubi'  
09:36:31.823 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'trino_jdbc' is missing library 'drivers/trino'  
09:36:31.824 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'h2_embedded' is missing library 'drivers/h2'  
09:36:31.824 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'h2_embedded_v2' is missing library 'drivers/h2_v2'  
09:36:31.825 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'microsoft' is missing library 'drivers/mssql/new'  
09:36:31.826 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'mysql8' is missing library 'drivers/mysql/mysql8'  
09:36:31.826 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'mariaDB' is missing library 'drivers/mariadb'  
09:36:31.827 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'oracle_thin' is missing library 'drivers/oracle'  
09:36:31.827 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'postgres-jdbc' is missing library 'drivers/postgresql'  
09:36:31.828 [main] ERROR io.cloudbeaver.server.CBPlatform --     Driver 'sqlite_jdbc' is missing library 'drivers/sqlite/xerial'  
09:36:31.828 [main] INFO io.cloudbeaver.server.CBPlatform -- Available drivers: PostgreSQL  
09:36:31.913 [main] INFO io.cloudbeaver.server.CBPlatform -- Web platform initialized (2489ms)  
09:36:31.975 [main] INFO io.cloudbeaver.model.app.BaseServerConfigurationController -- Workspace path initialized: C:\Users\Administrator\IdeaProjects\cloudbeaver\config\core\workspace  
09:36:31.999 [main] DEBUG io.cloudbeaver.server.CBApplication -- CloudBeaver CE Server 24.2.4.qualifier is starting  
09:36:31.999 [main] DEBUG io.cloudbeaver.server.CBApplication --     OS: Windows 11 10.0 (amd64)  
09:36:31.999 [main] DEBUG io.cloudbeaver.server.CBApplication --     Java version: 21.0.4 by Eclipse Adoptium (64bit)  
09:36:32.000 [main] DEBUG io.cloudbeaver.server.CBApplication --     Install path: 'C:\Users\Administrator\Downloads\eclipse-jee-2024-09-R-win32-x86_64\eclipse'  
09:36:32.000 [main] DEBUG io.cloudbeaver.server.CBApplication --     Global workspace: 'file:/C:/Users/Administrator/runtime-CloudbeaverServer.product/'  
09:36:32.002 [main] DEBUG io.cloudbeaver.server.CBApplication --     Memory available 68Mb/4028Mb  
09:36:32.002 [main] DEBUG io.cloudbeaver.server.CBApplication --     Content root: C:\Users\Administrator\IdeaProjects\cloudbeaver\config\core\web  
09:36:32.002 [main] DEBUG io.cloudbeaver.server.CBApplication --     Drivers storage: C:\Users\Administrator\IdeaProjects\cloudbeaver\config\core\drivers  
09:36:32.004 [main] DEBUG io.cloudbeaver.server.CBApplication --     Listen port: 8978 on all interfaces  
09:36:32.004 [main] DEBUG io.cloudbeaver.server.CBApplication --     Base URI: /api/  
09:36:32.004 [main] DEBUG io.cloudbeaver.server.CBApplication --     Production mode  
09:36:32.004 [main] DEBUG io.cloudbeaver.server.CBApplication --     Server is in configuration mode!  
09:36:32.005 [main] DEBUG io.cloudbeaver.server.CBApplication --     Run in Docker container (host.docker.internal/192.168.99.60)?  
09:36:32.010 [main] DEBUG io.cloudbeaver.server.CBApplication --     Local host addresses:  
09:36:32.011 [main] DEBUG io.cloudbeaver.server.CBApplication --         192.168.99.60 (host.docker.internal)  
09:36:32.012 [main] DEBUG io.cloudbeaver.server.CBApplication --         0:0:0:0:0:0:0:1 (0:0:0:0:0:0:0:1)  
09:36:32.012 [main] DEBUG io.cloudbeaver.server.CBApplication --         192.168.96.94 (DESKTOP-FD)  
09:36:32.946 [main] DEBUG io.cloudbeaver.service.auth.ReverseProxyConfigurator -- Reverse proxy provider disabled, migration not needed  
09:36:33.306 [main] DEBUG io.cloudbeaver.service.security.db.CBDatabase -- Initiate management database  
09:36:33.395 [main] DEBUG io.cloudbeaver.service.security.db.CBDatabase -- Shutdown database  
09:36:33.395 [main] ERROR io.cloudbeaver.server.CBApplication -- Error initializing database  
org.jkiss.dbeaver.DBException: Error creating driver 'H2 Embedded V.2' instance.  
Most likely required jar files are missing.  
You should configure jars in driver settings.  
```

```
16:25:03.623 [main] DEBUG io.cloudbeaver.server.CBApplication --     Global workspace: 'file:/C:/Users/Administrator/runtime-CloudbeaverServer.product/'
16:25:03.627 [main] DEBUG io.cloudbeaver.server.CBApplication --     Memory available 74Mb/4028Mb
16:25:03.627 [main] DEBUG io.cloudbeaver.server.CBApplication --     Content root: C:\Users\Administrator\Downloads\eclipse-jee-2024-09-R-win32-x86_64\eclipse\web
16:25:03.627 [main] DEBUG io.cloudbeaver.server.CBApplication --     Drivers storage: C:\Users\Administrator\IdeaProjects\cloudbeaver\deploy\cloudbeaver\drivers
16:25:03.630 [main] DEBUG io.cloudbeaver.server.CBApplication --     Listen port: 8978 on all interfaces
```

Content root 这个是工作目录设置的路径 启动命令的参数 -web-config也是基于这个路径 要设置为C:\Users\Administrator\IdeaProjects\cloudbeaver\deploy\cloudbeaver

![img.png](/pic/img.png)

日志位置 org.jkiss.dbeaver.launcher.DBeaverLauncher#computeLogFileLocation

![img_1.png](/pic/img_1.png)

这里面是日志位置

![img_2.png](/pic/img_2.png)

C:\Users\Administrator\eclipse-workspace2.metadata.plugins\org.eclipse.pde.core\CloudbeaverServer.product
这里面是config.ini和dev.properties文件位置


# idea环境
!SESSION Mon Dec 23 09:53:04 CST 2024 ------------------------------------------ !ENTRY org.eclipse.equinox.launcher 4 0 2024-12-23 09:53:04.617 !MESSAGE Exception launching the Eclipse Platform: !STACK java.io.FileNotFoundException: Could not find framework under c:\Users\Administrator\IdeaProjects\cloudbeaver-all\dbeaver\target\classes\plugins at org.jkiss.dbeaver.launcher.DBeaverLauncher.getBootPath(DBeaverLauncher.java:962) at org.jkiss.dbeaver.launcher.DBeaverLauncher.basicRun(DBeaverLauncher.java:590) at org.jkiss.dbeaver.launcher.DBeaverLauncher.run(DBeaverLauncher.java:1461) at org.jkiss.dbeaver.launcher.DBeaverLauncher.main(DBeaverLauncher.java:1434)

配置文件需要正确 正确的配置  
-name  
CloudBeaverceServer  
-product  
io.cloudbeaver.product.ce.product  
-configuration  
"file:C:/Users/Administrator/IdeaProjects/cloudbeaver-all/dbeaver-workspace/products/CloudbeaverServer.product"  
-dev  
"file:C:/Users/Administrator/IdeaProjects/cloudbeaver-all/dbeaver-workspace/products/CloudbeaverServer.product/dev.properties"  
-nl   
en  
-consoleLog  
-showsplash  
-web-config  
conf/cloudbeaver.conf  
-registryMultiLanguage  
-vmargs  
-Xmx4096M  

启动找不到jkiss utils common模块是下划线找不到无法编译  
启动找不到CEApplicationCE类 需要先build dev配置文件是再target目录下面找class  
cd cloudbeaver sh generate_workspace.sh 这个是会生成dbeaver-workspace 里面是一些eclipse的第三方依赖 这个工具解析了maven依赖去下载依赖 C:\Users\Administrator\IdeaProjects\cloudbeaver-all\dbeaver-workspace\products\CloudbeaverServer.product里也有一些config.ini和dev.properties配置 日志也在这里  
idea打开dbeaver-workspace/cloudbeaver-ce 会自动生成启动配置  


# cloudbeaver
我的理解是eclipse插件开发使用了osgi规范。
Eclipse完全构建在OSGi之上，它包含有一个OSGi规范的完整实现Equinox。所以Eclipse的一个Plug-in，基本上就等于OSGi的一个Bundle

feature是功能部件，它里面没有实际运行的库，它只是eclipse用来管理plugins的一种途径, 控制哪些插件启用
plugins目录有插件没有被任何一个功能部件包络的话, 我称之为"野插件", 就是eclipse 启动,它也一定会启动，
feature是插件的集合，我们在给Eclipse安装插件的时候往往安装的是feature，而不是安装一个个的plugin jar。

p2-maven-plugin插件来生成p2类型的更新站点，该插件用来将定义好的feature和plugin生成p2站点。可以将jar包自动生成bundler
dbeaver\product\localRepository\pom.xml 使用了 p2-maven-plugin
https://github.com/reficio/p2-maven-plugin/

Tycho 通过定义了一套 Maven 生命周期插件（ tycho-maven-plugin ），它为 Eclipse 项目提供了一套完整的生命周期目标（goals），简化了Eclipse、OSGi插件中的pom.xml，它实际上是一系列专用于build Eclipse插件和OSGi模块的maven插件的集合。
https://tycho.eclipseprojects.io/doc/4.0.9/index.html

target-platform-configuration插件（用来build最后的product）的配置，告诉Tycho的这个maven插件我想要分别对windows，linux，macos三种操作系统build 专门的product，这样Tycho在build product的时候就会根据不同的os，选用不同的环境包并入最后的product中。

https://github.com/dbeaver/dbeaver-common
https://github.com/dbeaver/cloudbeaver
https://dbeaver.com/docs/cloudbeaver/Connection-Templates-Management/
https://blog.csdn.net/weixin_47314019/article/details/128576190
https://zhuanlan.zhihu.com/p/587648719

C:\Users\Administrator\IdeaProjects\cloudbeaver-all\cloudbeaver\server\product\web-server\CloudbeaverServer.product
这个配置是最后会生成C:\Users\Administrator\IdeaProjects\cloudbeaver-all\cloudbeaver\server\product\web-server\target\products\io.cloudbeaver.product\all\all\all\configuration\config.ini

-product io.cloudbeaver.product.ce.product 这个参数是C:\Users\Administrator\IdeaProjects\cloudbeaver-all\cloudbeaver\server\bundles\io.cloudbeaver.product.ce\plugin.xml 里org.eclipse.core.runtime.products" id="product234"扩展点配置的

org.jkiss.dbeaver.registry.DataSourceProviderRegistry.readDriversConfig()

io.cloudbeaver.WebSessionGlobalProjectImpl.isDataSourceAccessible(DBPDataSourceContainer)  改这个方法 不用授权能查到资产  
org.jkiss.dbeaver.registry.rm.DataSourceRegistryRM  资产管理 可以考虑改造 从dsc获取资产  
io.cloudbeaver.model.session.WebSession.loadProjects()  登录加载项目  
io.cloudbeaver.model.session.WebUserContext.isNonAnonymousUserAuthorizedInSM() 可以校验是否是匿名登录  
io.cloudbeaver.model.session.WebSessionWorkspace.getWorkspaceId() workspace返回的是userId  
io.cloudbeaver.auth.SMAuthProviderExternal<AUTH_SESSION> 可以考虑继承该类 去自己实现一个登录  
io.cloudbeaver.server.graphql.GraphQLEndpoint.executeQuery(HttpServletRequest, HttpServletResponse, String, Map<String, Object>, String) 解析operationName  
io.cloudbeaver.service.core.WebServiceBindingCore#bindWiring  

暂且这样 后面交给别人开发了
总结 eclipse就按照官方文档一路下来就能调试
idea麻烦点
https://github.com/dbeaver/cloudbeaver/wiki/Develop-in-IDEA