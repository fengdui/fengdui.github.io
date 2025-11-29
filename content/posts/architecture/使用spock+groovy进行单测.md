---
title: "使用spock+groovy进行单测"
date: "2019-12-22"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

JUnit: 作为主要的测试框架，用于编写和执行单元测试  
Spring TestContext Framework: 用于加载Spring上下文，支持依赖注入  
Groovy测试: 部分测试使用Groovy语言编写，位于src/test/groovy目录  
Java测试: 主要测试使用Java语言编写，位于src/test/java目录  
Mapper层测试：主要使用Java + JUnit实现  
Service层和Receiver测试：主要使用Groovy + Spock实现  
测试配置统一：无论是Java还是Groovy测试，都使用相同的applicationContext-test.xml配置  

spock+groovy示例
```
@ContextConfiguration(locations = "classpath:applicationContext-test.xml")
@Transactional(transactionManager = "transactionManager", rollbackFor = Exception.class)
@Rollback
class FscContractAppServiceImplSpec extends Specification {

    @SpringBean
    ContractApiService contractApiService = Mock() {
        singleSign(_) >> new ContractResult<ContractDTO>(success: true)
    }
    @Autowired
    FscContractAppService fscContractAppService
    @Autowired
    OrderQueryService orderQueryService

    @Unroll
    def "签订合同"() {
        when:
        fscContractAppService.signSingleContract(new SignContractDTO(orderId: orderId, captcha: "zxftest", userMobile: "15123445678",
                contractTemplateId: contractTemplateId, contractType: contractType))
        then:
        notThrown(Exception)
        where:
        orderId | contractTemplateId | contractType
        1522L   | 16                 | 30
        1734L   | 19                 | 40
        1734L   | 20                 | 30
        1917L   | 17                 | 10
        1983L   | 11                 | 40
        1412L   | 14                 | 1
    }
}
````

Spock采用了given-when-then的BDD风格结构，使测试逻辑更加清晰 使用了明确的阶段划分，提高了测试的可读性  

强大的数据驱动测试能力  
通过@Unroll注解和where块，可以轻松实现表格驱动测试 大大减少了重复代码  
更好的异常处理和断言 Spock的断言机制更加简洁直观 异常测试可以更优雅地表达  

junit+java示例
```
@RunWith(SpringJUnit4ClassRunner.class)
@ContextConfiguration("classpath:applicationContext-test.xml")
@Transactional(transactionManager = "transactionManager",rollbackFor=Exception.class)
@Rollback
public class CarPickUpMapperTest {

    @Autowired
    private CarPickUpMapper carPickUpMapper;

    @Test
    public void deleteByPrimaryKey() {
        assertTrue(carPickUpMapper.deleteByPrimaryKey(1264L) > 0);
    }
}
```
Mapper层测试  
测试数据库操作的正确性  
示例：SupplierRecordMapperTest.java SupplierRecordMapperTest.java  
Service层测试  
测试业务逻辑的正确性  
大多数使用Groovy编写，以Spec结尾  
Receiver测试  
测试消息接收和处理逻辑  
例如各种ReceiverSpec.groovy文件  