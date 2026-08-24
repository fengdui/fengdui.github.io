

Dify 工作台一共 6 种常见的"应用/模块"创建类型，项目里对应情况列一下，没用的也标出来：

1. 聊天助手（Chat App / Chatbot）
工作台里创建按钮就是"聊天助手"，纯聊天，不带工具、不带编排
is_agent=false 时走的是聊天助手

2. 智能体（Agent）
工作台里创建类型选"Agent / 智能体"，特点是让大模型自己判断要不要调用外部工具（ReAct 思维链）
DifyChatClient请求体里带 is_agent=true 字段，值从 DifyAgent 表的 agentType 推出来

3. 聊天工作流（Chatflow / Chat Workflow）—— 用了
工作台里创建类型选"聊天工作流"，有聊天入口但背后是节点图编排（意图识别→RAG→大模型→分支→输出）
调的是 DifyClientFactory.createChatWorkflowClient → DifyChatflowClient.sendChatMessageStream

4. 工作流（Workflow）
DifyWorkflowClient 工作台里创建类型选"工作流"，没有聊天窗口，是单次 POST /v1/workflows/{id}/run 拿同步结果

5. 文本生成（Completion App / 文章续写类）
DifyCompletionClient "文本生成/补全"应用，走 /v1/completion-messages 接口（新版本很多被合并进聊天助手里了）

6. 数据集/知识库（Dataset）—— （虽然不是"应用类型"，但它是工作台独立大模块）
工作台里左上角那个"知识库 / 数据集"tab，专门做文档上传、切分、embedding、向量入库
DifyClientFactory.createDatasetsClient → DifyDatasetsClient.createDocumentByFile

```
public String createDocument(DifyDatasetsClient datasetsClient, File file, String datasetId) throws IOException, DifyApiException {
    try {
        RetrievalModel.RerankingModel rerankingModel = new RetrievalModel.RerankingModel();
        RetrievalModel retrievalModel = new RetrievalModel();
        retrievalModel.setSearchMethod("hybrid_search");
        retrievalModel.setRerankingEnable(true);
        retrievalModel.setRerankingModel(rerankingModel);
        retrievalModel.setTopK(2);
        retrievalModel.setScoreThresholdEnabled(false);
        ProcessRule.PreProcessingRule removeSpaces = PreProcessingRule.builder().id("remove_extra_spaces").enabled(true).build();
        ProcessRule.PreProcessingRule removeUrls = PreProcessingRule.builder().id("remove_urls_emails").enabled(false).build();
        ProcessRule.Segmentation segmentation = Segmentation.builder().separator("\n").maxTokens(1000).build();
        ProcessRule.SubchunkSegmentation subchunkSegmentation = SubchunkSegmentation.builder().separator("\n").maxTokens(1000).build();
        ProcessRule.Rules rules = Rules.builder().preProcessingRules(Arrays.asList(removeSpaces, removeUrls)).segmentation(segmentation).subchunkSegmentation(subchunkSegmentation).parentMode("full-doc").build();
        ProcessRule processRule = ProcessRule.builder().mode("custom").rules(rules).build();
        CreateDocumentByFileRequest request = CreateDocumentByFileRequest.builder().indexingTechnique("high_quality").docForm("hierarchical_model").docLanguage("Chinese").retrievalModel(retrievalModel).processRule(processRule).build();
        DocumentResponse response = datasetsClient.createDocumentByFile(datasetId, request, file);
        return response.getDocument().getId();
    } catch (DifyApiException e) {
        log.error("Error creating document in Dify dataset: {}", e.getMessage(), e);
        return null;
    }
}
```