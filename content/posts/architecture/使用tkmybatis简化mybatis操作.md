---
title: "使用tkmybatis简化mybatis操作"
date: "2019-09-06"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
项目中所有的Mapper接口都继承自tkmybatis的Mapper<T>接口，不过注意这里使用的是TKMybatis的Mapper<T>接口，而不是Mybatis的Mapper<T>接口。
```
import tk.mybatis.mapper.common.Mapper;
public interface UserMapper extends Mapper<UserDO> {

    /**
     * 根据注册手机号查询记录
     *
     * @param phone
     * @return UserDO
     */
    UserDO selectByPhone(String phone);
}
```

通过继承Mapper<T>接口，自动获得以下基础CRUD方法：  
insert (插入数据)  
delete (删除数据)  
update (更新数据)  
select (查询数据)  
 
而mapper.xml里面很多都可以简化 可以去掉resultMap 返回值直接使用resultType+实体类
只需要维护你自定义的ql  
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.finance.user.dao.mapper.UserMapper">

  <sql id="Base_Column_List">
    id, phone, user_iid, identity_type, channel_id, capital_id, date_create, date_update
    ,user_name,is_test
  </sql>

  <select id="selectByPhone" parameterType="java.lang.String" resultType="com.finance.domain.user.UserDO">
    select
    <include refid="Base_Column_List" />
    from fsc_user
    where phone = #{phone,jdbcType=VARCHAR}
  </select>
</mapper>
```

实体类需要加上jpa的注解
```
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.Table;
import lombok.Data;

@Data
@Entity
@Table(name = "fsc_user")
public class UserDO {
    @Id
    private Long id;
    private String phone;
    private String userIid;
    private Integer identityType;
    private Integer channelId;
    private Integer capitalId;
    private Date dateCreate;
    private Date dateUpdate;
    private String userName;
    private Boolean isTest;
}
```
基础的CRUD方法已经被自动实现，会类似jpa一样生成sql 你只需要关注自定义的查询方法。  
主要步骤是：  
引入tkmybatis依赖  
创建实体类（如CapitalDO）  
创建Mapper接口，继承Mapper<T>  
可选择在XML文件中定义自定义SQL  
在Service层中注入Mapper接口并使用  
通过这种方式，项目可以充分利用tkmybatis提供的通用CRUD操作，同时保持自定义SQL的灵活性。  