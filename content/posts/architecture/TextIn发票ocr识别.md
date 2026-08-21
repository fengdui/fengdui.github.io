---
title: "TextIn发票ocr识别"
date: "2026-07-01"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

```
{
  "code": 200,
  "message": "success",
  "result": {
    "object_list": [      // ← 一张图可能包含多张票据，所以是数组
      {
        "type": "vat_special_invoice",          // 票据类型代码（路由用）
        "type_description": "增值税专用发票",    // 票据类型中文
        "class_": "nation_tax_invoice",
        "item_list": [       // ← 核心：所有识别出的字段都在这，是 key-value 数组
          {"key": "vat_invoice_daima",     "value": "3300204130"},
          {"key": "vat_invoice_haoma",     "value": "08854321"},
          {"key": "vat_invoice_issue_date","value": "2026年01月12日"},
          {"key": "vat_invoice_seller_name","value": "浙江某某科技有限公司"},
          {"key": "vat_invoice_total_cover_tax_digits","value": "¥11300.00"},
          {"key": "vat_invoice_tax_rate",  "value": "13%\n9%"},
          {"key": "vat_invoice_tax_total", "value": "¥1300.00"}
        ]
      }
    ]
  }
}
```

OCR 识别出来的是 "13%"、"9%"、"6%" 这种人能看懂的文字，但财务系统（用友 NC Cloud / NCC）生成会计凭证时不认这种文字，它要的是 CN13、CN12、CN07 这种预设的内部编码（叫「税码」）。
税码映射就是做这个翻译