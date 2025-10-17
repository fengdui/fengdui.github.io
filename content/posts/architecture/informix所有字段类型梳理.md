---
title: "informix所有字段类型梳理"
date: "2024-08-07"
tags: ["架构"]
ShowToc: false
TocOpen: false
---




https://www.ibm.com/docs/zh/informix-servers/12.10?topic=types-datetime-data-type

| 列类型名称                                               | 是否支持脱敏   | gateway类型 |
|-----------------------------------------------------|----------|-------------|
| bigint                                              | True     | NUMBER      |
| bigserial                                           | True     | NUMBER      |
| blob                                                | False    |             |
| boolean                                             | False    |             |
| byte                                                | False    |             |
| char                                                | True     | STRING      |
| character varying                                   | False    |             |
| clob                                                | False    |             |
| date                                                | True     | DATE        |
| DATETIME largest_qualifier TO smallest_qualifier    | False    |             |
| decimal                                             | False    |             |
| double precision                                    | False    |             |
| float                                               | False    |             |
| int8                                                | True     | NUMBER      |
| integer                                             | True     | NUMBER      |
| INTERVAL largest_qualifier(n) TO smallest_qualifier | False    | |
| interval                                            | False    |             |
| lvarchar                                            | True     | VAR_STRING  |
| money                                               | True     | NUMBER      |
| numeric                                             | False    |             |
| nchar                                               | True     | STRING      |
| nvarchar                                            | True     | VAR_STRING  |
| serial                                              | True     | NUMBER      |
| serial8                                             | True     | NUMBER      |
| set                                                 | False    |             |
| smallfloat                                          | True     | NUMBER      |
| smallint                                            | True     | NUMBER      |
| text                                                | False    |             |
| varchar                                             | True     | VAR_STRING  |