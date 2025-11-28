---
title: "tomcat-embed+maven assembly启动java工程"
date: "2015-09-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-assembly-plugin</artifactId>
    <configuration>
        <descriptors>
            <descriptor>src/main/assemble/assembly.xml</descriptor>
        </descriptors>
        <finalName>${project.artifactId}-${project.version}</finalName>
    </configuration>
    <executions>
        <execution>
            <id>make-assembly</id>
            <phase>install</phase>
            <goals>
                <goal>single</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```
assembly.xml
```
<?xml version="1.0" encoding="UTF-8" ?>
<assembly
	xmlns="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/plugins/maven-assembly-plugin/assembly/1.1.3 http://maven.apache.org/xsd/assembly-1.1.3.xsd">
	<id>SNAPSHOT</id>
	<formats>
		<format>zip</format>
	</formats>
	<dependencySets>
		<!-- 项目的依赖 -->
		<dependencySet>
			<!-- 排除当前项目的构件 -->
			<useProjectArtifact>true</useProjectArtifact>
			<outputDirectory>lib</outputDirectory>
		</dependencySet>
	</dependencySets>
	<fileSets>
		<fileSet>
			<directory>bin/</directory>
			<outputDirectory>/bin</outputDirectory>
			<includes>
				<include>*.sh</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>${project.build.directory}</directory>
			<outputDirectory>/bin</outputDirectory>
			<includes>
				<include>${project.artifactId}-${project.version}.jar</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>src/main/resources</directory>
			<outputDirectory>/conf</outputDirectory>
			<includes>
				<include>*.properties</include>
				<include>logback.xml</include>
			</includes>
		</fileSet>
		
		<!-- 提倡建议所有工程输出java API docs和sources jar -->
		<fileSet>
			<directory>${project.build.directory}</directory>
			<outputDirectory>/source</outputDirectory>
			<includes>
				<include>${project.artifactId}-${project.version}-sources.jar</include>
			</includes>
		</fileSet>
		<fileSet>
			<directory>${project.build.directory}</directory>
			<outputDirectory>/doc</outputDirectory>
			<includes>
				<include>${project.artifactId}-${project.version}-javadoc.jar</include>
			</includes>
		</fileSet>
	</fileSets>
</assembly>
```
```
<dependencySet>
    <useProjectArtifact>true</useProjectArtifact>
    <outputDirectory>lib</outputDirectory>
</dependencySet>
执行 mvn install 后会生成如下目录结构：
```
```
项目名-版本-SNAPSHOT.zip
├── bin/           # 启动脚本和主jar包
├── lib/           # 所有第三方依赖jar包
├── conf/          # 配置文件
├── webapp/        # Web资源文件
├── source/        # 源码jar包
└── doc/           # 文档jar包
```
maven-jar-plugin 打包 指定classpathPrefix
```
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-jar-plugin</artifactId>
    <version>2.3.2</version>
    <configuration>
        <excludes>
            <exclude>*.properties</exclude>
        </excludes>
        <archive>
            <manifest>
                <addDefaultImplementationEntries>true</addDefaultImplementationEntries>
                <addDefaultSpecificationEntries>true</addDefaultSpecificationEntries>
                <addClasspath>true</addClasspath>
                <classpathPrefix>../lib/</classpathPrefix>
                <mainClass>org.jsbd.boss.Booter</mainClass>
            </manifest>
            <manifestEntries>
                <Class-Path>../conf/</Class-Path>
            </manifestEntries>
        </archive>
    </configuration>
</plugin>
```
运行方式
cd 项目名-版本-SNAPSHOT/
java -jar bin/项目名-版本.jar

我的main方法在org.jsbd.boss.Booter类中 就一句话TomcatEmbed.main(args);
```
<dependency>
    <groupId>com.toolkit</groupId>
    <artifactId>toolkit-tomcat-embed</artifactId>
    <version>0.0.8</version>
    <scope>compile</scope>
</dependency>
```

