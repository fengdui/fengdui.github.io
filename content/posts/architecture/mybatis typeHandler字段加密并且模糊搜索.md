---
title: "mybatis typeHandler字段加密并且模糊搜索"
date: "2021-09-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

xml里resultmap配置
```
<result column="responsible_person_encrypt" jdbcType="VARCHAR" 
        property="responsiblePersonEncrypt" 
        typeHandler="com.tuya.crypto.encryption.handler.EncryptedStringTypeHandler"/>
```
sql里面使用
```
<!-- 在查询条件中使用 -->
<if test="principal != null">
    AND `principal_encrypt` LIKE CONCAT("%",#{principal, 
        typeHandler=com.tuya.crypto.encryption.handler.EncryptedStringTypeHandler},"%")
</if>

<!-- 在更新语句中使用 -->
<if test="processEngineer != null">
    process_engineer_encrypt = #{processEngineer,jdbcType=VARCHAR, 
        typeHandler=com.basic.crypto.encryption.handler.EncryptedStringTypeHandler},
</if>
```
插入语句
```
<!-- 在插入语句中配置 -->
insert into issue_track (biz_type, responsible_person_encrypt, ...)
values (#{bizType,jdbcType=VARCHAR}, 
        #{responsiblePersonEncrypt,jdbcType=VARCHAR, 
         typeHandler=com.basic.crypto.encryption.handler.EncryptedStringTypeHandler}, ...)
```
MyBatis类型处理器的工作流程：
数据写入时：Handler将明文字符串加密为密文存储到数据库
数据读取时：Handler将数据库中的密文字符串解密为明文返回给Java对象
查询条件处理：在SQL查询时，Handler自动将查询条件加密后再进行匹配

如何进行模糊搜索
分片/分组加密
将数据按固定长度分组（如每2个字符一组），每组分别加密。
搜索时，将搜索词也按同样规则分组，然后对每组密文进行精确匹配