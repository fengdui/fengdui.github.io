---
title: "Bpmn.js BPMN文件部署到Camunda流程引擎"
date: "2025-05-08"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

前后端对接工作流引擎一般是这种情况：
1. 前端发送BPMN文件到后端 一般是一个xml字符串
2. 后端将BPMN文件部署到Camunda流程引擎 Bpmn.js BPMN文件部署到Camunda流程引擎
3. 前端可以查询和操作部署在引擎中的流程实例
```
public Map<String, String> getProcessBpmn(String processId) {
    // 1. 从数据库获取流程信息
    Process process = processMapper.selectById(processId);
    
    // 2. 通过Camunda API获取BPMN XML
    String bpmn = processEngine.getRepositoryService()
        .getProcessModel(process.getActProcdefId());
    
    // 3. 返回BPMN字符串
    Map<String, String> result = new HashMap<>();
    result.put("bpmn", bpmn);
    return result;
}
```

然后Camunda自己维护的XML结构只知道节点A连接到节点B 数据库里面也同步维护一份 包括节点的业务配置（比如审批人是谁）
```
<bpmn:process id="process_1">
  <bpmn:startEvent id="start"/>
  <bpmn:userTask id="task1" name="审批"/>
  <bpmn:exclusiveGateway id="gateway1"/>
  <bpmn:endEvent id="end"/>
  <bpmn:sequenceFlow sourceRef="start" targetRef="task1"/>
</bpmn:process>
```
数据库里面维护成
```
{
  "nodes": [
    {
      "id": "task1",
      "assignee": ["user1", "user2"],
      "approvalType": "countersign",
      "conditions": {...}
    }
  ]
}
```
前端流程设计器需要：
节点A连接到节点B（从XML获取）
节点A的审批人是张三、李四（从数据库获取）

xml解析
```
public static List<Relation> parseXML(String xml) {
    List<Relation> resultList = new ArrayList();
    Document document = null;

    try {
        document = DocumentHelper.parseText(xml);
        Element root = document.getRootElement();
        Iterator<Element> rootIter = root.elementIterator();

        while(rootIter.hasNext()) {
            Element ele = (Element)rootIter.next();
            chile(ele, resultList);
        }
    } catch (Exception e) {
        e.printStackTrace();
    }

    return resultList;
}
```
```
public static void chile(Element ele, List<Relation> resultList) {
    Iterator<Element> childIter = ele.elementIterator();

    while(childIter.hasNext()) {
        Element attr = (Element)childIter.next();
        Relation relation = new Relation();
        if ("sequenceFlow".equals(attr.getName().trim())) {
            relation.setType("node");

            for(Attribute attribute : attr.attributes()) {
                if ("sourceRef".equals(attribute.getName().trim())) {
                    relation.setFrom(attribute.getValue().trim());
                }

                if ("targetRef".equals(attribute.getName().trim())) {
                    relation.setTo(attribute.getValue().trim());
                }

                if ("name".equals(attribute.getName().trim())) {
                    relation.setName(attribute.getValue().trim());
                }
            }

            resultList.add(relation);
        }

        if ("startEvent".equals(attr.getName().trim())) {
            relation.setType("start");

            for(Attribute attribute : attr.attributes()) {
                if ("id".equals(attribute.getName().trim())) {
                    relation.setFrom(attribute.getValue().trim());
                }
            }

            resultList.add(relation);
        }

        if ("endEvent".equals(attr.getName().trim())) {
            relation.setType("end");

            for(Attribute attribute : attr.attributes()) {
                if ("id".equals(attribute.getName().trim())) {
                    relation.setFrom(attribute.getValue().trim());
                }
            }

            resultList.add(relation);
        }
    }

}
``` 
最后做一个合并给到前端