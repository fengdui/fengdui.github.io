---
title: "配置Thymeleaf生成静态h5页面"
date: "2020-05-10"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

用户请求 → 路由匹配 → 权限验证 → 内容获取 → 动态渲染 → 页面输出

Page (页面) ←→ Template (模板) ←→ Content (内容)  
Include (片段) ←→ Template (模板) ←→ Content (内容)  
ViewApi (接口) ←→ Content (内容)  
Message (消息) ←→ Template (模板) ←→ Content (内容)  

- 一对一关系 每个Page/Include/Message都有独立的Template
- 版本管理 Template支持预览版和发布版双内容
- 内容分离 实际内容存储在Content表中
- 组件复用 Include通过名称机制实现跨页面复用

三层内容结构  
1.页面层 (Page)  
├── 页面配置 (URI、标题、权限)  
└── 页面模板 → 页面内容 (HTML + Include占位符)  
2.组件层 (Include)  
├── 组件配置 (名称、标题)  
└── 组件模板 → 组件内容 (HTML片段)  
3.接口层 (ViewApi)  
├── 接口配置 (URI、权限、缓存)  
└── 接口内容 (JSON数据)  

```
// === 第一层：获取Page内容 ===
// 1. 根据URI获取Page
Page page = pageService.getPageByUri("/product/list");

// 2. 获取Page的Template
Template pageTemplate = templateRepository.findOne(page.getTemplateId()); // Template ID: 1001

// 3. 获取Page的Content
long pageContentId = pageTemplate.getPreviewContentId(); // Content ID: 2001
String pageContent = contentService.get(pageContentId);
// pageContent = "<html><body>CHD_VIEW_INCLUDE_header<div>页面内容</div></body></html>"

// === 第二层：替换Include内容 ===
// 4. 处理Include占位符
content = StringUtil.replace(pageContent, ModuleConsts.VIEW_INCLUDE_PATTERN, (matched) -> {
    String includeName = matched.substring("CHD_VIEW_INCLUDE_".length()); // "header"
    
    // 5. 获取Include的Template信息
    ContentIdsCacheVo includeCache = includeService.getCacheByName(includeName);
    // 内部流程：
    // Include include = includeRepository.getByName("header");
    // Template includeTemplate = templateRepository.findOne(include.getTemplateId()); // Template ID: 1002
    // includeCache.set(includeTemplate); // 设置previewContentId=2002, publishedContentId=0
    
    if (includeCache != null) {
        // 6. 获取Include的实际Content
        long includeContentId = includeCache.getContentId(preview); // Content ID: 2002
        return contentService.get(includeContentId);
        // 返回: "<header><nav>导航菜单</nav></header>"
    }
    return matched;
});

// 最终结果
// content = "<html><body><header><nav>导航菜单</nav></header><div>页面内容</div></body></html>"
```