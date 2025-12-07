---
title: "doiteasy二次开发 无需注解只需注释生成接口文档"
date: "2020-06-23"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
像swagger这种 生成接口文档 需要加一堆注释  
但是其实我们的代码一般都有注释 可以解析注释生成接口文档 这几乎是零成本的  
同事使用了doiteasy这个开源项目 可以直接生成接口文档 不过支持了dubbo 网关 开放平台的接口解析  
现在我们要对doiteasy进行二次开发 支持spring mvc http接口  
springmvc接口有它自己的一些注解 比如@RequestMapping @GetMapping @PostMapping 等 我们需要解析它  

```
package com.github.doiteasy.apidocs.parser;

import com.github.doiteasy.apidocs.LogUtils;
import com.github.javaparser.ast.Node;
import com.github.javaparser.ast.NodeList;
import com.github.javaparser.ast.body.ClassOrInterfaceDeclaration;
import com.github.javaparser.ast.body.MethodDeclaration;
import com.github.javaparser.ast.expr.*;
import com.github.javaparser.ast.type.Type;
import com.github.doiteasy.apidocs.ParseUtils;
import com.github.doiteasy.apidocs.Utils;

import java.util.ArrayList;
import java.util.Arrays;
import java.util.List;
import java.util.Optional;

/**
 * use for spring mvc
 */
public class SpringControllerParser extends AbsControllerParser {

    private final static String[] MAPPING_ANNOTATIONS = {
            "GetMapping", "PostMapping", "PutMapping",
            "PatchMapping", "DeleteMapping", "RequestMapping"
    };

    @Override
    protected void parseMethodDocs(ClassOrInterfaceDeclaration c){
        c.findAll(MethodDeclaration.class).stream()
            .filter(m ->  m.getAnnotationByName("GetMapping").isPresent()
                    || m.getAnnotationByName("PostMapping").isPresent()
                    || m.getAnnotationByName("PutMapping").isPresent()
                    || m.getAnnotationByName("PatchMapping").isPresent()
                    || m.getAnnotationByName("DeleteMapping").isPresent()
                    || m.getAnnotationByName("RequestMapping").isPresent()
            ).forEach(m -> {
            AnnotationExpr expr = m.getAnnotationByName("GetMapping").orElseGet(
                    () -> m.getAnnotationByName("PostMapping").orElseGet(
                            () -> m.getAnnotationByName("PutMapping").orElseGet(
                                    () -> m.getAnnotationByName("PatchMapping").orElseGet(
                                            () -> m.getAnnotationByName("DeleteMapping").orElseGet(
                                                    () -> m.getAnnotationByName("RequestMapping").get()
                                            )
                                    )
                            )
                    )
            );
            Optional.of(expr).ifPresent(an -> {
                    RequestNode requestNode = new RequestNode();
                    requestNode.setControllerNode(controllerNode);
                    requestNode.setMethodName(m.getNameAsString());
                    m.getAnnotationByClass(Deprecated.class).ifPresent(f -> {requestNode.setDeprecated(true);});
                    m.getJavadoc().ifPresent( d -> {
                        String description = d.getDescription().toText();
                        requestNode.setDescription(description);
                        d.getBlockTags().stream()
                                .filter(t -> t.getTagName().equals("param"))
                                .forEach(t ->{
                                    ParamNode paramNode = new ParamNode();
                                    paramNode.setName(t.getName().get());
                                    paramNode.setDescription(t.getContent().toText());
                                    requestNode.addParamNode(paramNode);
                                });
                    });

                    m.getParameters().forEach(p -> {
                        String paraName  = p.getName().asString();
                        ParamNode paramNode = requestNode.getParamNodeByName(paraName);
                        if(paramNode != null){
                            paramNode.setType(ParseUtils.unifyType(p.getType().asString()));
                        }
                    });

                    com.github.javaparser.ast.type.Type resultClassType = null;
//                        if(an instanceof SingleMemberAnnotationExpr){
//                            resultClassType = ((ClassExpr) ((SingleMemberAnnotationExpr) an).getMemberValue()).getType();
//                        }else if(an instanceof NormalAnnotationExpr){
//                            for(MemberValuePair pair : ((NormalAnnotationExpr)an).getPairs()){
//                                final String pairName = pair.getNameAsString();
//                                if("result".equals(pairName) || "value".equals(pairName)){
//                                    resultClassType = ((ClassExpr) pair.getValue()).getType();
//                                }else if(pairName.equals("url")){
//                                    requestNode.setUrl(((StringLiteralExpr) pair.getValue()).getValue());
//                                }else if(pairName.equals("method")){
//                                    requestNode.addMethod(((StringLiteralExpr) pair.getValue()).getValue());
//                                }
//                            }
//                        }

                if(an instanceof SingleMemberAnnotationExpr){
                    try {
                        resultClassType = ((ClassExpr) ((SingleMemberAnnotationExpr) an).getMemberValue()).getType();
                    } catch (Exception e) {
                    }
                    SingleMemberAnnotationExpr singleMemberAnnotationExpr = (SingleMemberAnnotationExpr) an;
                    requestNode.setUrl(singleMemberAnnotationExpr.getMemberValue().toString().replace("\"", ""));

                }else if(an instanceof NormalAnnotationExpr){
                    for(MemberValuePair pair : ((NormalAnnotationExpr)an).getPairs()){
                        final String pairName = pair.getNameAsString();
                        if("result".equals(pairName)){
                            try {
                                resultClassType = ((ClassExpr) pair.getValue()).getType();
                            } catch (Exception e) {
                            }
                        }else if(pairName.equals("value")){
                            requestNode.setUrl(pair.getValue().toString());
                        }else if(pairName.equals("method")){
//                                    requestNode.addMethod(((StringLiteralExpr) pair.getValue()).getValue());
                            requestNode.addMethod(pair.getValue().toString());
                        }
                    }

                }

                    afterHandleMethod(requestNode, m);

                    if(resultClassType == null){
                        if(m.getType() == null){
                            return;
                        }
                        resultClassType = m.getType();
                    }

                    ResponseNode responseNode = new ResponseNode();
                    responseNode.setRequestNode(requestNode);
                    ParseUtils.parseClassNodeByType(javaFile, responseNode, resultClassType.getElementType());
                    requestNode.setResponseNode(responseNode);
                    controllerNode.addRequestNode(requestNode);
                });
            });
    }


    @Override
    protected void afterHandleController(ControllerNode controllerNode, ClassOrInterfaceDeclaration clazz) {
        clazz.getAnnotationByName("RequestMapping").ifPresent(a -> {
            if (a instanceof SingleMemberAnnotationExpr) {
                String baseUrl = ((SingleMemberAnnotationExpr) a).getMemberValue().toString();
                controllerNode.setBaseUrl(Utils.removeQuotations(baseUrl));
                return;
            }
            if (a instanceof NormalAnnotationExpr) {
                ((NormalAnnotationExpr) a).getPairs().stream()
                        .filter(v -> isUrlPathKey(v.getNameAsString()))
                        .findFirst()
                        .ifPresent(p -> {
                            controllerNode.setBaseUrl(Utils.removeQuotations(p.getValue().toString()));
                        });
            }
        });

    }

    @Override
    protected void afterHandleMethod(RequestNode requestNode, MethodDeclaration md) {
        md.getAnnotations().forEach(an -> {
            String name = an.getNameAsString();
            if (Arrays.asList(MAPPING_ANNOTATIONS).contains(name)) {
                String method = Utils.getClassName(name).toUpperCase().replace("MAPPING", "");
                if (!"REQUEST".equals(method)) {
                    requestNode.addMethod(RequestMethod.valueOf(method).name());
                }

                if (an instanceof NormalAnnotationExpr) {
                    ((NormalAnnotationExpr) an).getPairs().forEach(p -> {
                        String key = p.getNameAsString();
                        if (isUrlPathKey(key)) {
                            requestNode.setUrl(Utils.removeQuotations(p.getValue().toString()));
                        }

                        if ("headers".equals(key)) {
                            Expression methodAttr = p.getValue();
                            if (methodAttr instanceof ArrayInitializerExpr) {
                                NodeList<Expression> values = ((ArrayInitializerExpr) methodAttr).getValues();
                                for (Node n : values) {
                                    String[] h = n.toString().split("=");
                                    requestNode.addHeaderNode(new HeaderNode(h[0], h[1]));
                                }
                            } else {
                                String[] h = p.getValue().toString().split("=");
                                requestNode.addHeaderNode(new HeaderNode(h[0], h[1]));
                            }
                        }

                        if ("method".equals(key)) {
                            Expression methodAttr = p.getValue();
                            if (methodAttr instanceof ArrayInitializerExpr) {
                                NodeList<Expression> values = ((ArrayInitializerExpr) methodAttr).getValues();
                                for (Node n : values) {
                                    requestNode.addMethod(RequestMethod.valueOf(Utils.getClassName(n.toString())).name());
                                }
                            } else {
                                requestNode.addMethod(RequestMethod.valueOf(Utils.getClassName(p.getValue().toString())).name());
                            }
                        }
                    });
                }
                if (an instanceof SingleMemberAnnotationExpr) {
                    String url = ((SingleMemberAnnotationExpr) an).getMemberValue().toString();
                    requestNode.setUrl(Utils.removeQuotations(url));
                    return;
                }

            }
        });

        md.getParameters().forEach(p -> {
            String paraName = p.getName().asString();
            ParamNode paramNode = requestNode.getParamNodeByName(paraName);
            if (paramNode != null) {

                p.getAnnotations().forEach(an -> {
                    String name = an.getNameAsString();
                    if (!"RequestParam".equals(name) && !"RequestBody".equals(name) && !"PathVariable".equals(name)) {
                        return;
                    }

                    if ("RequestBody".equals(name)) {
                        setRequestBody(paramNode, p.getType());
                    }

                    if (an instanceof MarkerAnnotationExpr) {
                        paramNode.setRequired(true);
                        return;
                    }

                    if(an instanceof SingleMemberAnnotationExpr){
                        paramNode.setName(((StringLiteralExpr) ((SingleMemberAnnotationExpr) an).getMemberValue()).getValue());
                        return;
                    }

                    if (an instanceof NormalAnnotationExpr) {
                        ((NormalAnnotationExpr) an).getPairs().forEach(pair -> {
                            String exprName = pair.getNameAsString();
                            if("required".equals(exprName)){
                                Boolean exprValue = ((BooleanLiteralExpr) pair.getValue()).getValue();
                                paramNode.setRequired(Boolean.valueOf(exprValue));
                            }else if("value".equals(exprName)){
                                String exprValue = ((StringLiteralExpr) pair.getValue()).getValue();
                                paramNode.setName(exprValue);
                            }
                        });
                    }

                });

                //如果参数是个对象
                if(!paramNode.isJsonBody() && ParseUtils.isModelType(p.getType().asString())){
                    ClassNode classNode = new ClassNode();
                    ParseUtils.parseClassNodeByType(getControllerFile(), classNode, p.getType());
                    List<ParamNode> paramNodeList = new ArrayList<>();
                    toParamNodeList(paramNodeList, classNode, "");
                    requestNode.getParamNodes().remove(paramNode);
                    requestNode.getParamNodes().addAll(paramNodeList);
                }
            }
        });
    }

    private void setRequestBody(ParamNode paramNode, Type paramType) {
        if (ParseUtils.isModelType(paramType.asString())) {
            ClassNode classNode = new ClassNode();
            ParseUtils.parseClassNodeByType(getControllerFile(), classNode, paramType);
            paramNode.setJsonBody(true);
            paramNode.setDescription(classNode.toJsonApi());
        }
    }

    private void toParamNodeList( List<ParamNode> paramNodeList, ClassNode formNode, String parentName){
        formNode.getChildNodes().forEach(filedNode -> {
            if(filedNode.getChildNode() != null){
                toParamNodeList(paramNodeList, filedNode.getChildNode(), filedNode.getName() + ".");
            }else{
                ParamNode paramNode = new ParamNode();
                paramNode.setName(parentName + filedNode.getName());
                paramNode.setType(filedNode.getType());
                paramNode.setDescription(filedNode.getDescription());
                paramNodeList.add(paramNode);
            }
        });
    }

    private boolean isUrlPathKey(String name) {
        return name.equals("path") || name.equals("value");
    }
}

```
AbsControllerParser是一个抽象类，是框架提供的  
com.github.doiteasy 开源的可以搜索一下  