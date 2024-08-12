---
title: "使用 Java 把 RGB 转为 CMYK"
date: 2024-08-01T19:58:03+08:00
# weight: 1
# aliases: ["/tech"]
tags: ["tech", "java"]
author: ["Spring"]
showToc: true
TocOpen: false
draft: false
hidemeta: false
comments: false
description: "使用 JAI 把 RGB 转为 CMYK."
canonicalURL: "https://canonical.url/to/page"
disableHLJS: false # to disable highlightjs
disableShare: false
hideSummary: false
searchHidden: false
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
ShowRssButtonInSectionTermList: true
UseHugoToc: true
editPost: # github 当前文章修改/建议
    URL: "https://github.com/heyzqq/heyzqq-content.github.io/tree/main/content"
    Text: "Suggest Changes"
    appendFilePath: true # to append file path to Edit link
---

## 00 前言

最近接到了一个需求，要把 RGB 的图片转为 CMYK 的颜色模式，用于印刷。

网上对于 RGB 转为 CMYK 的资料较少，Java 也没有内置的方法。主流方法有两种，一种是手搓强转，另一种则是借助 ICC 文件来转换。

该需求对于分辨率、物理尺寸有一定的要求，且原途径就是通过 PS 处理的，正好用同样的 ICC 文件转换，能达到相对一致的颜色要求。

## 01 色彩空间转换

### 1.1 关键概念

**RGB 模式**是以色光三原色为基础建立的色彩模式，**红、绿、蓝**，是电脑、手机、投影仪、电视等屏幕显示的最佳颜色模式。

**CMYK**（青色、品红色、黄色、黑色）是印刷材料的色彩空间。CMY 是 3 种印刷油墨名称的首字母：**青色 Cyan、洋红色 Magenta、黄色 Yellow**，由于目前制造工艺还不能造出高纯度的油墨，CMY 相加实际得到的是深灰色或深褐色，故加入纯黑色。

**ICC Profile 色彩特性文件**，是一组用来描述色彩空间的特性的数据集合，因由国际色彩联盟（ICC）主持制定其规范而得名。RGB 转为 CMYK 就是把 RGB 值转为 CMYK 值。

**像素**（Pixel）是指构成图片的小色点。
**分辨率**（单位 DPI）是指每英寸（Inch）上的**像素**数量，++1 英寸 = 2.54 厘米++。
像素相同时，分辨率越高则像素**密度**越大，实际打印尺寸越小，图像也越细腻。

### 1.2 转换步骤

首先，引入 JAI（Java Advanced Imaging），最新版 1.1.3，这个库好几年没更新了：

```XML
<dependency>
    <groupId>javax.media.jai</groupId>
    <artifactId>com.springsource.javax.media.jai.codec</artifactId>
    <version>1.1.3</version>
</dependency>
<!-- 这个库只用到了 TIFF 的静态变量，可以不引入 -->
<dependency>
    <groupId>com.github.jai-imageio</groupId>
    <artifactId>jai-imageio-core</artifactId>
    <version>1.4.0</version>
</dependency>
```

然后，加载 ICC 文件，这里选择 `JapanColor2001Coated.icc`（按需选择即可），安装 PS 就有，在 `C:\Program Files (x86)\Common Files\Adobe\Color\Profiles\Recommended` 目录下，路径不一样找不到的搜一下就可以。

```JAVA
private static void readICCProfile() throws IOException {
    if (cmykColorSpace == null) {
        synchronized (CMYKUtil.class) {
            if (cmykColorSpace == null) {
                try (InputStream inputStream = CMYKUtil.class.getClassLoader().getResourceAsStream("icc/JapanColor2001Coated.icc")) {
                    ICC_Profile cmykProfile = ICC_Profile.getInstance(inputStream);  // 如果是读取外部文件，把文件的绝对路径传入即可
                    cmykColorSpace = new ICC_ColorSpace(cmykProfile);
                }
            }
        }
    }
}
```

需要注意的是，这里我是从 `resources` 直接加载的配置文件，需要额外配置 Maven，否则编译时 ICC 文件会被 Maven 修改，报错 `Invalid ICC Profile Data`：

```XML
<!-- pom.xml -->
<build>
    <resources>
        <!-- 保留其他文件的处理方式 -->
        <resource>
            <directory>src/main/resources</directory>
            <filtering>true</filtering>
            <includes>
                <include>**/*.xml</include>
                <include>**/*.properties</include>
                <include>**/*.y*ml</include>
                <include>banner.txt</include>
            </includes>
        </resource>
        <!-- 禁止编译时修改 ICC 文件 -->
        <resource>
            <directory>src/main/resources</directory>
            <!-- 默认会替换文件中的占位符属性，导致文件被修改 -->
            <filtering>false</filtering>
            <includes>
                <include>**/*.icc</include>
            </includes>
        </resource>
    </resources>
    
    <!-- ... -->
</build>
```

如果不想修改项目配置，只能通过读取外部文件，把 ICC 文件放到指定目录去读取即可。

然后, 准备需要的编码参数，用来调整图片：

```java
private static TIFFEncodeParam prepareEncodeParams(int dpi) {
    TIFFEncodeParam params = new TIFFEncodeParam();

    // 加点压缩, 不然图片太大了
    // params.setCompression(TIFFEncodeParam.COMPRESSION_LZW); // 不支持
    params.setCompression(TIFFEncodeParam.COMPRESSION_DEFLATE);

    // 分辨率 DPI
    // doc: https://download.java.net/media/jai-imageio/javadoc/1.1/constant-values.html
    params.setExtraFields(new TIFFField[]{
            // DIP，x y 统一，不统一图片会被拉伸或压缩
            new TIFFField(BaselineTIFFTagSet.TAG_X_RESOLUTION,               // 282
                    TIFFField.TIFF_RATIONAL, 1, new long[][]{{dpi, 1}}),
            new TIFFField(BaselineTIFFTagSet.TAG_Y_RESOLUTION,               // 283
                    TIFFField.TIFF_RATIONAL, 1, new long[][]{{dpi, 1}}),
            // 分辨率单位 2 (inch)
            new TIFFField(BaselineTIFFTagSet.TAG_RESOLUTION_UNIT,            // 296
                    TIFFField.TIFF_SHORT, 1,
                    new char[]{BaselineTIFFTagSet.RESOLUTION_UNIT_INCH}), 
            // 位深度 8*4 (无效)
            // new TIFFField(BaselineTIFFTagSet.TAG_BITS_PER_SAMPLE,
            //         TIFFField.TIFF_SHORT, 1, new char[]{8})
    });

    return params;
}
```

* 由于需求是要打印固定的尺寸的图片，但是图片的分辨率不固定，所以每次都要计算 DPI。
* 其他特殊需求, 需要自己去查去摸索了，参考 `BaselineTIFFTagSet` 内的代码，内部有些许说明。

到这里，可以封装一个转换方法了：

```JAVA
public static ByteArrayOutputStream rgb2Cmyk(BufferedImage rgbImage, int dpi)
        throws IOException {

    // 加载 ICC 配置文件
    readICCProfile();
    // 准备编码的各种参数
    TIFFEncodeParam params = prepareEncodeParams(dpi);

    // 创建颜色转换实体：从源色彩空间转到 cmyk 色彩空间
    ColorConvertOp op = new ColorConvertOp(
                                rgbImage.getColorModel().getColorSpace(),
                                cmykColorSpace,
                                null);

    // 转换, 第二个是目标图像源, 为空则自动创建合适的
    BufferedImage cmykImage = op.filter(rgbImage, null);

    // 用指定的参数转换为 CMYK, 放到 ByteArrayOutputStream 只是为了返回给其他地方使用
    // 如果不需要可以直接写入文件等即可
    ByteArrayOutputStream baoStream = new ByteArrayOutputStream();
    ImageEncoder encoder2 = ImageCodec.createImageEncoder("TIFF",
                                baoStream,
                                params);
    // 编码为 TIFF
    encoder2.encode(cmykImage);
    return baoStream;
}
```

最后，可以把转换好的图片写入文件（或者其他具体操作）：

```JAVA
public static void writeImage(ByteArrayOutputStream outputStream, String filepath) {
    try (FileImageOutputStream fo = new FileImageOutputStream(new File(filepath))) {
        fo.write(outputStream.toByteArray());
    } catch (IOException e) {
        log.error("写入文件失败, {}, {}", filepath, e.getMessage(), e);
    }
}
```

封装好后，使用步骤很简单了：

```JAVA
@Test
public void test_rgb2Cmyk() throws IOException {
    // 读取图片
    BufferedImage rgbImage = ImageIO.read(new File("D:\\image\\rgb.png"));
    // 计算 DPI, 取宽/高最大值, 如果取小的那边偏差较多
    // 分辨率越高偏差越大, 固定 DPI 或不在乎物理尺寸的可以忽略
    int dpi = CMYKUtil.getDpi(rgbImage.getWidth(), 100);  // 像素, 厘米
    ByteArrayOutputStream bs = CMYKUtil.rgb2Cmyk(rgbImage, dpi);
    CMYKUtil.writeImage(bs, "D:\\image\\cmyk.tif");
}
```

## 03 总结

要把 RGB 图片转为 CMYK 等其他色彩空间的图片，使用 ICC 是比较方便的。但是 JAI 许久未更新了，有些功能还不支持，好几个参数试了没效果，如果有其他更详细的需求，未必能够用它来实现。

总结一下转换的步骤：

1. 引入 JAI 依赖，读取 ICC 配置文件（把 ICC 放到项目 Resources 下的需要注意过滤）。
2. 为转换准备相应的参数。
3. 初始化源色彩空间（ColorConvertOp op），转换原图为目标格式（op.filter）。
4. 使用准备的参数编码转换后的图像。
5. 对转换好多图片进行自定义处理。

注意，同时压缩太多图片的话，建议压缩（优先使用 TIFFEncodeParam 配置），否则很容易内存不足。

## 完整代码

```JAVA
package fun.springx.image.utils;

import cn.hutool.core.util.StrUtil;
import com.github.jaiimageio.plugins.tiff.BaselineTIFFTagSet;
import com.sun.media.jai.codec.ImageCodec;
import com.sun.media.jai.codec.ImageEncoder;
import com.sun.media.jai.codec.TIFFEncodeParam;
import com.sun.media.jai.codec.TIFFField;
import lombok.extern.slf4j.Slf4j;

import javax.imageio.stream.FileImageOutputStream;
import java.awt.color.ColorSpace;
import java.awt.color.ICC_ColorSpace;
import java.awt.color.ICC_Profile;
import java.awt.image.BufferedImage;
import java.awt.image.ColorConvertOp;
import java.io.*;

/**
 * CMYK 工具类
 *
 * @author Spring
 * @since 2024-07-25
 */
@Slf4j
public class CMYKUtil {
    private static final double INCH_2_CM = 2.54d;

    static volatile ColorSpace cmykColorSpace = null;

    private static void readICCProfile() throws IOException {
        if (cmykColorSpace == null) {
            synchronized (CMYKUtil.class) {
                if (cmykColorSpace == null) {
                    try (InputStream inputStream = CMYKUtil.class.getClassLoader().getResourceAsStream("icc/JapanColor2001Coated.icc")) {
                        ICC_Profile cmykProfile = ICC_Profile.getInstance(inputStream);
                        cmykColorSpace = new ICC_ColorSpace(cmykProfile);
                    }
                }
            }
        }
    }

    private static TIFFEncodeParam prepareEncodeParams(int dpi) {
        TIFFEncodeParam params = new TIFFEncodeParam();
    
        // 加点压缩, 不然图片太大了
        // params.setCompression(TIFFEncodeParam.COMPRESSION_LZW); // 不支持
        params.setCompression(TIFFEncodeParam.COMPRESSION_DEFLATE);
    
        // 分辨率 DPI
        // doc: https://download.java.net/media/jai-imageio/javadoc/1.1/constant-values.html
        params.setExtraFields(new TIFFField[]{
                new TIFFField(BaselineTIFFTagSet.TAG_X_RESOLUTION,               // 282
                        TIFFField.TIFF_RATIONAL, 1, new long[][]{{dpi, 1}}),
                new TIFFField(BaselineTIFFTagSet.TAG_Y_RESOLUTION,               // 283
                        TIFFField.TIFF_RATIONAL, 1, new long[][]{{dpi, 1}}),
                new TIFFField(BaselineTIFFTagSet.TAG_RESOLUTION_UNIT,            // 296
                        TIFFField.TIFF_SHORT, 1,
                        new char[]{BaselineTIFFTagSet.RESOLUTION_UNIT_INCH}), // 分辨率单位 2 (inch)
                // new TIFFField(BaselineTIFFTagSet.TAG_BITS_PER_SAMPLE,
                //         TIFFField.TIFF_SHORT, 1, new char[]{8}) // 位深度 8*4 (无效)
        });
    
        return params;
    }

    public static ByteArrayOutputStream rgb2Cmyk(BufferedImage rgbImage, int dpi)
            throws IOException {
    
        // 加载 ICC 配置文件
        readICCProfile();
        // 准备编码的各种参数
        TIFFEncodeParam params = prepareEncodeParams(dpi);
    
        // 创建颜色转换实体：从源色彩空间转到 cmyk 色彩空间
        ColorConvertOp op = new ColorConvertOp(
                                    rgbImage.getColorModel().getColorSpace(),
                                    cmykColorSpace,
                                    null);
    
        // 转换, 第二个是目标图像源, 为空则自动创建合适的
        BufferedImage cmykImage = op.filter(rgbImage, null);
    
        // 用指定的参数转换为 CMYK, 放到 ByteArrayOutputStream 只是为了返回给其他地方使用
        // 如果不需要可以直接写入文件等即可
        ByteArrayOutputStream baoStream = new ByteArrayOutputStream();
        ImageEncoder encoder2 = ImageCodec.createImageEncoder("TIFF",
                                    baoStream,
                                    params);
        // 编码为 TIFF
        encoder2.encode(cmykImage);
        return baoStream;
    }

    public static int getDpi(int pixel, int cm) {
        double d = pixel / (cm / INCH_2_CM);
        log.info("Pixel={}, cm={}, DPI={}", pixel, cm, d);
        // 四舍五入偏差比较小
        return Double.valueOf(Math.round(d)).intValue();
    }

    public static void writeImage(ByteArrayOutputStream outputStream, String filepath) {
        try (FileImageOutputStream fo = new FileImageOutputStream(new File(filepath))) {
            fo.write(outputStream.toByteArray());
        } catch (IOException e) {
            log.error("写入文件失败, {}, {}", filepath, e.getMessage(), e);
        }
    }
}
```

## References

[1] 艺海拾贝Design. RGB 和 CMYK 色彩模式的区别与用途. https://www.shejidaren.com/rgb-cmyk-qu-bie-yu-yong-tu.html.  
[2] SMILE嘻嘻. Maven 编译后资源文件发生改变问题. https://blog.csdn.net/zt_16KK/article/details/72459160, 2017-05-18.  
[3] whyMyHelloWorld. java 多张jpg合成tif后避免tif文件过大的方法. https://blog.csdn.net/sinat_29048381/article/details/80006951, 2018-04-19.  

