---
title: "openhtmltopdf生成pdf"
date: "2026-07-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

先生成html 在将html转pdf

```
html.append("<!DOCTYPE html><html><head><meta charset=\"UTF-8\"/>")
    .append("<style>")
    .append("@page { size: A4; margin: 2cm; }")
    .append("body { font-family: 'WenQuanZhengHei', sans-serif; font-size: 12px; color: #333; line-height: 1.6; }")
    .append(".title { text-align: center; font-size: 22px; font-weight: bold; margin-bottom: 30px; }")
    .append(".info-row { display: flex; justify-content: space-between; margin-bottom: 8px; }")
    .append(".info-left { width: 70%; }")
    .append(".info-right { width: 28%; text-align: left; }")
    .append(".section-title { font-weight: bold; margin-top: 20px; margin-bottom: 8px; font-size: 13px; }")
    .append(".result-pass { color: #52c41a; font-weight: bold; }")
    .append(".result-reject { color: #f5222d; font-weight: bold; }")
    .append("table { width: 100%; border-collapse: collapse; margin-top: 8px; }")
    .append("th, td { border: 1px solid #ccc; padding: 6px; text-align: center; }")
    .append("th { background-color: #f5f5f5; }")
    .append(".member-item { margin-top: 12px; }")
    .append(".member-title { font-weight: bold; margin-bottom: 6px; }")
    .append(".member-content { padding-left: 12px; }")
    .append(".divider { border-top: 1px solid #ddd; margin: 16px 0; }")
    .append("</style></head><body>");
    ....
```

```
private byte[] htmlToPdfBytes(String html) throws Exception {
    String fontFamily = "WenQuanZhengHei";
    ByteArrayOutputStream os = new ByteArrayOutputStream();
    PdfRendererBuilder builder = new PdfRendererBuilder();
    builder.useFont(() -> {
        try {
            return new ClassPathResource("fonts/WenQuanZhengHei.ttf").getInputStream();
        } catch (IOException e) {
            throw new RuntimeException(e);
        }
    }, fontFamily);
    builder.withHtmlContent(html, null);
    builder.toStream(os);
    builder.run();
    return os.toByteArray();
}
```