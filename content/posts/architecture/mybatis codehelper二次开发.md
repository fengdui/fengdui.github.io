---
title: "mybatis codehelper二次开发"
date: "2020-08-28"
tags: ["架构"]
ShowToc: false
TocOpen: false
---

codehelper二次开发 因为工程里面有自定义的mapper基类 需要生成的xml文件有对应的sql片段  
于是需要在codehelper的mapper基类中添加一些方法 或者去掉一些方法  

GenMapperService是生成mapper的服务类 也就是我们的xml  
在这个类修改 根据我们的pojo 生成一些代码片段 比如insert update 分页查询等等 删除是逻辑删除  
思路是这样 看源码即可  
改完之后的类
```
package com.ccnode.codegenerator.genCode;

import com.ccnode.codegenerator.pojo.*;
import com.ccnode.codegenerator.pojoHelper.OnePojoInfoHelper;
import com.ccnode.codegenerator.util.*;
import com.ccnode.codegenerator.enums.FileType;
import com.ccnode.codegenerator.enums.MethodName;
import com.ccnode.codegenerator.function.EqualCondition;
import com.ccnode.codegenerator.function.MapperCondition;
import com.ccnode.codegenerator.pojoHelper.GenCodeResponseHelper;
import com.ccnode.codegenerator.pojoHelper.OnePojoInfoHelper;
import com.google.common.collect.Lists;
import org.apache.commons.lang3.StringUtils;
import org.apache.commons.lang3.tuple.Pair;
import org.slf4j.Logger;

import java.util.List;

/**
 * What always stop you is what you always believe.
 * <p>
 * Created by zhengjun.du on 2016/05/28 21:14
 */
public class GenMapperService {

    private final static Logger LOGGER = LoggerWrapper.getLogger(GenMapperService.class);

    private static String COMMA = ",";
    private static String COMMA_AND_BLANK = ", ";

    public static void genMapper(GenCodeResponse response) {
        for (OnePojoInfo pojoInfo : response.getPojoInfos()) {
            try{
                GeneratedFile fileInfo = GenCodeResponseHelper.getByFileType(pojoInfo, FileType.MAPPER, response);
                genMapper(response,pojoInfo,fileInfo);
            }catch(Throwable e){
                LOGGER.error("GenMapperService genMapper error", e);
                response.failure("GenMapperService genMapper error");
            }
        }
    }

    private static void genMapper(GenCodeResponse response,OnePojoInfo onePojoInfo, GeneratedFile fileInfo) {
        List<String> oldLines = fileInfo.getOldLines();
        ListInfo<String> listInfo = new ListInfo<String>();
        if(oldLines.isEmpty()){
            onePojoInfo.setSuffix(fileInfo.getSuffix());
            listInfo.setFullList(getMapperHeader(onePojoInfo));
        }else{
            listInfo.setFullList(oldLines);
        }

        Pair<Integer, Integer> posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<resultMap id=\"BaseResultMap\" type=", "</resultMap>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genAllColumnMap(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                String match1 = RegexUtil.getMatch("result(.*)property", o1);
                String match2 = RegexUtil.getMatch("result(.*)property", o2);
                if(StringUtils.isBlank(match1) ){
                    return false;
                }
                return match1.equals(match2);
            }
        });

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<sql id=\"baseColumn\">", "</sql>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genAllColumn(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {

                String match1 = RegexUtil.getMatch("[0-9A-Za-z_ ,]{1,100}", o1);
                String match2 = RegexUtil.getMatch("[0-9A-Za-z_ ,]{1,100}", o2);
                if(StringUtils.isBlank(match1) ){
                    return false;
                }
                return match1.equals(match2);
            }
        });

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<sql id=\"page-limit\">", "</sql>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(getLimit(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<sql id=\"qc\">", "</sql>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(getQC(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<sql id=\"set\">", "</sql>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(getSet(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<sql id=\"batchSet\">", "</sql>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(getBatchSet(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<insert id=\""+ "add" +"\"", "</insert>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genAddMethod(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return  o1.equals(o2);
            }
        });

        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<update id=\""+ "update" +"\"", "</update>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genUpdateMethod(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return  o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());


        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<insert id=\""+ "batchInsert" +"\"", "</insert>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genBatchInsertMethod(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                String match1 = RegexUtil.getMatch("#\\{item.(.*)\\}", o1);
                String match2 = RegexUtil.getMatch("#\\{item.(.*)\\}", o2);
                if(StringUtils.isBlank(match1) ){
                    return false;
                }
                return  match1.equals(match2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<update id=\""+ "batchUpdate" +"\"", "</update>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genBatchUpdateMethod(onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                String match1 = RegexUtil.getMatch("#\\{item.(.*)\\}", o1);
                String match2 = RegexUtil.getMatch("#\\{item.(.*)\\}", o2);
                if(StringUtils.isBlank(match1) ){
                    return false;
                }
                return  match1.equals(match2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<select id=\"list\"", "</select>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genSelectMethod(response,onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<select id=\"countByQc\"", "</select>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genCountQC(response,onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        posPair = ReplaceUtil
                .getPos(listInfo.getFullList(), "<select id=\"getByQc\"", "</select>", new MapperCondition());
        listInfo.setPos(posPair);
        listInfo.setNewSegments(genGetQC(response,onePojoInfo));
        ReplaceUtil.merge(listInfo, new EqualCondition<String>() {
            @Override
            public boolean isEqual(String o1, String o2) {
                if(StringUtils.isBlank(o1) ){
                    return false;
                }
                return o1.equals(o2);
            }
        });
        fileInfo.setNewLines(listInfo.getFullList());

        List<String> newList = listInfo.getFullList();
        newList = adjustList(newList);
        fileInfo.setNewLines(newList);
    }

    private static List<String> adjustList(List<String> newList) {
        newList = PojoUtil.avoidEmptyList(newList);
        List<String> retList = Lists.newArrayList();
        for (String s : newList) {
            if(!s.contains("</mapper>")){
                retList.add(s);
            }
        }
        retList.add("</mapper>");
        return retList;
    }

    public static List<String> getMapperHeader(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        retList.add( "<?xml version=\"1.0\" encoding=\"UTF-8\" ?>");
        retList.add( "<!DOCTYPE mapper PUBLIC \"-//mybatis.org//DTD Mapper 3.0//EN\" \"http://mybatis.org/dtd/mybatis-3-mapper.dtd\" >") ;
        retList.add("<mapper namespace=\"" + "XX" +"."+ onePojoInfo.getPojoName().substring(0, onePojoInfo.getPojoName().length()-2) + onePojoInfo.getSuffix() + "\">");
        retList.add(StringUtils.EMPTY);
        retList.add(
                GenCodeUtil.ONE_RETRACT+ "<resultMap id=\"BaseResultMap\" type=\""+onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName()+"\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</resultMap>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT+ "<sql id=\"baseColumn\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</sql>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT+ "<sql id=\"page-limit\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</sql>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"qc\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</sql>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"set\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</sql>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"batchSet\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</sql>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT + "<insert id=\"add\" parameterType=\""+ onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName() +"\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</insert>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT + "<update id=\"update\" parameterType=\""+ onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName() +"\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</update>");

        retList.add(StringUtils.EMPTY);
        retList.add( GenCodeUtil.ONE_RETRACT + "<insert id=\"batchInsert\" parameterType=\""+ onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName() +"\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</insert>");

        retList.add(StringUtils.EMPTY);
        retList.add( GenCodeUtil.ONE_RETRACT + "<update id=\"batchUpdate\"> parameterType=\""+ onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName() + "\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</update>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT+ "<select id=\"list\" parameterType=\"QC\" resultMap=\"BaseResultMap\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</select>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT+ "<select id=\"countByQc\" parameterType=\"QC\" resultType=\"java.lang.Long\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</select>");

        retList.add(StringUtils.EMPTY);
        retList.add(GenCodeUtil.ONE_RETRACT+ "<select id=\"getByQc\" parameterType=\"QC\" resultMap=\"BaseResultMap\">");
        retList.add(GenCodeUtil.ONE_RETRACT+"</select>");

        retList.add("</mapper>");
        return retList;
    }

    private static List<String> genAllColumnMap(OnePojoInfo onePojoInfo){
        List<String> retList = Lists.newArrayList();
        retList.add(GenCodeUtil.ONE_RETRACT+ "<resultMap id=\"BaseResultMap\" type=\""+onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName()+"\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "<result column=\"id\" property=\"id\"/>");
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String fieldName = fieldInfo.getFieldName();
            retList.add(String.format("%s<result column=\"%s\" property=\"%s\"/>",
                    GenCodeUtil.TWO_RETRACT, GenCodeUtil.getUnderScore(fieldName),fieldName));
        }
        retList.add(GenCodeUtil.ONE_RETRACT+"</resultMap>");
        return retList;

    }

    private static List<String> genAllColumn(OnePojoInfo onePojoInfo) {

        List<String> retList = Lists.newArrayList();
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"baseColumn\">");
//        retList.add(GenCodeUtil.THREE_RETRACT + "<![CDATA[");
        int index = 0;
        StringBuilder stringBuilder = new StringBuilder(GenCodeUtil.TWO_RETRACT);
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String s = GenCodeUtil.getUnderScore(fieldInfo.getFieldName()) +COMMA_AND_BLANK ;
            if(index == onePojoInfo.getPojoFieldInfos().size() - 1){
                s = s.replace(COMMA_AND_BLANK, StringUtils.EMPTY);
            }
            stringBuilder.append(s);
            index ++;
        }
        retList.add(stringBuilder.toString());
//        retList.add(GenCodeUtil.THREE_RETRACT + "]]>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</sql>");
        return retList;

    }

    /**
     *  <sql id="page-limit">
     *         <if test="offset !=null and limit != null">
     *             LIMIT #{offset},#{limit}
     *         </if>
     *     </sql>
     * @param onePojoInfo
     * @return
     */
    private static List<String> getLimit(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"page-limit\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "<if test=\"offset !=null and limit != null\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "LIMIT #{offset},#{limit}");
        retList.add(GenCodeUtil.TWO_RETRACT + "</if>");
        retList.add(GenCodeUtil.ONE_RETRACT + "");
        return retList;
    }

    private static List<String> getQC(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"qc\">");
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String fieldName = fieldInfo.getFieldName();
            String testCondition = GenCodeUtil.TWO_RETRACT +  String.format("<if test=\"%s != null\">",fieldName);
            String updateField =  String.format("AND %s = #{%s}", GenCodeUtil.getUnderScore(fieldName), fieldName);
            retList.add(testCondition + " " + updateField + " " + "</if>");

        }
        retList.add(GenCodeUtil.ONE_RETRACT + "</sql>");
        return retList;
    }

    private static List<String> getSet(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"set\">");
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String fieldName = fieldInfo.getFieldName();
            String s = GenCodeUtil.TWO_RETRACT + String.format("<if test=\"%s != null\">%s = #{%s}, </if>"
                    ,fieldName, GenCodeUtil.getUnderScore(fieldName), fieldName);
            retList.add(s);
        }
        retList.add(GenCodeUtil.ONE_RETRACT + "</sql>");
        return retList;
    }

    private static List<String> getBatchSet(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        retList.add(GenCodeUtil.ONE_RETRACT + "<sql id=\"batchSet\">");
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String fieldName = fieldInfo.getFieldName();
            String s = GenCodeUtil.TWO_RETRACT + String.format("<if test=\"item.%s != null\">%s = #{item.%s}, </if>"
                    ,fieldName, GenCodeUtil.getUnderScore(fieldName), fieldName);
            retList.add(s);
        }
        retList.add(GenCodeUtil.ONE_RETRACT + "</sql>");
        return retList;
    }

    private static List<String> genAddMethod(OnePojoInfo onePojoInfo ) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<insert id=\"add\" parameterType=" + "\""+ onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName() +"\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "INSERT INTO " + tableName );
        retList.add(GenCodeUtil.TWO_RETRACT + "<set>");
//        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
//            String fieldName = fieldInfo.getFieldName();
//            String s = GenCodeUtil.THREE_RETRACT +  String.format("<if test=\"pojo.%s != null\"> %s, </if>"
//                ,fieldName, GenCodeUtil.getUnderScore(fieldName));
//            retList.add(s);
//        }
//        retList.add(GenCodeUtil.TWO_RETRACT + "</trim>");
//        retList.add(GenCodeUtil.TWO_RETRACT + "VALUES");
//        retList.add(GenCodeUtil.TWO_RETRACT + "<trim prefix=\"(\" suffix=\")\" suffixOverrides=\",\">");
//        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
//            String fieldName = fieldInfo.getFieldName();
//            String s = GenCodeUtil.THREE_RETRACT + String.format("<if test=\"pojo.%s != null\"> #{pojo.%s}, </if>"
//                ,fieldName,fieldName);
//            retList.add(s);
//        }
        retList.add(GenCodeUtil.THREE_RETRACT + "<include refid=\"set\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + "</set>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</insert>");
        return retList;
    }
    private static List<String> genUpdateMethod(OnePojoInfo onePojoInfo ) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<update id=\"update\" parameterType=\"" + onePojoInfo.getPojoPackage() +"." + onePojoInfo.getPojoName() +"\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "UPDATE INTO " + tableName );
        retList.add(GenCodeUtil.TWO_RETRACT + "<set>");
        retList.add(GenCodeUtil.THREE_RETRACT + "<include refid=\"set\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + "</set>");
        retList.add(GenCodeUtil.TWO_RETRACT + "WHERE id = #{id}");
        retList.add(GenCodeUtil.ONE_RETRACT + "</update>");
        return retList;
    }

    private static List<String> genAddsMethod(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add( GenCodeUtil.ONE_RETRACT + "<insert id=\"batchInsert\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "INSERT INTO " + tableName + "(");
        retList.add(GenCodeUtil.TWO_RETRACT + "<include refid=\"baseColumn\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + ")VALUES");
        retList.add(GenCodeUtil.TWO_RETRACT + "<foreach collection=\"list\" item=\"item\" index=\"index\" separator=\",\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "(");
        int index = 0;
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String fieldName = fieldInfo.getFieldName();
            String s = GenCodeUtil.THREE_RETRACT +  String.format("#{item.%s},",fieldName);
            if(index == onePojoInfo.getPojoFieldInfos().size() - 1){
                s = s.replace(COMMA, StringUtils.EMPTY);
            }
            retList.add(s);
            index ++;
        }
        retList.add(GenCodeUtil.THREE_RETRACT + ")");
        retList.add(GenCodeUtil.TWO_RETRACT + "</foreach>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</insert>");
        return retList;
    }
    private static List<String> genBatchInsertMethod(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<insert id=\"batchInsert\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "<foreach collection=\"list\" item=\"item\" index=\"index\" separator=\";\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "INSERT INTO " + tableName);
        retList.add(GenCodeUtil.THREE_RETRACT + "<set>");
        retList.add(GenCodeUtil.FOUR_RETRACT + "<include refid=\"batchSet\"/>");
        retList.add(GenCodeUtil.THREE_RETRACT + "</set>");
        retList.add(GenCodeUtil.TWO_RETRACT + "</foreach>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</insert>");
        return retList;
    }

    private static List<String> genBatchUpdateMethod(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<update id=\"batchUpdate\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "<foreach collection=\"list\" item=\"item\" index=\"index\" separator=\";\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "Update " + tableName);
        retList.add(GenCodeUtil.THREE_RETRACT + "<set>");
        retList.add(GenCodeUtil.FOUR_RETRACT + "<include refid=\"batchSet\"/>");
        retList.add(GenCodeUtil.THREE_RETRACT + "</set>");
        retList.add(GenCodeUtil.THREE_RETRACT + "WHERE id = #{item.id}\n");
        retList.add(GenCodeUtil.TWO_RETRACT + "</foreach>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</update>");
        return retList;
    }

    private static List<String> genAddsMethodV2(OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add( GenCodeUtil.ONE_RETRACT + "<insert id=\"batchInsert\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "INSERT INTO " + tableName + "(");
        retList.add(GenCodeUtil.TWO_RETRACT + "<include refid=\"baseColumn\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + ")VALUES");
        retList.add(GenCodeUtil.TWO_RETRACT + "<foreach collection=\"list\" item=\"item\" index=\"index\" separator=\",\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "(");
        int index = 0;
        for (PojoFieldInfo fieldInfo : onePojoInfo.getPojoFieldInfos()) {
            String fieldName = fieldInfo.getFieldName();
            String s = GenCodeUtil.THREE_RETRACT +  String.format("#{item.%s},",fieldName);
            if(index == onePojoInfo.getPojoFieldInfos().size() - 1){
                s = s.replace(COMMA, StringUtils.EMPTY);
            }
            retList.add(s);
            index ++;
        }
        retList.add(GenCodeUtil.THREE_RETRACT + ")");
        retList.add(GenCodeUtil.TWO_RETRACT + "</foreach>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</insert>");
        return retList;
    }


    private static List<String> genSelectMethod(GenCodeResponse response, OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<select id=\"list\" parameterType=\"QC\" resultMap=\"BaseResultMap\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "SELECT <include refid=\"baseColumn\"/>"  );
        retList.add(GenCodeUtil.TWO_RETRACT + "FROM " + tableName  );
        retList.add(GenCodeUtil.TWO_RETRACT + "<where>");
        retList.add(GenCodeUtil.THREE_RETRACT + "`is_deleted` = \"N\"");
        retList.add(GenCodeUtil.THREE_RETRACT + "<include refid=\"qc\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + "</where>");
        retList.add(GenCodeUtil.TWO_RETRACT + "<if test=\"orderBy != null\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "ORDER BY ${orderBy}");
        retList.add(GenCodeUtil.TWO_RETRACT + "</if>");
        retList.add(GenCodeUtil.TWO_RETRACT + "<include refid=\"page-limit\"/>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</select>");
        return retList;
    }


    private static List<String> genCountQC(GenCodeResponse response, OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<select id=\"countByQc\" parameterType=\"QC\" resultType=\"java.lang.Long\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "SELECT COUNT(1)"  );
        retList.add(GenCodeUtil.TWO_RETRACT + "FROM " + tableName  );
        retList.add(GenCodeUtil.TWO_RETRACT + "<where>");
        retList.add(GenCodeUtil.THREE_RETRACT + "`is_deleted` = \"N\"");
        retList.add(GenCodeUtil.THREE_RETRACT + "<include refid=\"qc\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + "</where>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</select>");
        return retList;
    }

    private static List<String> genGetQC(GenCodeResponse response, OnePojoInfo onePojoInfo) {
        List<String> retList = Lists.newArrayList();
        String tableName = GenCodeUtil.getUnderScore(onePojoInfo.getTableName());
        retList.add(GenCodeUtil.ONE_RETRACT + "<select id=\"countByQc\" parameterType=\"QC\" resultMap=\"BaseResultMap\">");
        retList.add(GenCodeUtil.TWO_RETRACT + "SELECT <include refid=\"baseColumn\"/>"  );
        retList.add(GenCodeUtil.TWO_RETRACT + "FROM " + tableName  );
        retList.add(GenCodeUtil.TWO_RETRACT + "<where>");
        retList.add(GenCodeUtil.THREE_RETRACT + "`is_deleted` = \"N\"");
        retList.add(GenCodeUtil.THREE_RETRACT + "<include refid=\"qc\"/>");
        retList.add(GenCodeUtil.TWO_RETRACT + "</where>");
        retList.add(GenCodeUtil.TWO_RETRACT + "<if test=\"orderBy != null\">");
        retList.add(GenCodeUtil.THREE_RETRACT + "ORDER BY ${orderBy}");
        retList.add(GenCodeUtil.TWO_RETRACT + "</if>");
        retList.add(GenCodeUtil.TWO_RETRACT + "<include refid=\"page-limit\"/>");
        retList.add(GenCodeUtil.ONE_RETRACT + "</select>");
        return retList;
    }


    public static void main(String[] args) {
//        Pattern day3DataPattern = Pattern.compile("var dataSK = (.*)");
        String match = RegexUtil.getMatch("(.*)pojo.(.*)",
                "refund_finish_time = #{pojo.refundFinishTime},");
        System.out.println(match);

    }

}
```
最后打包 idea加载插件