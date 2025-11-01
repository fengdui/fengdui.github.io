---
title: "借助kettle+类加载器实现多版本数据库驱动管理"
date: "2025-05-07"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

流程如下  
com.aaa.datasource.cmd.DatasourceDataCmdMain  
com.aaa.datasource.cmd.DatasourceMetaCmdMain#getDbMetaDataByJdbc  
DbTableMetaData dbTableMetaData = DbObjectEngine.getInstance().getDbMetaDataByJdbc(...)  
com.aaa.datasource.core.dbobject.JdbcDbObjectImpl#getColumnsByCatalog(DatabaseMeta, String, String, int, String)  
然后就到了kettle的database  
org.pentaho.di.core.database.Database#connectUsingClass 这里面会Class.forName(classname);DriverManager.getConnection(url, properties);  
这里走的驱动是自己写的一个通用的驱动类 baseDiverproxy 通用驱动类里面替换了jdbcurl为真实的 使用类加载器 在去加载驱动 然后返回连接  
每种数据库的插件都定义在一个目录下 根据插件的id去对应路径下加载驱动 connection.getMetaData()得到元数据  
所以我理解是没有用到了kettle的插件 而是自己实现了一套插件机制  
插件里面还有各种sql 支持原生sql去获取数据库信息 而非connection.getMetaData()  
```
import java.io.IOException;
import java.net.URL;
import java.net.URLClassLoader;
import java.util.Enumeration;
import java.util.Iterator;
import java.util.LinkedList;
import java.util.List;

public class ChildFirstClassLoader extends URLClassLoader {
    private final ClassLoader sysClzLoader = null;

    public ChildFirstClassLoader(URL[] urls, ClassLoader parent) {
        super(urls, parent);
    }

    public ChildFirstClassLoader(URL[] urls) {
        super(urls, Thread.currentThread().getContextClassLoader());
    }

    protected Class<?> loadClass(String name, boolean resolve) throws ClassNotFoundException {
        Class<?> loadedClass = this.findLoadedClass(name);
        if (loadedClass == null) {
            try {
                if (this.sysClzLoader != null) {
                    loadedClass = this.sysClzLoader.loadClass(name);
                }
            } catch (ClassNotFoundException var6) {
            }

            try {
                if (loadedClass == null) {
                    loadedClass = this.findClass(name);
                }
            } catch (ClassNotFoundException var5) {
                loadedClass = super.loadClass(name, resolve);
            }
        }

        if (resolve) {
            this.resolveClass(loadedClass);
        }

        return loadedClass;
    }

    public Enumeration<URL> getResources(String name) throws IOException {
        final List<URL> allRes = new LinkedList();
        if (this.sysClzLoader != null) {
            Enumeration<URL> sysResources = this.sysClzLoader.getResources(name);
            if (sysResources != null) {
                while(sysResources.hasMoreElements()) {
                    allRes.add(sysResources.nextElement());
                }
            }
        }

        Enumeration<URL> thisRes = this.findResources(name);
        if (thisRes != null) {
            while(thisRes.hasMoreElements()) {
                allRes.add(thisRes.nextElement());
            }
        }

        Enumeration<URL> parentRes = super.findResources(name);
        if (parentRes != null) {
            while(parentRes.hasMoreElements()) {
                allRes.add(parentRes.nextElement());
            }
        }

        return new Enumeration<URL>() {
            Iterator<URL> it = allRes.iterator();

            public boolean hasMoreElements() {
                return this.it.hasNext();
            }

            public URL nextElement() {
                return (URL)this.it.next();
            }
        };
    }

    public URL getResource(String name) {
        URL res = null;
        if (this.sysClzLoader != null) {
            res = this.sysClzLoader.getResource(name);
        }

        if (res == null) {
            res = this.findResource(name);
        }

        if (res == null) {
            res = super.getResource(name);
        }

        return res;
    }
}
```