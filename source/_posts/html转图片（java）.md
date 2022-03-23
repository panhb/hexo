---
title: html转图片（java）
date: 2022-03-23 12:00:00
tags: [java]
---

html转图片，java没有好的现成的免费库能直接把html转成图片，itext有收费的库可以实现，我的思路是基于itext将html转成pdf，再使用pdfbox转成图片达到效果。

<!-- more -->

pom依赖      
```java       
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>itext7-core</artifactId>
    <version>7.1.12</version>
    <type>pom</type>
</dependency>
<dependency>
    <groupId>com.itextpdf</groupId>
    <artifactId>html2pdf</artifactId>
    <version>2.1.5</version>
</dependency>
<dependency>
    <groupId>org.apache.pdfbox</groupId>
    <artifactId>pdfbox</artifactId>
    <version>2.0.25</version>
</dependency>
```        

代码如下
```java       
import cn.hutool.core.io.IoUtil;
import com.google.common.base.Charsets;
import com.itextpdf.html2pdf.ConverterProperties;
import com.itextpdf.html2pdf.HtmlConverter;
import com.itextpdf.html2pdf.resolver.font.DefaultFontProvider;
import com.itextpdf.io.font.FontProgram;
import com.itextpdf.io.font.FontProgramFactory;
import com.itextpdf.kernel.geom.PageSize;
import com.itextpdf.kernel.pdf.PdfDocument;
import com.itextpdf.kernel.pdf.PdfWriter;
import com.itextpdf.layout.Document;
import com.itextpdf.layout.element.BlockElement;
import com.itextpdf.layout.element.IElement;
import com.itextpdf.layout.font.FontProvider;
import lombok.Cleanup;
import lombok.SneakyThrows;
import org.apache.pdfbox.pdmodel.PDDocument;
import org.apache.pdfbox.rendering.ImageType;
import org.apache.pdfbox.rendering.PDFRenderer;

import javax.imageio.ImageIO;
import java.awt.image.BufferedImage;
import java.io.ByteArrayOutputStream;
import java.util.List;

/**
 * @author hongbo.pan
 * @date 2022/3/4
 */
public class ConvertImage {

    @SneakyThrows
    public static byte[] convertToImage(String html) {
        @Cleanup ByteArrayOutputStream pdfStream = new ByteArrayOutputStream();
        @Cleanup ByteArrayOutputStream imgStream = new ByteArrayOutputStream();
        PdfWriter pdfWriter = new PdfWriter(pdfStream);
        PdfDocument pdfDocument = new PdfDocument(pdfWriter);
        ConverterProperties props =  new ConverterProperties();
        FontProvider fontProvider = new DefaultFontProvider();
        //目前只适配微软雅黑粗体（html里面只使用了雅黑，如有其他需要加载对应字体）
        FontProgram msyhbd = FontProgramFactory.createFont(IoUtil.readBytes(ConvertImage.class.getResourceAsStream("/msyhbd.ttc")), 0, true);
        fontProvider.addFont(msyhbd);
        props.setFontProvider(fontProvider);
        List<IElement> iElements = HtmlConverter.convertToElements(html, props);
        //图片比例需要按照a4比例进行调整
        PageSize pageSize = new PageSize(595.0F, 790.3F);
        Document document = new Document(pdfDocument, pageSize, true);
        //去除边界
        document.setMargins(0F, 0F, 0F, 0F);
        for (IElement iElement : iElements) {
            BlockElement blockElement = (BlockElement) iElement;
            blockElement.setMargins(0F, 0F, 0F, 0F);
            document.add(blockElement);
        }
        document.close();
        PDDocument pd = PDDocument.load(pdfStream.toByteArray());
        PDFRenderer pdfRenderer = new PDFRenderer(pd);
        BufferedImage bufferedImage = pdfRenderer.renderImageWithDPI(0, 120, ImageType.RGB);
        ImageIO.write(bufferedImage, "jpg", imgStream);
        pd.close();
        return imgStream.toByteArray();
    }

    public static void main(String[] args) {
        //jdk版本过低需要设置该属性
        System.setProperty("sun.java2d.cmm", "sun.java2d.cmm.kcms.KcmsServiceProvider");
        convertToImage(IoUtil.read(ConvertImage.class.getResourceAsStream("/haha.html"), Charsets.UTF_8);
    }

}
```        