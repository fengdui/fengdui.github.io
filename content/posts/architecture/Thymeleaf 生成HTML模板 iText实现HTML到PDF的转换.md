---
title: "Thymeleaf 生成HTML模板 iText实现HTML到PDF的转换"
date: "2023-02-16"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
Thymeleaf 生成HTML模板 在线预览  
iText实现HTML到PDF的转换 进行导出  

系统中已经存在用于 Web 展示的页面模板（例如订单详情、电子发票），可以直接复用这些 HTML 模板，稍作调整后用于 PDF 生成，无需重新设计一套 PDF 专用布局  
避免直接操作 PDF 底层 API 的复杂性  