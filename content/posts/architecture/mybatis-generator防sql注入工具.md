---
title: "mybatis-generator防sql注入工具"
date: "2018-01-31"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


mybatis-generator防sql注入工具 生成的example java类传入字段的时候进行正则表达式匹配  
example.createCriteria().andUsernameEqualTo(username)），系统会：  
先将你的输入传给 SqlInjectUtils.isValid() 检查  
如果发现有危险关键词，立即抛出异常，阻止执行  
如果安全，才会真正添加这个查询条件  

整个执行流程可以概括为：

MBG 启动 → 加载所有插件  
↓
SqlInjectPlugin.validate() → 确认插件有效  
↓
SqlInjectPlugin.contextGenerateAdditionalJavaFiles() → 生成 SqlInjectUtils 工具类  
↓
IntrospectedTableMyBatis3Impl.calculateJavaModelGenerators() → 检查是否存在 SqlInjectPlugin  
↓
如果存在 → 使用 SqlInjectExampleGenerator 生成带注入防护的 Example 类  
↓
生成的 Example 类中包含对 SqlInjectUtils.isValid() 的调用  


思路:
SqlInjectPlugin 请求了SqlInjectUtilsGenerator 动态生成生成SqlInjectUtils.java工具类
```
public class SqlInjectPlugin extends PluginAdapter{
    /**
     * This method is called after all the setXXX methods are called, but before
     * any other method is called. This allows the plugin to determine whether
     * it can run or not. For example, if the plugin requires certain properties
     * to be set, and the properties are not set, then the plugin is invalid and
     * will not run.
     *
     * @param warnings add strings to this list to specify warnings. For example, if
     *                 the plugin is invalid, you should specify why. Warnings are
     *                 reported to users after the completion of the run.
     * @return true if the plugin is in a valid state. Invalid plugins will not
     * be called
     */
    public boolean validate(List<String> warnings) {
        return true;
    }

    @Override
    public List<GeneratedJavaFile> contextGenerateAdditionalJavaFiles() {
        SqlInjectUtilsGenerator generator = new SqlInjectUtilsGenerator();
        generator.setContext(getContext());
        List<CompilationUnit> compilationUnits = generator.getCompilationUnits();
        List<GeneratedJavaFile> answer = new ArrayList<GeneratedJavaFile>();
        for (CompilationUnit compilationUnit : compilationUnits) {
            GeneratedJavaFile gjf = new GeneratedJavaFile(compilationUnit,
                    context.getJavaModelGeneratorConfiguration()
                            .getTargetProject(),
                    context.getProperty(PropertyRegistry.CONTEXT_JAVA_FILE_ENCODING),
                    context.getJavaFormatter());
            answer.add(gjf);
        }
        return answer;
    }

    @Override
    public List<GeneratedJavaFile> contextGenerateAdditionalJavaFiles(IntrospectedTable introspectedTable) {
        return super.contextGenerateAdditionalJavaFiles(introspectedTable);
    }
}
```

SqlInjectUtilsGenerator 
```java 
public class SqlInjectUtilsGenerator extends AbstractJavaGenerator {
    public List<CompilationUnit> getCompilationUnits() {

        List<CompilationUnit> answer = new ArrayList<CompilationUnit>();
        JavaModelGeneratorConfiguration config = context
                .getJavaModelGeneratorConfiguration();

        StringBuilder sb = new StringBuilder();
        sb.append(config.getTargetPackage());
        sb.append(sb.length()>0?".":"");
        sb.append("SqlInjectUtils");
        TopLevelClass topLevelClass = new TopLevelClass(sb.toString());
        topLevelClass.setVisibility(JavaVisibility.PUBLIC);
        topLevelClass.addImportedType(new FullyQualifiedJavaType("java.util.regex.Pattern"));
        topLevelClass.addImportedType(new FullyQualifiedJavaType("java.io.FileInputStream"));
        topLevelClass.addImportedType(new FullyQualifiedJavaType("java.util.Properties"));
        topLevelClass.addImportedType(new FullyQualifiedJavaType("java.io.InputStream"));

        Field field = new Field();
        field.setVisibility(JavaVisibility.PUBLIC);
        field.setType(FullyQualifiedJavaType.getStringInstance());
        field.setName("DEFAULT_REGEXP"); //$NON-NLS-1$
        field.setStatic(true);
        field.setInitializationString("\"\\\\b(and|or)\\\\b.{1,6}?(=|>|<|\\\\bin\\\\b|\\\\blike\\\\b)|/\\\\*.+?\\\\*/|<\\\\s*script\\\\b|\\\\bEXEC\\\\b|UNION.+?SELECT|UPDATE.+?SET|INSERT\\\\s+INTO.+?VALUES|(SELECT|DELETE).+?FROM|(CREATE|ALTER|DROP|TRUNCATE)\\\\s+(TABLE|DATABASE)\"");
        topLevelClass.addField(field);

        field = new Field();
        field.setVisibility(JavaVisibility.PUBLIC);
        field.setType(FullyQualifiedJavaType.getStringInstance());
        field.setName("USE_REGEXP"); //$NON-NLS-1$
        field.setStatic(true);
        topLevelClass.addField(field);

        field = new Field();
        field.setVisibility(JavaVisibility.PUBLIC);
        field.setType(new FullyQualifiedJavaType("java.util.regex.Pattern"));
        field.setName("SQL_PATTERN"); //$NON-NLS-1$
        field.setStatic(true);
        field.setVolatile(true);
        topLevelClass.addField(field);

        field = new Field();
        field.setVisibility(JavaVisibility.PUBLIC);
        field.setType(FullyQualifiedJavaType.getStringInstance());
        field.setName("DEFAULT_FILE_PATHS"); //$NON-NLS-1$
        field.setStatic(true);
        field.setInitializationString("\"sql_inject_regex.properties\"");
        topLevelClass.addField(field);

        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("init"); //$NON-NLS-1$
        method.addBodyLine("Properties properties = new Properties();");
        method.addBodyLine("try (InputStream in = new FileInputStream(DEFAULT_FILE_PATHS)){");
        method.addBodyLine("properties.load(in);");
        method.addBodyLine("String customRegexP = properties.getProperty(\"regexp\");");
        method.addBodyLine("if (customRegexP != null) {");
        method.addBodyLine("USE_REGEXP = customRegexP;");
        method.addBodyLine("} else {");
        method.addBodyLine("USE_REGEXP = DEFAULT_REGEXP;");
        method.addBodyLine("}");
        method.addBodyLine("} catch (Exception e) {");
        method.addBodyLine("USE_REGEXP = DEFAULT_REGEXP;");
        method.addBodyLine("} finally {");
        method.addBodyLine("SQL_PATTERN = Pattern.compile(USE_REGEXP, Pattern.CASE_INSENSITIVE);");
        method.addBodyLine("}");
        method.setStatic(true);
        method.setSynchronized(true);
        topLevelClass.addMethod(method);
        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("isValid"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType.getObjectInstance(), "obj")); //$NON-NLS-1$
        method.addBodyLine("if (SQL_PATTERN == null) {");
        method.addBodyLine("synchronized (SqlInjectUtils.class) {");
        method.addBodyLine("if (SQL_PATTERN == null)");
        method.addBodyLine("init();");
        method.addBodyLine("}");
        method.addBodyLine("}");
        method.addBodyLine("if (obj instanceof String) {");
        method.addBodyLine("if (SQL_PATTERN.matcher((String)obj).find()) {");
        method.addBodyLine("throw new RuntimeException(obj+\" 存在sql注入风险\");");
        method.addBodyLine("}");
        method.addBodyLine("}");
        method.setStatic(true);

        topLevelClass.addMethod(method);

        answer.add(topLevelClass);
        return answer;
    }
}
```

IntrospectedTableMyBatis3Impl 改写calculateJavaModelGenerators方法 判断this.useSqlInjectPlugin()是否为true true的话使用SqlInjectExampleGenerator
```
protected void calculateJavaModelGenerators(List<String> warnings,
            ProgressCallback progressCallback) {
    if (getRules().generateExampleClass()) {
        AbstractJavaGenerator javaGenerator = null;
        if (this.useSqlInjectPlugin()) {
            javaGenerator = new SqlInjectExampleGenerator();
        } else {
            javaGenerator = new ExampleGenerator();
        }
        initializeAbstractGenerator(javaGenerator, warnings,
                progressCallback);
        javaModelGenerators.add(javaGenerator);
    }

    if (getRules().generatePrimaryKeyClass()) {
        AbstractJavaGenerator javaGenerator = new PrimaryKeyGenerator();
        initializeAbstractGenerator(javaGenerator, warnings,
                progressCallback);
        javaModelGenerators.add(javaGenerator);
    }

    if (getRules().generateBaseRecordClass()) {
        AbstractJavaGenerator javaGenerator = new BaseRecordGenerator();
        initializeAbstractGenerator(javaGenerator, warnings,
                progressCallback);
        javaModelGenerators.add(javaGenerator);
    }

    if (getRules().generateRecordWithBLOBsClass()) {
        AbstractJavaGenerator javaGenerator = new RecordWithBLOBsGenerator();
        initializeAbstractGenerator(javaGenerator, warnings,
                progressCallback);
        javaModelGenerators.add(javaGenerator);
    }
}

private boolean useSqlInjectPlugin() {
    List<Plugin> plugins = ((PluginAggregator)context.getPlugins()).plugins;
    for (Plugin plugin : plugins) {
        if (plugin instanceof SqlInjectPlugin) {
            return true;
        }
    }
    return false;
}
```

SqlInjectExampleGenerator 就是一顿字符串拼接 可以对比ExampleGenerator看改动点  
在生成的代码片段 criteria.add(new Criterion(condition));之前会加上一段代码 SqlInjectUtils.isValid(condition)  

<details>
<summary>点击展开/折叠代码</summary>

```java
public class SqlInjectExampleGenerator extends AbstractJavaGenerator {
    public SqlInjectExampleGenerator() {
        super();
    }

    public List<CompilationUnit> getCompilationUnits() {
        JavaModelGeneratorConfiguration config = context
                .getJavaModelGeneratorConfiguration();
        StringBuilder sb = new StringBuilder();
        sb.append(config.getTargetPackage());
        sb.append(sb.length()>0?".":"");
        sb.append("SqlInjectUtils.isValid");

        FullyQualifiedTable table = introspectedTable.getFullyQualifiedTable();
        progressCallback.startTask(getString(
                "Progress.6", table.toString())); //$NON-NLS-1$
        CommentGenerator commentGenerator = context.getCommentGenerator();

        FullyQualifiedJavaType type = new FullyQualifiedJavaType(
                introspectedTable.getExampleType());
        TopLevelClass topLevelClass = new TopLevelClass(type);
        topLevelClass.setVisibility(JavaVisibility.PUBLIC);
        commentGenerator.addJavaFileComment(topLevelClass);
        // topLevelClass.addImportedType(sb.toString());
        // add default constructor
        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setConstructor(true);
        method.setName(type.getShortName());
        method.addBodyLine("oredCriteria = new ArrayList<Criteria>();"); //$NON-NLS-1$

        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        // add field, getter, setter for orderby clause
        Field field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);
        field.setType(FullyQualifiedJavaType.getStringInstance());
        field.setName("orderByClause"); //$NON-NLS-1$
        commentGenerator.addFieldComment(field, introspectedTable);
        topLevelClass.addField(field);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("setOrderByClause"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "orderByClause")); //$NON-NLS-1$
        //add
        method.addBodyLine(sb.toString()+"(orderByClause);");
        method.addBodyLine("this.orderByClause = orderByClause;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setReturnType(FullyQualifiedJavaType.getStringInstance());
        method.setName("getOrderByClause"); //$NON-NLS-1$
        method.addBodyLine("return orderByClause;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        // add field, getter, setter for distinct
        field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);
        field.setType(FullyQualifiedJavaType.getBooleanPrimitiveInstance());
        field.setName("distinct"); //$NON-NLS-1$
        commentGenerator.addFieldComment(field, introspectedTable);
        topLevelClass.addField(field);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("setDistinct"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getBooleanPrimitiveInstance(), "distinct")); //$NON-NLS-1$
        method.addBodyLine("this.distinct = distinct;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setReturnType(FullyQualifiedJavaType
                .getBooleanPrimitiveInstance());
        method.setName("isDistinct"); //$NON-NLS-1$
        method.addBodyLine("return distinct;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        // add field and methods for the list of ored criteria
        field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);

        FullyQualifiedJavaType fqjt = new FullyQualifiedJavaType(
                "java.util.List<Criteria>"); //$NON-NLS-1$
        field.setType(fqjt);
        field.setName("oredCriteria"); //$NON-NLS-1$
        commentGenerator.addFieldComment(field, introspectedTable);
        topLevelClass.addField(field);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setReturnType(fqjt);
        method.setName("getOredCriteria"); //$NON-NLS-1$
        method.addBodyLine("return oredCriteria;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("or"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getCriteriaInstance(), "criteria")); //$NON-NLS-1$
        method.addBodyLine("oredCriteria.add(criteria);"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("or"); //$NON-NLS-1$
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        method.addBodyLine("Criteria criteria = createCriteriaInternal();"); //$NON-NLS-1$
        method.addBodyLine("oredCriteria.add(criteria);"); //$NON-NLS-1$
        method.addBodyLine("return criteria;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("createCriteria"); //$NON-NLS-1$
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        method.addBodyLine("Criteria criteria = createCriteriaInternal();"); //$NON-NLS-1$
        method.addBodyLine("if (oredCriteria.size() == 0) {"); //$NON-NLS-1$
        method.addBodyLine("oredCriteria.add(criteria);"); //$NON-NLS-1$
        method.addBodyLine("}"); //$NON-NLS-1$
        method.addBodyLine("return criteria;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("createCriteriaInternal"); //$NON-NLS-1$
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        method.addBodyLine("Criteria criteria = new Criteria();"); //$NON-NLS-1$
        method.addBodyLine("return criteria;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("clear"); //$NON-NLS-1$
        method.addBodyLine("oredCriteria.clear();"); //$NON-NLS-1$
        method.addBodyLine("orderByClause = null;"); //$NON-NLS-1$
        method.addBodyLine("distinct = false;"); //$NON-NLS-1$
        commentGenerator.addGeneralMethodComment(method, introspectedTable);
        topLevelClass.addMethod(method);

        // now generate the inner class that holds the AND conditions
        topLevelClass
                .addInnerClass(getGeneratedCriteriaInnerClass(topLevelClass));

        topLevelClass.addInnerClass(getCriteriaInnerClass(topLevelClass));

        topLevelClass.addInnerClass(getCriterionInnerClass(topLevelClass));

        List<CompilationUnit> answer = new ArrayList<CompilationUnit>();
        if (context.getPlugins().modelExampleClassGenerated(
                topLevelClass, introspectedTable)) {
            answer.add(topLevelClass);
        }
        return answer;
    }

    private InnerClass getCriterionInnerClass(TopLevelClass topLevelClass) {
        Field field;
        Method method;

        InnerClass answer = new InnerClass(new FullyQualifiedJavaType(
                "Criterion")); //$NON-NLS-1$
        answer.setVisibility(JavaVisibility.PUBLIC);
        answer.setStatic(true);
        context.getCommentGenerator().addClassComment(answer,
                introspectedTable);

        field = new Field();
        field.setName("condition"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getStringInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("value"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getObjectInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("secondValue"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getObjectInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("noValue"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getBooleanPrimitiveInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("singleValue"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getBooleanPrimitiveInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("betweenValue"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getBooleanPrimitiveInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("listValue"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getBooleanPrimitiveInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        field = new Field();
        field.setName("typeHandler"); //$NON-NLS-1$
        field.setType(FullyQualifiedJavaType.getStringInstance());
        field.setVisibility(JavaVisibility.PRIVATE);
        answer.addField(field);
        answer.addMethod(getGetter(field));

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("Criterion"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addBodyLine("super();"); //$NON-NLS-1$
        method.addBodyLine("this.condition = condition;"); //$NON-NLS-1$
        method.addBodyLine("this.typeHandler = null;"); //$NON-NLS-1$
        method.addBodyLine("this.noValue = true;"); //$NON-NLS-1$
        answer.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("Criterion"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "typeHandler")); //$NON-NLS-1$
        method.addBodyLine("super();"); //$NON-NLS-1$
        method.addBodyLine("this.condition = condition;"); //$NON-NLS-1$
        method.addBodyLine("this.value = value;"); //$NON-NLS-1$
        method.addBodyLine("this.typeHandler = typeHandler;"); //$NON-NLS-1$
        method.addBodyLine("if (value instanceof List<?>) {"); //$NON-NLS-1$
        method.addBodyLine("this.listValue = true;"); //$NON-NLS-1$
        method.addBodyLine("} else {"); //$NON-NLS-1$
        method.addBodyLine("this.singleValue = true;"); //$NON-NLS-1$
        method.addBodyLine("}"); //$NON-NLS-1$
        answer.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("Criterion"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value")); //$NON-NLS-1$
        method.addBodyLine("this(condition, value, null);"); //$NON-NLS-1$
        answer.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("Criterion"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "secondValue")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "typeHandler")); //$NON-NLS-1$
        method.addBodyLine("super();"); //$NON-NLS-1$
        method.addBodyLine("this.condition = condition;"); //$NON-NLS-1$
        method.addBodyLine("this.value = value;"); //$NON-NLS-1$
        method.addBodyLine("this.secondValue = secondValue;"); //$NON-NLS-1$
        method.addBodyLine("this.typeHandler = typeHandler;"); //$NON-NLS-1$
        method.addBodyLine("this.betweenValue = true;"); //$NON-NLS-1$
        answer.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("Criterion"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "secondValue")); //$NON-NLS-1$
        method.addBodyLine("this(condition, value, secondValue, null);"); //$NON-NLS-1$
        answer.addMethod(method);

        return answer;
    }

    private InnerClass getCriteriaInnerClass(TopLevelClass topLevelClass) {
        Method method;

        InnerClass answer = new InnerClass(FullyQualifiedJavaType
                .getCriteriaInstance());

        answer.setVisibility(JavaVisibility.PUBLIC);
        answer.setStatic(true);
        answer.setSuperClass(FullyQualifiedJavaType
                .getGeneratedCriteriaInstance());

        context.getCommentGenerator().addClassComment(answer,
                introspectedTable, true);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("Criteria"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addBodyLine("super();"); //$NON-NLS-1$
        answer.addMethod(method);

        return answer;
    }

    private InnerClass getGeneratedCriteriaInnerClass(
            TopLevelClass topLevelClass) {

        JavaModelGeneratorConfiguration config = context
                .getJavaModelGeneratorConfiguration();
        StringBuilder sqlsb = new StringBuilder();
        sqlsb.append(config.getTargetPackage());
        sqlsb.append(sqlsb.length()>0?".":"");
        sqlsb.append("SqlInjectUtils.isValid");

        Field field;
        Method method;

        InnerClass answer = new InnerClass(FullyQualifiedJavaType
                .getGeneratedCriteriaInstance());

        answer.setVisibility(JavaVisibility.PROTECTED);
        answer.setStatic(true);
        answer.setAbstract(true);
        context.getCommentGenerator().addClassComment(answer,
                introspectedTable);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("GeneratedCriteria"); //$NON-NLS-1$
        method.setConstructor(true);
        method.addBodyLine("super();"); //$NON-NLS-1$
        method.addBodyLine("criteria = new ArrayList<Criterion>();"); //$NON-NLS-1$
        answer.addMethod(method);

        List<String> criteriaLists = new ArrayList<String>();
        criteriaLists.add("criteria"); //$NON-NLS-1$

        for (IntrospectedColumn introspectedColumn : introspectedTable
                .getNonBLOBColumns()) {
            if (stringHasValue(introspectedColumn
                    .getTypeHandler())) {
                String name = addtypeHandledObjectsAndMethods(
                        introspectedColumn, method, answer);
                criteriaLists.add(name);
            }
        }

        // now generate the isValid method
        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("isValid"); //$NON-NLS-1$
        method.setReturnType(FullyQualifiedJavaType
                .getBooleanPrimitiveInstance());
        StringBuilder sb = new StringBuilder();
        Iterator<String> strIter = criteriaLists.iterator();
        sb.append("return "); //$NON-NLS-1$
        sb.append(strIter.next());
        sb.append(".size() > 0"); //$NON-NLS-1$
        if (!strIter.hasNext()) {
            sb.append(';');
        }
        method.addBodyLine(sb.toString());
        while (strIter.hasNext()) {
            sb.setLength(0);
            OutputUtilities.javaIndent(sb, 1);
            sb.append("|| "); //$NON-NLS-1$
            sb.append(strIter.next());
            sb.append(".size() > 0"); //$NON-NLS-1$
            if (!strIter.hasNext()) {
                sb.append(';');
            }
            method.addBodyLine(sb.toString());
        }
        answer.addMethod(method);

        // now generate the getAllCriteria method
        if (criteriaLists.size() > 1) {
            field = new Field();
            field.setName("allCriteria"); //$NON-NLS-1$
            field.setType(new FullyQualifiedJavaType("List<Criterion>")); //$NON-NLS-1$
            field.setVisibility(JavaVisibility.PROTECTED);
            answer.addField(field);
        }

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setName("getAllCriteria"); //$NON-NLS-1$
        method.setReturnType(new FullyQualifiedJavaType("List<Criterion>")); //$NON-NLS-1$
        if (criteriaLists.size() < 2) {
            method.addBodyLine("return criteria;"); //$NON-NLS-1$
        } else {
            method.addBodyLine("if (allCriteria == null) {"); //$NON-NLS-1$
            method.addBodyLine("allCriteria = new ArrayList<Criterion>();"); //$NON-NLS-1$

            strIter = criteriaLists.iterator();
            while (strIter.hasNext()) {
                method.addBodyLine(String.format("allCriteria.addAll(%s);", strIter.next())); //$NON-NLS-1$
            }

            method.addBodyLine("}"); //$NON-NLS-1$
            method.addBodyLine("return allCriteria;"); //$NON-NLS-1$
        }
        answer.addMethod(method);

        // now we need to generate the methods that will be used in the SqlMap
        // to generate the dynamic where clause
        topLevelClass.addImportedType(FullyQualifiedJavaType
                .getNewListInstance());
        topLevelClass.addImportedType(FullyQualifiedJavaType
                .getNewArrayListInstance());

        field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);
        FullyQualifiedJavaType listOfCriterion = new FullyQualifiedJavaType(
                "java.util.List<Criterion>"); //$NON-NLS-1$
        field.setType(listOfCriterion);
        field.setName("criteria"); //$NON-NLS-1$
        answer.addField(field);

        method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setReturnType(field.getType());
        method.setName(getGetterMethodName(field.getName(), field
                .getType()));
        method.addBodyLine("return criteria;"); //$NON-NLS-1$
        answer.addMethod(method);

        // now add the methods for simplifying the individual field set methods
        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("addCriterion"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addBodyLine("if (condition == null) {"); //$NON-NLS-1$
        method.addBodyLine("throw new RuntimeException(\"Value for condition cannot be null\");"); //$NON-NLS-1$
        method.addBodyLine("}"); //$NON-NLS-1$
        //add
        method.addBodyLine(sqlsb.toString()+"(condition);");
        method.addBodyLine("criteria.add(new Criterion(condition));"); //$NON-NLS-1$
        if (criteriaLists.size() > 1) {
            method.addBodyLine("allCriteria = null;"); //$NON-NLS-1$
        }
        answer.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("addCriterion"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "property")); //$NON-NLS-1$
        method.addBodyLine("if (value == null) {"); //$NON-NLS-1$
        method.addBodyLine("throw new RuntimeException(\"Value for \" + property + \" cannot be null\");"); //$NON-NLS-1$
        method.addBodyLine("}"); //$NON-NLS-1$
        //add
        method.addBodyLine(sqlsb.toString()+"(condition);");
        method.addBodyLine("criteria.add(new Criterion(condition, value));"); //$NON-NLS-1$
        if (criteriaLists.size() > 1) {
            method.addBodyLine("allCriteria = null;"); //$NON-NLS-1$
        }
        answer.addMethod(method);

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName("addCriterion"); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value1")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value2")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "property")); //$NON-NLS-1$
        method.addBodyLine("if (value1 == null || value2 == null) {"); //$NON-NLS-1$
        method.addBodyLine("throw new RuntimeException(\"Between values for \" + property + \" cannot be null\");"); //$NON-NLS-1$
        method.addBodyLine("}"); //$NON-NLS-1$
        //add
        method.addBodyLine(sqlsb.toString()+"(condition);");
        method.addBodyLine("criteria.add(new Criterion(condition, value1, value2));"); //$NON-NLS-1$
        if (criteriaLists.size() > 1) {
            method.addBodyLine("allCriteria = null;"); //$NON-NLS-1$
        }
        answer.addMethod(method);

        FullyQualifiedJavaType listOfDates = new FullyQualifiedJavaType(
                "java.util.List<java.util.Date>"); //$NON-NLS-1$

        if (introspectedTable.hasJDBCDateColumns()) {
            topLevelClass.addImportedType(FullyQualifiedJavaType
                    .getDateInstance());
            topLevelClass.addImportedType(FullyQualifiedJavaType
                    .getNewIteratorInstance());
            method = new Method();
            method.setVisibility(JavaVisibility.PROTECTED);
            method.setName("addCriterionForJDBCDate"); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "condition")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getDateInstance(), "value")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "property")); //$NON-NLS-1$
            method.addBodyLine("if (value == null) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Value for \" + property + \" cannot be null\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method
                    .addBodyLine("addCriterion(condition, new java.sql.Date(value.getTime()), property);"); //$NON-NLS-1$
            answer.addMethod(method);

            method = new Method();
            method.setVisibility(JavaVisibility.PROTECTED);
            method.setName("addCriterionForJDBCDate"); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "condition")); //$NON-NLS-1$
            method.addParameter(new Parameter(listOfDates, "values")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "property")); //$NON-NLS-1$
            method.addBodyLine("if (values == null || values.size() == 0) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Value list for \" + property + \" cannot be null or empty\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method
                    .addBodyLine("List<java.sql.Date> dateList = new ArrayList<java.sql.Date>();"); //$NON-NLS-1$
            method.addBodyLine("Iterator<Date> iter = values.iterator();"); //$NON-NLS-1$
            method.addBodyLine("while (iter.hasNext()) {"); //$NON-NLS-1$
            method
                    .addBodyLine("dateList.add(new java.sql.Date(iter.next().getTime()));"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method.addBodyLine("addCriterion(condition, dateList, property);"); //$NON-NLS-1$
            answer.addMethod(method);

            method = new Method();
            method.setVisibility(JavaVisibility.PROTECTED);
            method.setName("addCriterionForJDBCDate"); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "condition")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getDateInstance(), "value1")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getDateInstance(), "value2")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "property")); //$NON-NLS-1$
            method.addBodyLine("if (value1 == null || value2 == null) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Between values for \" + property + \" cannot be null\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method
                    .addBodyLine("addCriterion(condition, new java.sql.Date(value1.getTime()), new java.sql.Date(value2.getTime()), property);"); //$NON-NLS-1$
            answer.addMethod(method);
        }

        if (introspectedTable.hasJDBCTimeColumns()) {
            topLevelClass.addImportedType(FullyQualifiedJavaType
                    .getDateInstance());
            topLevelClass.addImportedType(FullyQualifiedJavaType
                    .getNewIteratorInstance());
            method = new Method();
            method.setVisibility(JavaVisibility.PROTECTED);
            method.setName("addCriterionForJDBCTime"); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "condition")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getDateInstance(), "value")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "property")); //$NON-NLS-1$
            method.addBodyLine("if (value == null) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Value for \" + property + \" cannot be null\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method
                    .addBodyLine("addCriterion(condition, new java.sql.Time(value.getTime()), property);"); //$NON-NLS-1$
            answer.addMethod(method);

            method = new Method();
            method.setVisibility(JavaVisibility.PROTECTED);
            method.setName("addCriterionForJDBCTime"); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "condition")); //$NON-NLS-1$
            method.addParameter(new Parameter(listOfDates, "values")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "property")); //$NON-NLS-1$
            method.addBodyLine("if (values == null || values.size() == 0) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Value list for \" + property + \" cannot be null or empty\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method
                    .addBodyLine("List<java.sql.Time> timeList = new ArrayList<java.sql.Time>();"); //$NON-NLS-1$
            method.addBodyLine("Iterator<Date> iter = values.iterator();"); //$NON-NLS-1$
            method.addBodyLine("while (iter.hasNext()) {"); //$NON-NLS-1$
            method
                    .addBodyLine("timeList.add(new java.sql.Time(iter.next().getTime()));"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method.addBodyLine("addCriterion(condition, timeList, property);"); //$NON-NLS-1$
            answer.addMethod(method);

            method = new Method();
            method.setVisibility(JavaVisibility.PROTECTED);
            method.setName("addCriterionForJDBCTime"); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "condition")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getDateInstance(), "value1")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getDateInstance(), "value2")); //$NON-NLS-1$
            method.addParameter(new Parameter(FullyQualifiedJavaType
                    .getStringInstance(), "property")); //$NON-NLS-1$
            method.addBodyLine("if (value1 == null || value2 == null) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Between values for \" + property + \" cannot be null\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
            method
                    .addBodyLine("addCriterion(condition, new java.sql.Time(value1.getTime()), new java.sql.Time(value2.getTime()), property);"); //$NON-NLS-1$
            answer.addMethod(method);
        }

        for (IntrospectedColumn introspectedColumn : introspectedTable
                .getNonBLOBColumns()) {
            topLevelClass.addImportedType(introspectedColumn
                    .getFullyQualifiedJavaType());

            // here we need to add the individual methods for setting the
            // conditions for a field
            answer.addMethod(getSetNullMethod(introspectedColumn));
            answer.addMethod(getSetNotNullMethod(introspectedColumn));
            answer.addMethod(getSetEqualMethod(introspectedColumn));
            answer.addMethod(getSetNotEqualMethod(introspectedColumn));
            answer.addMethod(getSetGreaterThanMethod(introspectedColumn));
            answer
                    .addMethod(getSetGreaterThenOrEqualMethod(introspectedColumn));
            answer.addMethod(getSetLessThanMethod(introspectedColumn));
            answer.addMethod(getSetLessThanOrEqualMethod(introspectedColumn));

            if (introspectedColumn.isJdbcCharacterColumn()) {
                answer.addMethod(getSetLikeMethod(introspectedColumn));
                answer.addMethod(getSetNotLikeMethod(introspectedColumn));
            }

            answer.addMethod(getSetInOrNotInMethod(introspectedColumn, true));
            answer.addMethod(getSetInOrNotInMethod(introspectedColumn, false));
            answer.addMethod(getSetBetweenOrNotBetweenMethod(
                    introspectedColumn, true));
            answer.addMethod(getSetBetweenOrNotBetweenMethod(
                    introspectedColumn, false));
        }

        return answer;
    }

    private Method getSetNullMethod(IntrospectedColumn introspectedColumn) {
        return getNoValueMethod(introspectedColumn, "IsNull", "is null"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetNotNullMethod(IntrospectedColumn introspectedColumn) {
        return getNoValueMethod(introspectedColumn, "IsNotNull", "is not null"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetEqualMethod(IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn, "EqualTo", "="); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetNotEqualMethod(IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn, "NotEqualTo", "<>"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetGreaterThanMethod(IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn, "GreaterThan", ">"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetGreaterThenOrEqualMethod(
            IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn,
                "GreaterThanOrEqualTo", ">="); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetLessThanMethod(IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn, "LessThan", "<"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetLessThanOrEqualMethod(
            IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn,
                "LessThanOrEqualTo", "<="); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetLikeMethod(IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn, "Like", "like"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSetNotLikeMethod(IntrospectedColumn introspectedColumn) {
        return getSingleValueMethod(introspectedColumn, "NotLike", "not like"); //$NON-NLS-1$ //$NON-NLS-2$
    }

    private Method getSingleValueMethod(IntrospectedColumn introspectedColumn,
                                        String nameFragment, String operator) {
        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.addParameter(new Parameter(introspectedColumn
                .getFullyQualifiedJavaType(), "value")); //$NON-NLS-1$
        StringBuilder sb = new StringBuilder();
        sb.append(introspectedColumn.getJavaProperty());
        sb.setCharAt(0, Character.toUpperCase(sb.charAt(0)));
        sb.insert(0, "and"); //$NON-NLS-1$
        sb.append(nameFragment);
        method.setName(sb.toString());
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        sb.setLength(0);

        if (introspectedColumn.isJDBCDateColumn()) {
            sb.append("addCriterionForJDBCDate(\""); //$NON-NLS-1$
        } else if (introspectedColumn.isJDBCTimeColumn()) {
            sb.append("addCriterionForJDBCTime(\""); //$NON-NLS-1$
        } else if (stringHasValue(introspectedColumn
                .getTypeHandler())) {
            sb.append("add"); //$NON-NLS-1$
            sb.append(introspectedColumn.getJavaProperty());
            sb.setCharAt(3, Character.toUpperCase(sb.charAt(3)));
            sb.append("Criterion(\""); //$NON-NLS-1$
        } else {
            sb.append("addCriterion(\""); //$NON-NLS-1$
        }

        sb.append(MyBatis3FormattingUtilities
                .getAliasedActualColumnName(introspectedColumn));
        sb.append(' ');
        sb.append(operator);
        sb.append("\", "); //$NON-NLS-1$
        sb.append("value"); //$NON-NLS-1$
        sb.append(", \""); //$NON-NLS-1$
        sb.append(introspectedColumn.getJavaProperty());
        sb.append("\");"); //$NON-NLS-1$
        method.addBodyLine(sb.toString());
        method.addBodyLine("return (Criteria) this;"); //$NON-NLS-1$

        return method;
    }

    /**
     * Generates methods that set between and not between conditions
     *
     * @param introspectedColumn
     * @param betweenMethod
     * @return a generated method for the between or not between method
     */
    private Method getSetBetweenOrNotBetweenMethod(
            IntrospectedColumn introspectedColumn, boolean betweenMethod) {
        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        FullyQualifiedJavaType type = introspectedColumn
                .getFullyQualifiedJavaType();

        method.addParameter(new Parameter(type, "value1")); //$NON-NLS-1$
        method.addParameter(new Parameter(type, "value2")); //$NON-NLS-1$
        StringBuilder sb = new StringBuilder();
        sb.append(introspectedColumn.getJavaProperty());
        sb.setCharAt(0, Character.toUpperCase(sb.charAt(0)));
        sb.insert(0, "and"); //$NON-NLS-1$
        if (betweenMethod) {
            sb.append("Between"); //$NON-NLS-1$
        } else {
            sb.append("NotBetween"); //$NON-NLS-1$
        }
        method.setName(sb.toString());
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        sb.setLength(0);

        if (introspectedColumn.isJDBCDateColumn()) {
            sb.append("addCriterionForJDBCDate(\""); //$NON-NLS-1$
        } else if (introspectedColumn.isJDBCTimeColumn()) {
            sb.append("addCriterionForJDBCTime(\""); //$NON-NLS-1$
        } else if (stringHasValue(introspectedColumn
                .getTypeHandler())) {
            sb.append("add"); //$NON-NLS-1$
            sb.append(introspectedColumn.getJavaProperty());
            sb.setCharAt(3, Character.toUpperCase(sb.charAt(3)));
            sb.append("Criterion(\""); //$NON-NLS-1$
        } else {
            sb.append("addCriterion(\""); //$NON-NLS-1$
        }

        sb.append(MyBatis3FormattingUtilities
                .getAliasedActualColumnName(introspectedColumn));
        if (betweenMethod) {
            sb.append(" between"); //$NON-NLS-1$
        } else {
            sb.append(" not between"); //$NON-NLS-1$
        }
        sb.append("\", "); //$NON-NLS-1$
        sb.append("value1, value2"); //$NON-NLS-1$
        sb.append(", \""); //$NON-NLS-1$
        sb.append(introspectedColumn.getJavaProperty());
        sb.append("\");"); //$NON-NLS-1$
        method.addBodyLine(sb.toString());
        method.addBodyLine("return (Criteria) this;"); //$NON-NLS-1$

        return method;
    }

    /**
     *
     * @param introspectedColumn
     * @param inMethod
     *            if true generates an "in" method, else generates a "not in"
     *            method
     * @return a generated method for the in or not in method
     */
    private Method getSetInOrNotInMethod(IntrospectedColumn introspectedColumn,
                                         boolean inMethod) {
        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        FullyQualifiedJavaType type = FullyQualifiedJavaType
                .getNewListInstance();
        if (introspectedColumn.getFullyQualifiedJavaType().isPrimitive()) {
            type.addTypeArgument(introspectedColumn.getFullyQualifiedJavaType()
                    .getPrimitiveTypeWrapper());
        } else {
            type
                    .addTypeArgument(introspectedColumn
                            .getFullyQualifiedJavaType());
        }

        method.addParameter(new Parameter(type, "values")); //$NON-NLS-1$
        StringBuilder sb = new StringBuilder();
        sb.append(introspectedColumn.getJavaProperty());
        sb.setCharAt(0, Character.toUpperCase(sb.charAt(0)));
        sb.insert(0, "and"); //$NON-NLS-1$
        if (inMethod) {
            sb.append("In"); //$NON-NLS-1$
        } else {
            sb.append("NotIn"); //$NON-NLS-1$
        }
        method.setName(sb.toString());
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        sb.setLength(0);

        if (introspectedColumn.isJDBCDateColumn()) {
            sb.append("addCriterionForJDBCDate(\""); //$NON-NLS-1$
        } else if (introspectedColumn.isJDBCTimeColumn()) {
            sb.append("addCriterionForJDBCTime(\""); //$NON-NLS-1$
        } else if (stringHasValue(introspectedColumn
                .getTypeHandler())) {
            sb.append("add"); //$NON-NLS-1$
            sb.append(introspectedColumn.getJavaProperty());
            sb.setCharAt(3, Character.toUpperCase(sb.charAt(3)));
            sb.append("Criterion(\""); //$NON-NLS-1$
        } else {
            sb.append("addCriterion(\""); //$NON-NLS-1$
        }

        sb.append(MyBatis3FormattingUtilities
                .getAliasedActualColumnName(introspectedColumn));
        if (inMethod) {
            sb.append(" in"); //$NON-NLS-1$
        } else {
            sb.append(" not in"); //$NON-NLS-1$
        }
        sb.append("\", values, \""); //$NON-NLS-1$
        sb.append(introspectedColumn.getJavaProperty());
        sb.append("\");"); //$NON-NLS-1$
        method.addBodyLine(sb.toString());
        method.addBodyLine("return (Criteria) this;"); //$NON-NLS-1$

        return method;
    }

    private Method getNoValueMethod(IntrospectedColumn introspectedColumn,
                                    String nameFragment, String operator) {
        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        StringBuilder sb = new StringBuilder();
        sb.append(introspectedColumn.getJavaProperty());
        sb.setCharAt(0, Character.toUpperCase(sb.charAt(0)));
        sb.insert(0, "and"); //$NON-NLS-1$
        sb.append(nameFragment);
        method.setName(sb.toString());
        method.setReturnType(FullyQualifiedJavaType.getCriteriaInstance());
        sb.setLength(0);
        sb.append("addCriterion(\""); //$NON-NLS-1$
        sb.append(MyBatis3FormattingUtilities
                .getAliasedActualColumnName(introspectedColumn));
        sb.append(' ');
        sb.append(operator);
        sb.append("\");"); //$NON-NLS-1$
        method.addBodyLine(sb.toString());
        method.addBodyLine("return (Criteria) this;"); //$NON-NLS-1$

        return method;
    }

    /**
     * This method adds all the extra methods and fields required to support a
     * user defined type handler on some column.
     *
     * @param introspectedColumn
     * @param constructor
     * @param innerClass
     * @return the name of the List added to the class by this method
     */
    private String addtypeHandledObjectsAndMethods(
            IntrospectedColumn introspectedColumn, Method constructor,
            InnerClass innerClass) {
        String answer;
        StringBuilder sb = new StringBuilder();

        // add new private field and public accessor in the class
        sb.setLength(0);
        sb.append(introspectedColumn.getJavaProperty());
        sb.append("Criteria"); //$NON-NLS-1$
        answer = sb.toString();

        Field field = new Field();
        field.setVisibility(JavaVisibility.PROTECTED);
        field.setType(new FullyQualifiedJavaType("java.util.List<Criterion>")); //$NON-NLS-1$
        field.setName(answer);
        innerClass.addField(field);

        Method method = new Method();
        method.setVisibility(JavaVisibility.PUBLIC);
        method.setReturnType(field.getType());
        method.setName(getGetterMethodName(field.getName(), field
                .getType()));
        sb.insert(0, "return "); //$NON-NLS-1$
        sb.append(';');
        method.addBodyLine(sb.toString());
        innerClass.addMethod(method);

        // add constructor initialization
        sb.setLength(0);
        sb.append(field.getName());
        sb.append(" = new ArrayList<Criterion>();"); //$NON-NLS-1$;
        constructor.addBodyLine(sb.toString());

        // now add the methods for simplifying the individual field set methods
        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        sb.setLength(0);
        sb.append("add"); //$NON-NLS-1$
        sb.append(introspectedColumn.getJavaProperty());
        sb.setCharAt(3, Character.toUpperCase(sb.charAt(3)));
        sb.append("Criterion"); //$NON-NLS-1$

        method.setName(sb.toString());
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getObjectInstance(), "value")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "property")); //$NON-NLS-1$
        method.addBodyLine("if (value == null) {"); //$NON-NLS-1$
        method
                .addBodyLine("throw new RuntimeException(\"Value for \" + property + \" cannot be null\");"); //$NON-NLS-1$
        method.addBodyLine("}"); //$NON-NLS-1$

        method.addBodyLine(
                String.format("%s.add(new Criterion(condition, value, \"%s\"));", //$NON-NLS-1$
                        field.getName(), introspectedColumn.getTypeHandler()));
        method.addBodyLine("allCriteria = null;"); //$NON-NLS-1$
        innerClass.addMethod(method);

        sb.setLength(0);
        sb.append("add"); //$NON-NLS-1$
        sb.append(introspectedColumn.getJavaProperty());
        sb.setCharAt(3, Character.toUpperCase(sb.charAt(3)));
        sb.append("Criterion"); //$NON-NLS-1$

        method = new Method();
        method.setVisibility(JavaVisibility.PROTECTED);
        method.setName(sb.toString());
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "condition")); //$NON-NLS-1$
        method.addParameter(new Parameter(introspectedColumn
                .getFullyQualifiedJavaType(), "value1")); //$NON-NLS-1$
        method.addParameter(new Parameter(introspectedColumn
                .getFullyQualifiedJavaType(), "value2")); //$NON-NLS-1$
        method.addParameter(new Parameter(FullyQualifiedJavaType
                .getStringInstance(), "property")); //$NON-NLS-1$
        if (!introspectedColumn.getFullyQualifiedJavaType().isPrimitive()) {
            method.addBodyLine("if (value1 == null || value2 == null) {"); //$NON-NLS-1$
            method
                    .addBodyLine("throw new RuntimeException(\"Between values for \" + property + \" cannot be null\");"); //$NON-NLS-1$
            method.addBodyLine("}"); //$NON-NLS-1$
        }

        method.addBodyLine(
                String.format("%s.add(new Criterion(condition, value1, value2, \"%s\"));", //$NON-NLS-1$
                        field.getName(), introspectedColumn.getTypeHandler()));

        method.addBodyLine("allCriteria = null;"); //$NON-NLS-1$
        innerClass.addMethod(method);

        return answer;
    }
}
```
</details>
