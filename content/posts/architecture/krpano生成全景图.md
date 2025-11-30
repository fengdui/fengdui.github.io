---
title: "krpano生成全景图"
date: "2016-06-22"
tags: ["架构"]
ShowToc: false
TocOpen: false
---
需求是上传一张图片 然后生成一张全景图 前端加载全景图渲染展示  
使用krpano-1.19-pr4版本  
上传图片后使用命令行生成全景图文件  
```
public void generatePanorama(String filePath) {

//        String command = "D:\\fd\\krpano\\krpano-1.19-pr4\\krpanotools64.exe makepano -config=templates\\vtour-multires.config "+filePath;
    List<String> commands = new ArrayList();
    Process process = null;
    commands.add("D:\\fd\\\\krpano\\krpano-1.19-pr4\\krpanotools64.exe");
    commands.add("makepano");
    commands.add("-config=templates\\vtour-multires.config");
//        commands.add("C:\\Users\\fd\\Pictures\\fd.jpg");
    commands.add(filePath);
    ProcessBuilder pb = new ProcessBuilder(commands);
    pb.redirectErrorStream(true);
    try {
        process = pb.start();
//            process = Runtime.getRuntime().exec(command);
        new Thread(new ProcessStream(process.getInputStream(), null)).start();
//            new Thread(new ProcessStream(null, process.getOutputStream())).start();
        process.waitFor();
    } catch (Exception e) {
        e.printStackTrace();
        throw new CheckedException(Result.COMMAND_FAILURE);
    }
    finally {
        if (null != process) {
            process.destroy();
        }
    }
}
```
每个全景图对应一个独立的文件夹（如：81a1d250-4d8d-4512-97f8-b5e281db65b8）  
每个文件夹包含固定的tour.xml配置文件  
配置文件预先设置好各种参数（皮肤、VR设置、缩略图等）  

通过接口返回tour.xml文件路径  
```
@RequestMapping("/{panoramaId}")
public ModelAndView view(@PathVariable("panoramaId") String panoramaId) {
    mv.addObject("xmlPath", "/resources/krpano/"+panoramaId+"/tour.xml");
    return mv;
}
```
前端集成
```
<script src="${ctx!}/resources/krpano/tour.js"></script>
<script>
    embedpano({
        swf:"${ctx!}/resources/krpano/tour.swf", 
        xml:"${xmlPath!}",  // 动态传入的XML配置文件路径
        target:"pano", 
        html5:"auto", 
        mobilescale:1.0
    });
</script>
```
tour.js - krpano的JavaScript嵌入脚本  
tour.swf - krpano的Flash播放器（用于兼容性）  