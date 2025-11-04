---
title: "一个json扁平化小工具"
date: "2025-02-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

json
```
{
  "user": {
    "name": "张三",
    "address": {
      "city": "北京",
      "street": "长安街"
    },
    "hobbies": ["读书", "游泳"]
  },
  "age": 25
}
```
扁平化之后
```
user.name : 张三
user.address.city : 北京
user.address.street : 长安街
user.hobbies[0] : 读书
user.hobbies[1] : 游泳
age : 25
```

```
import com.alibaba.fastjson.JSONObject;
import com.github.wnameless.json.flattener.JsonFlattener;

import java.util.Map;
public class JsonUtils {

    public static void jsonFlatten(String jsonStr) {
        JSONObject jsonObj = JSONObject.parseObject(jsonStr);
        Map<String, Object> flatMap = JsonFlattener.flattenAsMap(jsonObj.toString());
        for (java.util.Map.Entry<String, Object> entry : flatMap.entrySet()) {
            System.out.println(entry.getKey() + " : " + entry.getValue());
        }

        //String unflattenJson = JsonUnflattener.unflatten(flattenJson);
    }
}
```