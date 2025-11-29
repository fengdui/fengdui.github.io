---
title: "使用ddd+spring data jpa搭建新工程"
date: "2025-11-13"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

采用了分层架构：  
domain：领域层，包含实体、聚合根  
infrastructure：基础设施层，包含仓储实现、适配器  
service：应用服务层  
model/dto：数据传输对象 避免直接暴露领域模型  
adapter: 使用适配器模式来集成外部系统  

聚合根与实体  
项目中定义了实体类(Entity)作为领域模型的核心  
```
public class AccountEntity extends AbstractAggregateRoot<AccountEntity> {
    // 实体定义
    
    // 工厂方法创建实体
    public static AccountEntity create(String phone, String nickname, Long organizationId, String organizationName, Integer role){
        AccountEntity accountEntity = new AccountEntity();
        // 设置属性
        return accountEntity;
    }
    
    // 领域行为方法
    public void changeNickname(String newNickname){
        nickname = newNickname;
    }
}
```
AbstractAggregateRoot是Spring Data框架提供的一个抽象基类，用于在领域驱动设计(DDD)中实现聚合根模式  
事件发布机制：AbstractAggregateRoot内置了领域事件发布功能，允许聚合根在状态变更时发布领域事件   
聚合根可以通过registerEvent()方法注册事件，当聚合根被保存时，这些事件会被自动发布  
聚合根标识：通过继承该类，明确标识一个实体为聚合根，作为聚合的入口点和一致性边界。作为聚合的根，负责维护整个聚合的一致性和领域规则  
Spring Data集成：提供与Spring Data框架的无缝集成，简化仓储层的实现  

领域服务(Domain Service) 领域服务来处理跨实体的业务逻辑  
```
@Service("carModelQueryService")
public class CarModelQueryServiceImpl implements CarModelQueryService {
    // 领域服务实现...
}
```
仓储层(Repository) spring data jpa 提供了JpaRepository接口，用于定义仓储方法
```
@Autowired
private CarModelQueryRepository carModelQueryRepository;

public interface CustomerManagerRepository extends JpaRepository<CustomerManagerEntity, Long> {
    String COLUMNS = "id, account_id, manager_name, phone, manager_email, enable, org_id, dealer_id, is_deleted, date_create, date_update ";
    String PAGE_QUERY_WHERE = "where (manager_name like CONCAT('%',:#{#param.nameOrPhone},'%') or phone like CONCAT('%',:#{#param.nameOrPhone},'%')  or :#{#param.nameOrPhone} is null) and (enable = :#{#param.enable} or :#{#param.enable} is null) and (org_id = :#{#param.orgId} or :#{#param.orgId} is null) and (date_create >= :#{#param.startDate} or :#{#param.startDate} is null) and (date_create <= :#{#param.endDate} or :#{#param.endDate} is null) and (dealer_id = :#{#param.dealerId})";

    @Query(
        value = "select id, account_id, manager_name, phone, manager_email, enable, org_id, dealer_id, is_deleted, date_create, date_update  from ef_customer_manager_org_rel where (manager_name like CONCAT('%',:#{#param.nameOrPhone},'%') or phone like CONCAT('%',:#{#param.nameOrPhone},'%')  or :#{#param.nameOrPhone} is null) and (enable = :#{#param.enable} or :#{#param.enable} is null) and (org_id = :#{#param.orgId} or :#{#param.orgId} is null) and (date_create >= :#{#param.startDate} or :#{#param.startDate} is null) and (date_create <= :#{#param.endDate} or :#{#param.endDate} is null) and (dealer_id = :#{#param.dealerId})",
        countQuery = "select count(*) from ef_customer_manager_org_rel where (manager_name like CONCAT('%',:#{#param.nameOrPhone},'%') or phone like CONCAT('%',:#{#param.nameOrPhone},'%')  or :#{#param.nameOrPhone} is null) and (enable = :#{#param.enable} or :#{#param.enable} is null) and (org_id = :#{#param.orgId} or :#{#param.orgId} is null) and (date_create >= :#{#param.startDate} or :#{#param.startDate} is null) and (date_create <= :#{#param.endDate} or :#{#param.endDate} is null) and (dealer_id = :#{#param.dealerId})",
        nativeQuery = true
    )
    Page<CustomerManagerEntity> pageByCondition(@Param("param") QueryCustomerManagerDTO var1, Pageable var2);

    CustomerManagerEntity findByDealerIdAndPhone(Long var1, String var2);

    @Modifying
    @Query(
        value = "update ef_customer_manager_org_rel set enable = :enable where dealer_id = :dealerId and phone = :phone",
        nativeQuery = true
    )
    Integer enableOrDisableByDealerId(@Param("dealerId") Long var1, @Param("enable") Integer var2, @Param("phone") String var3);

    CustomerManagerEntity findByAccountId(@Param("accountId") Long var1);

    Integer countByDealerIdAndEnable(@Param("dealerId") Long var1, @Param("enable") Integer var2);
}
```
储层只与聚合根交互，通过AbstractAggregateRoot，Spring Data可以自动处理聚合根的持久化和事件发布，简化了代码实现