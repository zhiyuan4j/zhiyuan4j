---
layout: post
title:  "ppt use poi"
excerpt: "java生成ppt"
categories: [poi,ppt]
commnet: true
---

# Apache POI 生成 PPT

作为一个后端开发，我们最常遇到的需求就是，报表导出，数据导出，这时我们用到的是Excel，你可以用Apache POI，也可以用二次开发优化过的ali的easy excel，这些都没什么难度。

偶有需求需要你生成PPT的，首先这个需求一般来说，**都是客户留个邮箱，然后你把客户下载ppt的请求扔到消息队列里慢慢处理的**

让程序处理ppt，无非是填充点文字和图片，再高级一点的话，这需求就有点过分了（比如根据塞个表格数据，然后还得根据这个表格数据生成个图表？

本文只介绍填充文字和图片的经验。

生成PPT的关键在于PPT模板的制作。

### 如何制作带有图文的PPT模板。

以下截图以2013版PPT为例

1. 打开PPT母版视图

![ppt1](..\..\img\ppt1.png)

2. 创建你自己的样式，**字体样式什么的一定要在母版视图下调整**，不然代码填充时，样式就会变成代码里默认的了

   ![ppt2](..\..\img\ppt2.png)

3. 调整好，关闭母版视图，以这个ppt母版来新增一页ppt。

### 解决mvn后模板文件损坏

添加`maven-resources-plugin`

```xml
<plugin>
    <groupId>org.apache.maven.plugins</groupId>
    <artifactId>maven-resources-plugin</artifactId>
    <configuration>
        <nonFilteredFileExtensions>
            <nonFilteredFileExtension>ppt</nonFilteredFileExtension>
            <nonFilteredFileExtension>pptx</nonFilteredFileExtension>
        </nonFilteredFileExtensions>
    </configuration>
</plugin>
```

### 加载PPTX

```java
// 没什么好说的，官方文档也有示例
private XMLSlideShow loadPPTTemplate(String templateFilename) {
    try {
        String path = PPTDownloadServiceImpl.class.getClassLoader().getResource("template/" + templateFilename).getPath();
        return new XMLSlideShow(new FileInputStream(new File(path)));
    } catch (IOException | NullPointerException e) {
        logger.error("fail to load ppt template", e);
        return null;
    }
}
```

### 填充文字

```java
// 获取当前PPT页所有的组件
List<XSLFShape> shapes = slide.getShapes();
// 遍历这些组件，找到要填充那个组件
for (XSLFShape shape : shapes) {
    if (shape instanceof XSLFAutoShape) {
        // 这里用组件里的text是最保险的匹配方式，没找到比较程序员方式的自定义占位符id
        XSLFAutoShape autoShape = (XSLFAutoShape) shape;
        // 这里trim()下就是经验之谈了，因为你ppt里一步小心就可多敲个空格，或者ppt自动给你追加个空格
        //trim()下以防万一，或者replaceAll("\\s", "")更实用
        String placeHolder = autoShape.getText().trim();
        if ("目标填充区占位符".equals(palceHolder)) {
            autoShape.setText("修改后的标发的说法但是发射点的发生发射点发题");
        }
    }
}
```

### 填充图片(难点)

官方文档中只给你一个例子，叫你怎么给ppt插入一张图片。但是没教你**怎么方便的规规矩矩就的把你的图片放在你的图片占位符里**。我进行了大量的谷歌，都没有什么直接能用的结果。最后在自己对api的辛苦研究中，找到了解决方法。**思路是：先将图片插入ppt中，再将图片占位符的anchor赋值给图片，最后将图片占位符删掉**，没有找到图片占位符里有什么api可以直接填充图片的。

```java
// 整个图片填充需要用到3个对象的参与
// 当前页XSLFSlide, 当前页里的图片占位符XSLFAutoShape，和图片InputStream
private void replaceImage(XSLFSlide slide, XSLFAutoShape imageShape, InputStream image) {
    if (image == null) {
        return;
    }
    try {
        // 先把图片加入到ppt中
        XSLFPictureData pictureData = slide.getSlideShow().addPicture(image, PictureData.PictureType.JPEG);
        XSLFPictureShape pictureShape = slide.createPicture(pictureData);
        // 再设置图片的anchor为占位符的anchor
        pictureShape.setAnchor(imageShape.getAnchor());
    } catch (IOException e) {
        logger.error("fail to create picture in ppt", e);
    }
}
```

#### 填充的顺序

默认图片是纵向优先的，我们一般会横向优先，比如ppt里你又一个 横向2个，纵向3个的图片阵列：如下图

```
1  2
3  4
5  6
```

你填充4张图片时，它是按1，3，5，2纵向优先填充的（不管我咋弄都是纵向的）。所以我们就要对`List<XSLFAutoShape> 图片占位符集合`重新按横向优先排序

```java
// 给图片排序，默认是纵向优先，排序之后是横向优先
// 这个排序要求同一行的图片组件的y值整数位一定要一样
// 第一行有个细节，(int)(img.getAnchor().getY())
// 是因为同一行的图片组件getY()可能只相差了0.00000x
// 所以我只取Y坐标的整数位横向分组
        Map<Integer, List<XSLFAutoShape>> yGroup = imagePanels.stream().collect(Collectors.groupingBy(img -> (int)(img.getAnchor().getY())));
        // 每一横排的排序
        yGroup.forEach((k,v) -> v.sort((o1, o2) -> (int)(o1.getAnchor().getX() - o2.getAnchor().getX())));
        LinkedHashMap<Integer, List<XSLFAutoShape>> ySorted = yGroup.entrySet().stream()
                .sorted(Map.Entry.comparingByKey())
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue,
                        (oldValue, newValue) -> oldValue, LinkedHashMap::new));
        List<XSLFAutoShape> sortedImagePanels = Lists.newArrayList();
        ySorted.forEach((k,v) -> sortedImagePanels.addAll(v));
```

### 复制某一页PPT

这里的坑在于，你复制了一页后，不能直接编辑里面的元素（会报一个元素断开连接的异常，具体名字记不清了）。

#### 首先，如何克隆一页PPT

```java
	// 入参是程序一开始读取的ppt模板
	// 这段代码不一定会百分之百复制完整你的PPT
	// 如果你当前页ppt有些内容是来自于ppt母版的统一控制，比如背景图片啥的，复制出来的ppt可能不会带上背景图，没做过详细测试研究
	// 如果你遇到这种情况，那你搜索的关键词就是 clone ppt slide master
private void cloneProjectDetailTemplate(XMLSlideShow ppt) {
    // 创建一页新ppt
    XSLFSlide newSlide = ppt.createSlide();
    // 获取项目模版页ppt
    XSLFSlide projectDetailSlide = ppt.getSlides().get(2);
    // 将项目模版页的元素复制到新的ppt页里
    XSLFSlideLayout srcLayout = projectDetailSlide.getSlideLayout();
    XSLFSlideLayout newLayout = newSlide.getSlideLayout();
    newLayout.importContent(srcLayout);
    newSlide.importContent(projectDetailSlide);
}
```

#### 如何解决克隆出来的ppt无法编辑

爬坑时，有看到说这是一个bug。变通的方案就是：**先根据你的业务数据，把ppt的页数克隆补全，把这个补全了的ppt保存到磁盘，再读取出来，就可以正常编辑所有页了**

```java
// demandProjectDetails是我的业务数据
// 我需要把ppt的详情页扩充到这个业务数据的size
// 这里有个ppt.setSlideOrder可以帮助你调整ppt页的顺序
private XMLSlideShow initPPTPages(XMLSlideShow ppt, List<PPTDataItem> demandProjectDetails) {

    if (ppt == null) {
        return null;
    }

    // 获取最后的感谢页
    XSLFSlide thankSlide = ppt.getSlides().get(ppt.getSlides().size() - 1);
    // 给每个方案都添加一个对应都ppt模版
    int cloneCount = demandProjectDetails.size() - 1;
    if (cloneCount > 0) {
        for (int i = 0; i<cloneCount; i++) {
            cloneProjectDetailTemplate(ppt);
        }
        // 将感谢页置最后
        ppt.setSlideOrder(thankSlide, ppt.getSlides().size() - 1);
        String path = PPTDownloadServiceImpl.class.getClassLoader().getResource("template").getPath();
        // 重新加载ppt，以解决克隆的ppt无法编辑的问题
        String fullTemplateFilename = "ppt_full_template_" + 3047 + ".pptx";
        boolean saveFlag = save(ppt, path + "/" + fullTemplateFilename);
        if (saveFlag) {
            return loadPPTTemplate(fullTemplateFilename);
        } else {
            return null;
        }
    } else {
        return ppt;
    }
}
```

### PS

apache poi 的文档是真心简陋...用起来全靠自己爬坑。

引入哪些jar包，[官网](http://poi.apache.org/components/index.html)列的很清楚，没必要无脑复制网上博客里列的

|                          Component                           |    Application type     |                       Maven artifactId                       |                            Notes                             |
| :----------------------------------------------------------: | :---------------------: | :----------------------------------------------------------: | :----------------------------------------------------------: |
|  [POIFS](http://poi.apache.org/components/poifs/index.html)  |     OLE2 Filesystem     |                            *poi*                             |        Required to work with OLE2 / POIFS based files        |
|   [HPSF](http://poi.apache.org/components/hpsf/index.html)   |   OLE2 Property Sets    |                            *poi*                             |                                                              |
|    [HSSF](http://poi.apache.org/components/spreadsheet/)     |        Excel XLS        |                            *poi*                             |       For HSSF only, if common SS is needed see below        |
|     [HSLF](http://poi.apache.org/components/slideshow/)      |     PowerPoint PPT      |                       *poi-scratchpad*                       |                                                              |
|      [HWPF](http://poi.apache.org/components/document/)      |        Word DOC         |                       *poi-scratchpad*                       |                                                              |
| [HDGF](http://poi.apache.org/components/diagram/index.html)  |        Visio VSD        |                       *poi-scratchpad*                       |                                                              |
|   [HPBF](http://poi.apache.org/components/hpbf/index.html)   |      Publisher PUB      |                       *poi-scratchpad*                       |                                                              |
|   [HSMF](http://poi.apache.org/components/hsmf/index.html)   |       Outlook MSG       |                       *poi-scratchpad*                       |                                                              |
|                             DDF                              | Escher common drawings  |                            *poi*                             |                                                              |
|                             HWMF                             |      WMF drawings       |                       *poi-scratchpad*                       |                                                              |
| [OpenXML4J](http://poi.apache.org/components/oxml4j/index.html) |          OOXML          | *poi-ooxml* plus either *poi-ooxml-schemas* or *ooxml-schemas* and *ooxml-security* |    See notes below for differences between these options     |
|    [XSSF](http://poi.apache.org/components/spreadsheet/)     |       Excel XLSX        |                         *poi-ooxml*                          |                                                              |
|     [XSLF](http://poi.apache.org/components/slideshow/)      |     PowerPoint PPTX     |                         *poi-ooxml*                          |                                                              |
|      [XWPF](http://poi.apache.org/components/document/)      |        Word DOCX        |                         *poi-ooxml*                          |                                                              |
| [XDGF](http://poi.apache.org/components/diagram/index.html)  |       Visio VSDX        |                         *poi-ooxml*                          |                                                              |
| [Common SL](http://poi.apache.org/components/slideshow/index.html) | PowerPoint PPT and PPTX |               *poi-scratchpad* and *poi-ooxml*               | SL code is in the core POI jar, but implementations are in poi-scratchpad and poi-ooxml. |
|  [Common SS](http://poi.apache.org/components/spreadsheet/)  |   Excel XLS and XLSX    |                         *poi-ooxml*                          | WorkbookFactory and friends all require poi-ooxml, not just core poi |