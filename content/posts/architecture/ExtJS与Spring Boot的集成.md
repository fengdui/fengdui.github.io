---
title: "ExtJS与Spring Boot的集成"
date: "2017-03-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

前端代码位于src/main/resources/static目录下，采用了ExtJS的MVC架构模式： 

demo/controller/ - 控制器  
demo/store/ - 数据存储  
demo/view/ - 视图组件  
ext4.0/ - ExtJS框架库  

app.js是通过index.html文件中的script标签引入的。在index.html 引入了app.js   
首先加载index.html作为入口文件  
然后按照index.html中script标签的顺序加载JavaScript文件：  
先加载ExtJS核心库：ext4.0/ext-all.js  
再加载中文语言包：ext4.0/ext-lang-zh_CN.js  
最后加载应用的入口文件：app.js  

app.js - 应用入口文件 是ExtJS应用程序的入口文件，主要由以下几个部分使用：  
1.浏览器加载和执行：当用户访问应用时，浏览器首先加载并执行app.js，它作为整个ExtJS应用的启动点。  
2。ExtJS框架：app.js通过调用Ext.application()方法向ExtJS框架注册应用信息，包括应用名称、控制器列表和初始化配置。  
3.应用组件：  
配置了应用名称为'demo'  
指定了应用文件夹路径为'demo'  
注册了UserController控制器  
在launch方法中创建了Viewport容器并加载userView组件  
4.应用初始化过程：当应用启动时，ExtJS框架会根据app.js中的配置，自动加载注册的控制器，并在DOM准备就绪后执行launch方法中的代码。  
```
Ext.Loader.setConfig({enabled: true});
Ext.application({
    name: 'demo',    
    appFolder: 'demo',
    controllers: [        
        'UserController'
    ],
    // ...
});
```
在UserStore.js中配置了数据代理，指定了后端API地址：  
```
Ext.define('demo.store.UserStore', {
    extend : 'Ext.data.Store',
    // ...
    proxy : {
        type : 'ajax',
        url : 'user/list',  // 后端API地址
        reader : {
            type : 'json',
            root : 'data',  // 数据根节点
            successProperty : 'success'  // 成功标识字段
        }
    }
});
```
在UserController.js中，实现了与后端API的交互逻辑：  
```
// 查询用户
searchBtnClicked : function(btn) {
    // ...
    store.getProxy().extraParams = queryMap;
    store.loadPage(1);  // 自动发送请求到后端
},

// 添加用户
addUser : function(btn) {
    // ...
    Ext.Ajax.request({
        url : 'user/add',  // 后端API地址
        method : 'POST',
        jsonData : values,  // 发送JSON数据
        // ...
    });
}
```

在UserService.java中，使用Spring Boot的注解定义了RESTful接口：  
```
@RestController
@RequestMapping("/user")
public class UserService {
    
    @RequestMapping("/list")
    public ExtJSResponse list(String userName){
        List<User> users = userBO.list(userName);
        return ExtJSResponse.successRes4Find(users, users.size());
    }
    
    @RequestMapping(value="/add", method={RequestMethod.POST})
    public ExtJSResponse add(@RequestBody User user){
        userBO.add(user);
        return ExtJSResponse.success();
    }
}
```
ExtJSResponse.java类，用于生成符合ExtJS数据存储要求的响应格式：  
```
public class ExtJSResponse extends HashMap<String, Object> {
    
    // 返回查询成功响应
    public static ExtJSResponse successRes4Find(final Object data, final Integer total) {
        final ExtJSResponse response = new ExtJSResponse(true);
        response.setData(data);
        response.put("total", total);  // 用于分页
        return response;
    }
    
    // 返回操作成功响应
    public static ExtJSResponse success() {
        return new ExtJSResponse(true);
    }
    
    // ...
}
```
Spring Boot会自动将src/main/resources/static目录下的静态资源提供给前端访问，无需额外配置。  