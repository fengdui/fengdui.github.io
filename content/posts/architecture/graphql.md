---
title: "graphql"
date: "2024-11-19"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


```
/**
 * 示例GraphQL服务接口
 */
public interface DBWServiceExample extends DBWService {
    /**
     * 获取示例数据
     */
    String getExampleData(@NotNull WebSession webSession, @NotNull String input) throws DBWebException;
    /**
     * 创建示例数据
     */
    boolean createExampleData(@NotNull WebSession webSession, @NotNull String name, int value) throws DBWebException;
}
```
创建 GraphQL 绑定类，负责将服务方法映射到 GraphQL 查询和变更

```
/**
 * 示例GraphQL服务绑定
 */
public class WebServiceBindingExample extends WebServiceBindingBase<DBWServiceExample> {
    private static final String SCHEMA_FILE_NAME = "schema/service.example.graphqls";
    public WebServiceBindingExample() {
        super(DBWServiceExample.class, new WebServiceExample(), SCHEMA_FILE_NAME);
    }
    @Override
    public void bindWiring(DBWBindingContext model) throws DBWebException {
        // 绑定查询
        model.getQueryType()
            .dataFetcher("exampleData", 
                env -> getService(env).getExampleData(
                    getWebSession(env), 
                    env.getArgument("input")
                )
            );
        // 绑定变更
        model.getMutationType()
            .dataFetcher("createExampleData",
                env -> getService(env).createExampleData(
                    getWebSession(env),
                    env.getArgument("name"),
                    env.getArgument("value")
                )
            );
    }
}
```
创建 GraphQL schema 文件来定义接口：
```
# 示例数据类型
type ExampleData {
    message: String!
    timestamp: DateTime!
}
extend type Query {
    "获取示例数据"
    exampleData(input: String!): String!
}
extend type Mutation {
    "创建示例数据"
    createExampleData(name: String!, value: Int!): Boolean!
}
```
接收请求
```
private void executeQuery(
        @NotNull HttpServletRequest request,
        @NotNull HttpServletResponse response,
        @NotNull String query,
        @Nullable Map<String, Object> variables,
        @Nullable String operationName
    ) throws IOException {
        Map<String, Object> mapOfContext =
            Map.of(
                "request", request,
                "response", response,
                "bindingContext", bindingContext
            );
        ExecutionInput.Builder contextBuilder = ExecutionInput.newExecutionInput()
            .graphQLContext(mapOfContext)
            .query(query);
        if (variables != null) {
            contextBuilder.variables(variables);
        }
        if (operationName != null) {
            contextBuilder.operationName(operationName);
        }
        String sessionId = GraphQLLoggerUtil.getSmSessionId(request);
        String userId = GraphQLLoggerUtil.getUserId(request);
        String loggerMessage = GraphQLLoggerUtil.buildLoggerMessage(sessionId, userId, variables);
        if (operationName != null) {
            log.debug("API > " + operationName + loggerMessage);
        } else if (DEBUG) {
            log.debug("API > " + query + loggerMessage);
        }
        LocalDateTime startTime = LocalDateTime.now();
        ExecutionInput executionInput = contextBuilder.build();
        ExecutionResult executionResult = null;
        Exception executionException = null;
        try {
            executionResult = graphQL.execute(executionInput);
        } catch (Exception e) {
            executionException = e;
            throw e;
        } finally {
            String errorMessage = null;
            if (executionResult != null && executionResult.getErrors() != null && !executionResult.getErrors().isEmpty()) {
                errorMessage = executionResult.getErrors().getFirst().getMessage();
            } else if (executionException != null) {
                errorMessage = executionException.getMessage();
            }
            if (WebAppUtils.getWebApplication() instanceof ApiCallInterceptor apiCallInterceptor) {
                apiCallInterceptor.onApiCallEvent(
                    request,
                    variables,
                    CommonUtils.notEmpty(operationName), userId, startTime,
                    errorMessage,
                    API_PROTOCOL
                );
            }
        }

        if (executionResult != null) {
            Map<String, Object> resJSON = executionResult.toSpecification();
            String resString = gson.toJson(resJSON);
            setDevelHeaders(request, response);
            response.setContentType(GraphQLConstants.CONTENT_TYPE_JSON_UTF8);
            response.getWriter().print(resString);
        }
    }
```
用到了operationName