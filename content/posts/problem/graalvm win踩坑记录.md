---
title: "graalvm win踩坑记录"
date: "2025-11-21"
tags: ["问题"]
ShowToc: false
TocOpen: false
---
win环境使用native插件 非springboot项目  
使用graalvm-ce-java8-windows-amd64-20.3.2 这个版本  
```
<plugin>
  <groupId>org.graalvm.buildtools</groupId>
  <artifactId>native-maven-plugin</artifactId>
  <version>0.9.18</version>
  <configuration>
    <mainClass>com.utopia.factory.web.Application</mainClass>
    <imageName>brobot-factory</imageName>
    <useArgFile>true</useArgFile>  <!-- 关键：启用参数文件 -->
    <buildArgs>
      <buildArg>--no-fallback</buildArg>
      <buildArg>-H:+ReportExceptionStackTraces</buildArg>
      <buildArg>--verbose</buildArg>
      <!-- 如果有反射配置文件且存在，可以添加 -->
      <!-- <buildArg>-H:ReflectionConfigurationFiles=src/main/resources/META-INF/native-image/reflection-config.json</buildArg> -->
    </buildArgs>
    <classpath>
      <param>target/classes</param>
      <param>target/*</param>
    </classpath>
  </configuration>
  <executions>
    <execution>
      <id>build-native</id>
      <goals>
        <goal>build</goal>
      </goals>
      <phase>package</phase>
    </execution>
  </executions>
</plugin>
```
一直报错Main entry point class '@target\tmp\native-image-3021878386242950399.args  
```
[INFO] Executing: C:\Users\admin\Documents\graalvm-ce-java8-windows-amd64-20.3.2\graalvm-ce-java8-20.3.2\bin\native-image.cmd @target\tmp\native-image-3021878386242950399.args
[@target\tmp\native-image-3021878386242950399.args:36792]    classlist:     897.10 ms,  1.05 GB
Error: Main entry point class '@target\tmp\native-image-3021878386242950399.args' not found.
Error: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
Error: Image build request failed with exit status 1
```
改成这个<useArgFile>false</useArgFile> 不生成文件  
又出现了 命令行过长问题 因为 cp后面一堆jar包 导致命令行太长了  
改用maven-antrun-plugin 插件先生成-cp的内容到文件 在buildArg里面@指向他 但是发现只是追加了应该-classpath参数 -cp还是存在 并没有变短 网上很多说的去掉-cp方法无效  
改用exec-maven-plugin插件 相当于手动执行native-image命令 最终如下  
```
<!-- 1. 生成类路径文件（target/classpath.txt） -->
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-antrun-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <phase>prepare-package</phase>
            <goals>
                <goal>run</goal>
            </goals>
            <configuration>
                <target>
                    <!-- 生成类路径 -->
                    <pathconvert property="classpath" pathsep=";">
                        <path refid="maven.runtime.classpath" />
                    </pathconvert>

                    <!-- 创建完整的native-image参数文件 -->
                    <echo file="${project.build.directory}/native-image-args.txt">-cp ${classpath}${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">-H:Name=relayserver${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">-H:Class=com.vaasplus.relayserver.RelayServer${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">--no-fallback${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">--report-unsupported-elements-at-runtime${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">-H:+ReportExceptionStackTraces${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">--verbose${line.separator}</echo>
                    <echo file="${project.build.directory}/native-image-args.txt" append="true">--reflection-config=${project.basedir}/src/main/resources/META-INF/native-image/reflection-config.json${line.separator}</echo>
                </target>
            </configuration>
        </execution>
    </executions>
</plugin>
<plugin>
    <groupId>org.codehaus.mojo</groupId>
    <artifactId>exec-maven-plugin</artifactId>
    <version>3.1.0</version>
    <executions>
        <execution>
            <id>build-native</id>
            <phase>package</phase>
            <goals>
                <goal>exec</goal>
            </goals>
            <configuration>
                <executable>cmd</executable>
                <workingDirectory>${project.basedir}</workingDirectory>
                <arguments>
                    <argument>/c</argument>
                    <argument>native-image</argument>
                    <argument>-cp</argument>
                    <argument>@target\classpath.txt</argument>
                    <argument>-H:Name=relayserver</argument>
                    <argument>-H:Class=com.vaasplus.relayserver.RelayServer</argument>
                    <argument>--no-fallback</argument>
                    <argument>--report-unsupported-elements-at-runtime</argument>
                    <argument>-H:+ReportExceptionStackTraces</argument>
                    <argument>--verbose</argument>
                    <argument>-H:ReflectionConfigurationFiles=${project.basedir}/src/main/resources/META-INF/native-image/reflect-config.json</argument>
                    <argument>com.vaasplus.relayserver.RelayServer</argument>
                </arguments>
            </configuration>
        </execution>
    </executions>
</plugin>
```
上面相当于直接使用native-image命令 先生成-cp的内容到文件 自己控制-cp后面引入文件  
发现还是无法解析生成的classpath文件 就是-cp的内容 @target\classpath.txt' \  
-imagecp  
'C:\Users\admin\Documents\graalvm-ce-java8-windows-amd64-20.3.2\graalvm-ce-java8-20.3.2\jre\lib\boot\graal-sdk.jar;C:\Users\admin\Documents\graalvm-ce-java8-windows-amd64-20.3.2\graalvm-ce-java8-20.3.2\jre\lib\boot\graaljs-scriptengine.jar;C:\Users\admin\Documents\graalvm-ce-java8  
-windows-amd64-20.3.2\graalvm-ce-java8-20.3.2\jre\lib\svm\library-support.jar;E:\IdeaProjects\server@target\classpath.txt' \  

于是使用直接使用native-image -jar命令 试一下  
```
Error: Error parsing JNI configuration in jar:file:/E:/IdeaProjects/RelayV5/relayserver/bin/netty-transport-classes-epoll-4.1.100.Final.jar!/META-INF/native-image/io.netty/netty-transport-classes-epoll/jni-config.json:
Unknown attribute 'condition' (supported attributes: allDeclaredConstructors, allPublicConstructors, allDeclaredMethods, allPublicMethods, allDeclaredFields, allPublicFields, methods, fields) in defintion of class io.netty.channel.ChannelException
Verify that the configuration matches the schema described in the -H:PrintFlags=+ output for option JNIConfigurationResources.
Error: Use -H:+ReportExceptionStackTraces to print stacktrace of underlying exception
Error: Image build request failed with exit status 1
```
我现在用的是graalvm-ce-java8-windows-amd64-20.3.2 版本 不支持condition属性  
找到最新的支持java8的版本https://github.com/graalvm/graalvm-ce-builds/releases/download/vm-21.3.1/graalvm-ce-java8-windows-amd64-21.3.1.zip  
# 升级版本(graalvm-ce-java8-windows-amd64-21.3.1)之后还是不支持condition属性

# 换了个 spring boot项目 不用插件出现 Error: Default native-compiler executable 'cl.exe' not found via environment variable PATH

# win下升级了一个graalvm版本 springboot和非springboot 使用插件useArgFile>true</useArgFile>支持了 但是出现了'cl.exe' not found的问题
升级版本之后 win下4种情况都殊途同归了  
1.有插件springboot 2.有插件非springboot 3没插件springboot 出现cl.exe' not found  
没插件非springboot 出现不支持condition属性的问题  
建议不要在win上搞 起码低版本不要  

# 总结
win使用native插件非springboot 命令行过长  不过换了一个版本的graalvm 插件改成改成这个<useArgFile>true</useArgFile> 能够读取内容了 但是会报错cl.exe 不存在  
win使用native插件springboot 命令行过长 换了一个版本的graalvm报错cl.exe 不存在  
win不使用native插件非springboot condition不存在需要适配java8  换了一个版本的graalvm还是有这个问题 奇怪的是用插件报错是cl.exe  
win不使用native插件springboot cl.exe 不存在  