---
title: "中转中心和dubbo泛化调用"
date: "2018-07-05"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

公司有个项目是收购过来的 部署在某个云上 现在财务对账等等都要接入公司自己的业务 公司的业务部署在另外一个云上面 直接走dubbo是不通的
所以想了一个办法 就是在收购过来的项目里面部署一个中转中心 中转中心和公司的业务都部署在同一个云上面 中转中心暴露dubbo接口到api网关 
在收购的项目里面改造 通过泛化调用用http的方式打到网关 网关泛化调用公司的业务dubbo接口

```
POST /api/oauthentry/user/1.0/createUser
Content-Type: application/json

{
    "user_name": "张三",
    "phone_number": "13800138000", 
    "user_info": {
        "real_name": "张三丰",
        "id_card": "110101199001011234",
        "address_info": {
            "province_name": "北京市",
            "city_name": "朝阳区"
        }
    },
    "order_list": [
        {
            "order_id": "12345",
            "order_status": "paid",
            "create_time": "2023-01-01"
        }
    ]
}
```
```
// 参数映射配置 (CarmenParamMapping)
user_name     → userName     (String)
phone_number  → phoneNumber  (String) 
user_info     → userInfo     (UserInfo对象)
order_list    → orderList    (List<Order>)

// Dubbo服务方法定义
interface UserService {
    Result createUser(String userName, String phoneNumber, UserInfo userInfo, List<Order> orderList);
}
```


list map json 对象 递归 等等参数转换 转成驼峰命名
```
// 构建调用参数
String[] typeArrays = {
    "java.lang.String",      // userName
    "java.lang.String",      // phoneNumber  
    "com.youzan.UserInfo",   // userInfo
    "java.util.List"         // orderList
};

Object[] objectArrays = {
    "张三",                   // userName
    "13800138000",           // phoneNumber
    {                        // userInfo (Map形式)
        "realName": "张三丰",
        "idCard": "110101199001011234",
        "addressInfo": {
            "provinceName": "北京市", 
            "cityName": "朝阳区"
        }
    },
    [                        // orderList (List形式)
        {
            "orderId": "12345",
            "orderStatus": "paid",
            "createTime": "2023-01-01"
        }
    ]
};

// 执行泛化调用
ReferenceConfig<GenericService> reference = new ReferenceConfig<>();
// 设置接口名
reference.setInterface("com.example.UserService");
// 设置为泛化接口
reference.setGeneric("true");
GenericService genericService = reference.get();
ref.$invoke("createUser", typeArrays, objectArrays);
```