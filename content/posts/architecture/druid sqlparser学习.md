---
title: "druid sqlparser学习"
date: "2024-12-10"
tags: ["架构"]
ShowToc: false
TocOpen: false
---


SQLExprTableSource 单表查询  
SQLJoinTableSource 表示join,from条件后面的表如果是join的情况,JoinType是join的类型  
SQLSubqueryTableSource 子查询  
SQLSelectQueryBlock(单表Sql查询)  
SQLUnionQuery(union表查询)  
SQLAllColumnExpr 表示 比如select * from a  
SQLSelectItem 查询的具体的字段 select a.x, a.*, a.b, b from a, 这里a.x, a.*,a.b, b都是  

select a.*, *, b.x as y from table_1 a, table_2 b where a.x = 1 and b.x = 2  
一个查询语句SQLSelectStatement，主要看其SQLSelect类型的成员select  
SQLSelect 成员query是去掉了子查询 orderby条件的sql片段 query是MySqlSelectQueryBlock类型（单表查询 union表查询是SQLUnionQuery）代码位于  
com.alibaba.druid.sql.ast.statement.SQLSelect#accept0  
MySqlSelectQueryBlock 成员selectList是查询的字段,以SQLSelectItem表示,例子中就是a.*,*, b.x as y这三个,除了selectList,还有from,例子中是SQLJoinTableSource,因为是join操作  
SQLSelectItem 例子中就是a.*, 这里 expr是SQLPropertyExpr,alias为空。如果查询字段有as的情况，这里别名就是as的值。  
SQLPropertyExpr的owner就是SQLIdentifierExpr， a.*这里SQLIdentifierExpr = a, SQLPropertyExpr的owner就是SQLIdentifierExpr。  

mysql转dm语句
```
public class Mysql2DMMysqlVisitor extends MySqlASTVisitorAdapter {
    private static boolean quoteSymbol;
    private static int upperLowerCase;

    private final static String LOG_PREFIX = "-----达梦语句适配-----, ";
    private final static String DM_ESCAPE = "\"";
    private final static String MYSQL_ESCAPE = "`";

    public void setQuoteSymbol(boolean quoteSymbol) {
        Mysql2DMMysqlVisitor.quoteSymbol = quoteSymbol;
    }

    public void setUpperLowerCase(int upperLowerCase) {
        Mysql2DMMysqlVisitor.upperLowerCase = upperLowerCase;
    }

    private static String replaceName(String str) {
        if (str != null && !("".equals(str))) {
            if (quoteSymbol) {
                if (upperLowerCase == 1) {
                    str = str.toUpperCase();
                } else if (upperLowerCase == 2) {
                    str = str.toLowerCase();
                }
                if (str.contains(MYSQL_ESCAPE)) {
                    return str.replace(MYSQL_ESCAPE, DM_ESCAPE);
                } else {
                    return DM_ESCAPE + str + DM_ESCAPE;
                }
            } else {
                if (str.contains(MYSQL_ESCAPE)) {
                    return str.replace(MYSQL_ESCAPE, DM_ESCAPE);
                } else {
                    return str;
                }
            }
        }
        return str;
    }

    /**
     * 达梦不支持日期格式函数参数为 %H:00形式, 需要加上", 改成%H":00"
     *
     * @param content
     * @return
     */
    private static String resolveForDateFormat(String content) {
        boolean flag = false;
        StringBuilder sb = new StringBuilder();
        for (int i = 0; i < content.length(); i++) {
            char thisChar = content.charAt(i);
            if (thisChar == '%') {
                if (i == 0) {
                    sb.append(thisChar);
                } else {
                    sb.append("\"").append(thisChar);
                }
                flag = true;
            } else {
                if (flag) {
                    if (i == content.length() - 1) {
                        sb.append(thisChar);
                    } else {
                        sb.append(thisChar).append("\"");
                    }
                } else {
                    if (i == content.length() - 1) {
                        sb.append(thisChar).append("\"");
                    } else {
                        sb.append(thisChar);
                    }

                }
                flag = false;
            }
        }
        return sb.toString();
    }

    /**
     * select列表项
     * 处理别名
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLSelectItem x) {
        x.getExpr().accept(this);
        x.setAlias(replaceName(x.getAlias()));
        return false;
    }

    /**
     * 属性, tab1.col1、schema1.tab1
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLPropertyExpr x) {
        x.getOwner().accept(this);
        x.setName(replaceName(x.getName()));
        return false;
    }

    /**
     * 替换属性符号, 数据库名、表名、列名等
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLIdentifierExpr x) {
        x.setName(replaceName(x.getName()));
        return false;
    }

    /**
     * 表项
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLExprTableSource x) {
        x.getExpr().accept(this);
        x.setAlias(replaceName(x.getAlias()));
        return false;
    }

    /**
     * 表项
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLSubqueryTableSource x) {
        x.getSelect().accept(this);
        x.setAlias(replaceName(x.getAlias()));
        return false;
    }

    /**
     * 表项
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLUnionQueryTableSource x) {
        x.getUnion().accept(this);
        x.setAlias(replaceName(x.getAlias()));
        return false;
    }

    /**
     * 表项
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLJoinTableSource x) {
        x.setAlias(replaceName(x.getAlias()));
        return true;
    }

    /**
     * 表达式处理
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLAggregateExpr x) {
        String methodName = x.getMethodName();
        String groupConcatMysqlName = "GROUP_CONCAT";
        String groupConcatDmName = "LISTAGG";
        String anyValueMysqlName = "ANY_VALUE";
        String anyValueDmName = "FIRST_VALUE";
        if (groupConcatMysqlName.equalsIgnoreCase(methodName)) {
            x.setMethodName(groupConcatDmName);
            if (x.getArguments().size() > 1) {
                SQLExpr left = x.getArguments().get(0);
                for (int i = 1; i < x.getArguments().size(); i++) {
                    SQLBinaryOpExpr binaryOpExpr = new SQLBinaryOpExpr();
                    binaryOpExpr.setLeft(left);
                    binaryOpExpr.setRight(x.getArguments().get(i));
                    binaryOpExpr.setOperator(SQLBinaryOperator.Concat);
                    left = binaryOpExpr;
                }
                left.setParent(x);
                x.getArguments().clear();
                x.addArgument(left);
            }
            Object separator = x.getAttributes().get("SEPARATOR");
            if (separator instanceof SQLExpr) {
                x.addArgument((SQLExpr) separator);
            } else {
                x.addArgument(new SQLCharExpr(","));
            }
            if (x.getOrderBy() != null) {
                x.setWithinGroup(true);
            }
            if (x.getAttributes() != null && x.getAttributes().size() != 0) {
                x.getAttributes().clear();
            }
        } else if (anyValueMysqlName.equalsIgnoreCase(methodName)) {
            x.setMethodName(anyValueDmName);
        }
        return true;
    }

    /**
     * 方法执行处理
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLMethodInvokeExpr x) {
        // 将IF转case when
        String methodName = x.getMethodName();
        methodName = methodName.toLowerCase();
        SQLObject parent = x.getParent();
        List<SQLExpr> arguments = x.getArguments();
        switch (methodName) {
            case "if":
                if (parent instanceof SQLReplaceable) {
                    SQLReplaceable sqlReplaceable = (SQLReplaceable) parent;
                    SQLExpr sqlExpr0 = arguments.get(0);
                    SQLExpr sqlExpr1 = arguments.get(1);
                    SQLExpr sqlExpr2 = arguments.get(2);
                    SQLCaseExpr sqlCaseExpr = new SQLCaseExpr();
                    sqlCaseExpr.addItem(new SQLCaseExpr.Item(sqlExpr0, sqlExpr1));
                    sqlCaseExpr.setElseExpr(sqlExpr2);
                    sqlReplaceable.replace(x, sqlCaseExpr);
                    sqlExpr0.accept(this);
                    sqlExpr1.accept(this);
                    sqlExpr2.accept(this);
                    return false;
                }
                break;
            case "convert":
                if (parent instanceof SQLReplaceable) {
                    SQLReplaceable sqlReplaceable = (SQLReplaceable) parent;
                    SQLExpr sqlExpr0 = arguments.get(0);
                    SQLExpr sqlExpr1 = arguments.get(1);
                    if (sqlExpr1 instanceof SQLDataTypeRefExpr) {
                        SQLCastExpr sqlCastExpr = new SQLCastExpr(sqlExpr0, ((SQLDataTypeRefExpr) sqlExpr1).getDataType());
                        sqlReplaceable.replace(x, sqlCastExpr);
                        sqlExpr0.accept(this);
                        sqlExpr1.accept(this);
                        return false;
                    }
                }
                break;
            case "date_format":
                SQLExpr sqlExpr = arguments.get(1);
                SQLCharExpr sqlCharExpr = (SQLCharExpr) sqlExpr;
                sqlCharExpr.setText(resolveForDateFormat(sqlCharExpr.getText()));
                return true;
            case "json_unquote":
                SQLExpr arg0 = arguments.get(0);
                x.setArgument(0, new SQLCharExpr("\""));
                x.setMethodName("TRIM");
                x.setFrom(arg0);
                arg0.accept(this);
                return false;
            case "st_contains":
                x.setMethodName("dmgeo.st_contains");
                return true;
            case "st_distance_sphere":
                x.setMethodName("dmgeo.ST_Distance");
                return true;
            case "uuid":
                x.setMethodName("sys_guid");
                return true;
            case "point":
                if (parent instanceof SQLReplaceable) {
                    SQLMethodInvokeExpr sqlMethodInvokeExpr = new SQLMethodInvokeExpr("dmgeo.ST_GeomFromText");
                    SQLMethodInvokeExpr concatMethod = new SQLMethodInvokeExpr("CONCAT");
                    concatMethod.addArgument(new SQLCharExpr("POINT("));
                    concatMethod.addArgument(arguments.get(0));
                    concatMethod.addArgument(new SQLCharExpr(String.valueOf(' ')));
                    concatMethod.addArgument(arguments.get(1));
                    concatMethod.addArgument(new SQLCharExpr(")"));
                    sqlMethodInvokeExpr.addArgument(concatMethod);
                    sqlMethodInvokeExpr.addArgument(new SQLNumberExpr(0));
                    ((SQLReplaceable) parent).replace(x, sqlMethodInvokeExpr);
                    for (SQLExpr argument : arguments) {
                        argument.accept(this);
                    }
                    return false;
                }
                break;
            case "geometryfromtext":
                if (parent instanceof SQLReplaceable) {
                    SQLMethodInvokeExpr sqlMethodInvokeExpr = new SQLMethodInvokeExpr("dmgeo.ST_GeomFromText");
                    sqlMethodInvokeExpr.addArgument(arguments.get(0));
                    sqlMethodInvokeExpr.addArgument(new SQLNumberExpr(0));
                    ((SQLReplaceable) parent).replace(x, sqlMethodInvokeExpr);
                    for (SQLExpr argument : sqlMethodInvokeExpr.getArguments()) {
                        argument.accept(this);
                    }
                    return false;
                }
                break;
            default:
                break;
        }
        return true;
    }

    /**
     * 二元运算表达式处理
     *
     * @param x
     * @return
     */
    @Override
    public boolean visit(SQLBinaryOpExpr x) {
        if (x.getLeft() instanceof SQLBooleanExpr) {
            boolean booleanValue = ((SQLBooleanExpr) x.getLeft()).getBooleanValue();
            x.setLeft(new SQLNumberExpr(booleanValue ? 1 : 0));
        }
        if (x.getRight() instanceof SQLBooleanExpr) {
            boolean booleanValue = ((SQLBooleanExpr) x.getRight()).getBooleanValue();
            x.setRight(new SQLNumberExpr(booleanValue ? 1 : 0));
        }

        SQLObject parent = x.getParent();
        if (x.getOperator() == SQLBinaryOperator.SubGt) {
            if (parent instanceof SQLReplaceable) {
                SQLMethodInvokeExpr sqlMethodInvokeExpr = new SQLMethodInvokeExpr("JSON_EXTRACT");
                SQLExpr left = x.getLeft();
                SQLExpr right = x.getRight();
                sqlMethodInvokeExpr.addArgument(left);
                sqlMethodInvokeExpr.addArgument(right);
                ((SQLReplaceable) parent).replace(x, sqlMethodInvokeExpr);
                left.accept(this);
                right.accept(this);
                return false;
            }
        } else if (x.getOperator() == SQLBinaryOperator.SubGtGt) {
            if (parent instanceof SQLReplaceable) {
                SQLMethodInvokeExpr sqlMethodInvokeExpr = new SQLMethodInvokeExpr("JSON_VALUE");
                SQLExpr left = x.getLeft();
                SQLExpr right = x.getRight();
                sqlMethodInvokeExpr.addArgument(left);
                sqlMethodInvokeExpr.addArgument(right);
                ((SQLReplaceable) parent).replace(x, sqlMethodInvokeExpr);
                left.accept(this);
                right.accept(this);
                return false;
            }
        } else if (x.getOperator() == SQLBinaryOperator.RegExp) {
            SQLMethodInvokeExpr sqlMethodInvokeExpr = new SQLMethodInvokeExpr("REGEXP_LIKE");
            SQLExpr left = x.getLeft();
            SQLExpr right = x.getRight();
            sqlMethodInvokeExpr.addArgument(left);
            sqlMethodInvokeExpr.addArgument(right);
            left.accept(this);
            right.accept(this);
            sqlMethodInvokeExpr.setParent(parent);
            if (parent instanceof SQLBinaryOpExpr) {
                if (((SQLBinaryOpExpr) parent).getLeft() == x) {
                    ((SQLBinaryOpExpr) parent).setLeft(sqlMethodInvokeExpr);
                } else if (((SQLBinaryOpExpr) parent).getRight() == x) {
                    ((SQLBinaryOpExpr) parent).setRight(sqlMethodInvokeExpr);
                }
            } else if (parent instanceof SQLSelectQueryBlock) {
                if (((SQLSelectQueryBlock) parent).getWhere() == x) {
                    ((SQLSelectQueryBlock) parent).setWhere(sqlMethodInvokeExpr);
                }
            }
            return false;
        }

        return true;
    }

    @Override
    public boolean visit(SQLUpdateSetItem x) {
        if (x.getValue() instanceof SQLBooleanExpr) {
            boolean booleanValue = ((SQLBooleanExpr) x.getValue()).getBooleanValue();
            x.setValue(new SQLNumberExpr(booleanValue ? 1 : 0));
        }

        return true;
    }

    @Override
    public boolean visit(SQLInsertStatement.ValuesClause x) {
        for(int i = 0; i < x.getValues().size(); ++i) {
            if (x.getValues().get(i) instanceof SQLBooleanExpr) {
                boolean booleanValue = ((SQLBooleanExpr) x.getValues().get(i)).getBooleanValue();
                x.getValues().set(i, new SQLNumberExpr(booleanValue ? 1 : 0));
            }
        }

        return true;
    }

    @Override
    public boolean visit(SQLAlterTableAddColumn x) {
        // 兼容Mysql有afterColumn
        x.getColumns().forEach(col -> col.accept(this));
        if (x.getAfterColumn() != null) {
            x.setAfterColumn(null);
        }
        return false;
    }

    @Override
    public boolean visit(MySqlAlterTableModifyColumn x) {
        x.getNewColumnDefinition().setParent(x);
        // 兼容Mysql有afterColumn
        x.setAfterColumn(null);
        return true;
    }

    @Override
    public boolean visit(MySqlAlterTableChangeColumn x) {
        x.getNewColumnDefinition().setParent(x);
        // 兼容Mysql有afterColumn
        x.setAfterColumn(null);
        return true;
    }

    @Override
    public boolean visit(SQLColumnDefinition x) {
        SQLDataType dataType = x.getDataType();
        if (dataType != null) {
            if (dataType instanceof SQLDataTypeImpl) {
                ((SQLDataTypeImpl) dataType).setUnsigned(false);
            }
            String name = dataType.getName();
            name = name.toLowerCase();
            switch (name) {
                case "bit":
                case "int":
                case "bigint":
                case "smallint":
                case "double":
                case "tinyint":
                case "integer":
                    if (dataType.getArguments() != null && dataType.getArguments().size() != 0) {
                        dataType.getArguments().clear();
                    }
                    break;
                case "json":
                    SQLCharacterDataType charType = new SQLCharacterDataType("VARCHAR");
                    SQLIntegerExpr arg = new SQLIntegerExpr(32767);
                    arg.setParent(charType);
                    charType.addArgument(arg);
                    charType.setCollate("utf8_bin");
                    charType.setParent(x);
                    x.setDataType(charType);
                    break;
                case "tinytext":
                case "mediumtext":
                case "longtext":
                case "text":
                    if (x.getParent() instanceof MySqlAlterTableModifyColumn ||
                            x.getParent() instanceof MySqlAlterTableChangeColumn) {
                        SQLCharacterDataType textType = new SQLCharacterDataType("VARCHAR");
                        SQLIntegerExpr textArg = new SQLIntegerExpr(32767);
                        textArg.setParent(textType);
                        textType.addArgument(textArg);
                        textType.setCollate("utf8_bin");
                        textType.setParent(x);
                        x.setDataType(textType);
                    } else {
                        dataType.setName("TEXT");
                    }
            }

        }

        if (x.getOnUpdate() != null && x.getOnUpdate() instanceof SQLCurrentTimeExpr) {
//            x.setOnUpdate(new SQLCurrentTimeExpr(SQLCurrentTimeExpr.Type.LOCALTIMESTAMP));
            x.setOnUpdate(null);
        }
        if (x.getComment() != null) {
            x.setComment((SQLExpr) null);
        }
        return true;
    }

    @Override
    public boolean visit(SQLBinaryExpr x) {
        SQLObject parent = x.getParent();
        if (parent instanceof SQLReplaceable) {
            ((SQLReplaceable) parent).replace(x, new SQLCharExpr(x.getText()));
        }
        return true;
    }

    @Override
    public boolean visit(SQLIntervalExpr x) {
        SQLExpr valueSqlExpr = x.getValue();
        if (valueSqlExpr instanceof SQLValuableExpr) {
            Object value = ((SQLValuableExpr) valueSqlExpr).getValue();
            x.setValue(new SQLCharExpr(Objects.toString(value)));
            return false;
        }
        return true;
    }

    @Override
    public boolean visit(SQLCreateIndexStatement x) {
        this.visit(x.getIndexDefinition());
        String tableName = x.getTableName();
        SQLIndexDefinition indexDefinition = x.getIndexDefinition();
        this.modifyIndexName(tableName, indexDefinition);
        return true;
    }

    private void modifyIndexName(String tableName, SQLIndexDefinition indexDefinition) {
        if (tableName != null && indexDefinition != null && indexDefinition.getName() instanceof SQLIdentifierExpr) {
            if ((tableName.charAt(0) == '"' && tableName.charAt(tableName.length() - 1) == '"') ||
                    (tableName.charAt(0) == '`' && tableName.charAt(tableName.length() - 1) == '`')) {
                tableName = tableName.substring(1, tableName.length() - 1);
            }
            SQLName sqlName = indexDefinition.getName();
            String indexName = ((SQLIdentifierExpr) sqlName).getName();
            if (indexName.charAt(0) == '`' && indexName.charAt(indexName.length() - 1) == '`') {
                indexName = indexName.substring(1, indexName.length() - 1);
            }
            indexName = tableName + "_" + indexName;
            ((SQLIdentifierExpr) sqlName).setName(indexName);
        } else {
            throw new IllegalArgumentException(LOG_PREFIX + "获取索引名失败");
        }
    }

    @Override
    public boolean visit(MySqlCreateTableStatement x) {
        if (x.getTableOptions() != null) {
            x.getTableOptions().clear();
        }
        if (x.getComment() != null) {
            x.setComment(null);
        }
        Iterator<SQLTableElement> iterator = x.getTableElementList().iterator();
        while (iterator.hasNext()) {
            SQLTableElement sqlTableElement = iterator.next();
            if (sqlTableElement instanceof MySqlTableIndex) {
                continue;
            }
            if (sqlTableElement instanceof MySqlPrimaryKey) {
                continue;
            }
            if (sqlTableElement instanceof MySqlUnique) {
                continue;
            }
            if (sqlTableElement instanceof MySqlKey) {
                if (sqlTableElement.getParent() == null) {
                    sqlTableElement.setParent(x);
                }
            }
        }
        return true;
    }

    @Override
    public boolean visit(MySqlPrimaryKey x) {
        this.visit(x.getIndexDefinition());
        return true;
    }

    @Override
    public boolean visit(MySqlKey x) {
        this.visit(x.getIndexDefinition());
        x.getName().accept(this);
        for (SQLSelectOrderByItem item : x.getColumns()) {
            item.accept(this);
        }

        return false;
    }

    @Override
    public boolean visit(SQLIndexDefinition x) {
        if (x.getOptions() != null && "BTREE".equalsIgnoreCase(x.getOptions().getIndexType())) {
            x.getOptions().setIndexType(null);
        }
        return true;
    }
}
```