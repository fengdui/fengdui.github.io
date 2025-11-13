---
title: "使用webflux+jackson完成api接口聚合平台"
date: "2020-04-29"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

当前端需要调用多个后端接口做聚合时，后端又不想额外开发一个聚合接口 于是公司开发了一个小工具  
可以通过配置的方式选择几个接口 然后生成一个额外的聚合接口 前端只需要调用这个接口即可  

具体的配置如下
```
{
  "apis": [
    {
      "id": "200020027",
      "method": "get",
      "data": {
        "pageNo": "1",
        "pageSize": "$request.pageSize"
      },
      "name": "getMyPurchase"
    },
    {
      "id": "200920043",
      "method": "get",
      "data": {
        "ids": "#getMyPurchase.$response.results.$items.id"
      },
      "name": "getColumnInfo"
    }
  ],
  "response": [
    {
      "path": "$response",
      "allOf": "#getMyPurchase.$response"
    },
    {
      "path": "$response.results.items.columnInfo",
      "matchOf": {
        "ref": "#getColumnInfo.$response.results.$items",
        "condition": "$response.results.$items.id == #getColumnInfo.$response.results.$items.id",
        "properties": [
          "id",
          "title"
        ]
      }
    }
  ]
}
```
// 配置写的是：{"ids": "#getMyPurchase.$response.results.$items.id"}  
// 系统做的事：  
String template = "#getMyPurchase.$response.results.$items.id";  
// 1. 找到 getMyPurchase 接口的返回结果  
// 2. 按路径 $response.results.$items.id 提取数据  
// 3. 把提取的值替换到参数中  第二个接口使用的是第一个接口返回的参数  
里面有很多层级关系 使用了jackson的路径表达式 可以很方便的提取数据  
// 使用 Spring WebFlux调 getMyPurchase 接口  
```
Supplier<String> result = webClient
.method(HttpMethod.GET)
.uri("https://api.net/api/200020027")
.retrieve()
.bodyToMono(String.class)
.toFuture()::get;  // 转换为Supplier
```

调用接口 200020027，参数：pageSize=10  
得到结果：{"results": {"items": [{"id": "123"}, {"id": "456"}]}}  

// 2. 用第一个接口的结果调第二个接口  
从结果中提取 id: "123,456"  
调用接口 200920043，参数：ids="123,456"  
得到结果：{"results": [{"id": "123", "title": "专栏1"}, {"id": "456", "title": "专栏2"}]}  

然后拼接
// 按照 response 配置拼装数据  
原始数据：  
getMyPurchase: {"results": {"items": [{"id": "123"}, {"id": "456"}]}}  
getColumnInfo: {"results": [{"id": "123", "title": "专栏1"}, {"id": "456", "title": "专栏2"}]}  

//path指定位置，matchOf提供数据  
// 配置写的是：path: "$response.results.items.columnInfo" 找到 $response.results.items 数组 给数组中的每个对象添加 columnInfo 字段 
// 系统做的事：  
// 1. 在最终结果的 $response.results.items 位置  
// 2. 给每个 item 加上 columnInfo 字段  
// 3. columnInfo 的值从 ref字段获取 condition是匹配的条件
// 按 id 匹配，把专栏信息塞到对应位置 

"condition": "$response.results.$items.id == #getColumnInfo.$response.results.$items.id"  
把最终结果中每个item的id，和getColumnInfo接口返回的每个item的id进行匹配 匹配上的才添加进去 其实就是一个简单的join操作  
具体数据示例  
左边：$response.results.$items.id
```
// 最终要返回的结果中的数据（来自getMyPurchase接口）
{
  "results": {
    "items": [
      {"id": "123", "learnCount": 5},
      {"id": "456", "learnCount": 10}
    ]
  }
}
```
右边：#getColumnInfo.$response.results.$items.id
```
// getColumnInfo接口返回的数据
{
  "results": [
    {"id": "123", "title": "专栏A", "price": 99},
    {"id": "456", "title": "专栏B", "price": 199},
    {"id": "789", "title": "专栏C", "price": 299}
  ]
}
```
最终结果
```
// 聚合后的结果
{
  "results": {
    "items": [
      {
        "id": "123", 
        "learnCount": 5,
        "columnInfo": {"id": "123", "title": "专栏A"}  // 通过id=123匹配上的
      },
      {
        "id": "456", 
        "learnCount": 10,
        "columnInfo": {"id": "456", "title": "专栏B"}  // 通过id=456匹配上的
      }
    ]
  }
}
```
```
// 输入：多个接口返回 + 聚合规则
JsonNode baseResponse = getMyPurchaseResult;  // 基础数据
JsonNode columnData = getColumnInfoResult;    // 要聚合的数据

// 技术：Jackson ObjectNode动态构建
ObjectNode finalResult = (ObjectNode) baseResponse.deepCopy();

// 按路径创建嵌套结构
JsonNode targetPath = finalResult.at("/results/items");  // JSON Pointer

// 遍历数组进行数据匹配
if (targetPath.isArray()) {
    ArrayNode arrayNode = (ArrayNode) targetPath;
    for (JsonNode item : arrayNode) {
        String itemId = item.get("id").asText();
        
        // 在columnData中查找匹配的数据
        JsonNode matchedColumn = findByCondition(columnData, "id", itemId);
        
        // 动态添加字段
        ((ObjectNode) item).set("columnInfo", matchedColumn);
    }
}
```
结果可能是接口返回的一部分 做条件匹配
```
// 输入：数据源 + 匹配条件
JsonNode columnData = getColumnInfoResult;
String condition = "$response.results.$items.id == #getColumnInfo.$response.results.$items.id";

// 技术：HashMap建立索引 + 快速查找
Map<String, JsonNode> indexMap = new HashMap<>();

// 建立索引
if (columnData.get("results").isArray()) {
    for (JsonNode item : columnData.get("results")) {
        String key = item.get("id").asText();
        
        // 只保留需要的字段
        ObjectNode filteredItem = JsonNodeFactory.instance.objectNode();
        for (String property : ["id", "title"]) {
            filteredItem.set(property, item.get(property));
        }
        
        indexMap.put(key, filteredItem);
    }
}

// 快速匹配：O(1)时间复杂度
JsonNode matched = indexMap.get("123");
```
最终结果：
```
{
  "results": {
    "items": [
      {
        "id": "123",
        "columnInfo": {"id": "123", "title": "专栏1"}
      },
      {
        "id": "456", 
        "columnInfo": {"id": "456", "title": "专栏2"}
      }
    ]
  }
}
```