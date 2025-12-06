---
title: "es geo_distance查询最近门店"
date: "2019-04-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


前端通过JavaScript调用地图服务获取用户当前位置
将经纬度坐标传递给后端
或者前端传地址 houduan 后端根据地址转成经纬度
索引
```
PUT /store_locations
{
  "mappings": {
    "properties": {
      "store_name": { "type": "text" },       // 门店名称，用于全文搜索
      "address": { "type": "keyword" },       // 详细地址，用于精确匹配
      "location": { "type": "geo_point" },    // **核心字段**：必须是 geo_point 类型
      "city": { "type": "keyword" }           // 城市，可用于过滤
    }
  },
  "settings": {
    "number_of_shards": 1,                    // 分片数，根据数据量调整
    "number_of_replicas": 1                   // 副本数，用于高可用
  }
}
```
写入
```
POST /store_locations/_doc/1
{
  "store_name": "上海中心旗舰店",
  "address": "浦东新区陆家嘴",
  "city": "上海",
  "location": { "lat": 31.2336, "lon": 121.5054 } // 使用对象格式
}
```
查询
```
GET /your_store_index/_search
{
  "query": {
    "bool": {
      "filter": {
        "geo_distance": {
          "distance": "20km",           // 初始筛选半径
          "location": {                 // 门店位置字段，需为geo_point类型
            "lat": 31.2304,            // 中心点纬度 (例如：上海)
            "lon": 121.4737            // 中心点经度
          }
        }
      }
    }
  },
  "sort": [                            // 按距离排序
    {
      "_geo_distance": {
        "location": {                  // 排序基准点，通常与筛选中心点一致
          "lat": 31.2304,
          "lon": 121.4737
        },
        "order": "asc",                // 升序，距离最近排第一
        "unit": "km",                  // 返回距离的单位
        "distance_type": "arc"         // 使用高精度球面计算
      }
    }
  ],
  "size": 1                            // 只返回最近的一个结果
}
```
distance (筛选)	初始地理围栏半径，用于缩小计算范围，提升性能。 例如 "20km"。应根据业务场景设置（如城市内可用5km，郊区可用20km）。   
location 字段	存储门店经纬度的字段，必须在索引映射中定义为 geo_point 类型。  
_geo_distance (排序)	根据文档与中心点的实际距离进行排序。 必须与 geo_distance 筛选的中心点坐标一致，以确保逻辑正确。  
distance_type	距离计算方式。arc（默认）精度更高；plane 计算更快但远距离误差大。 对精度要求高或全球数据用 arc；对速度要求极高、范围小时可考虑 plane。  
size	控制返回的文档数量。查找最近门店时设为 1。 如果需要获取前N家最近门店，可修改此值。  