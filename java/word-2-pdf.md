# 使用java将word转化为pdf
这两天公司需要将word文档转为pdf，在网上找了一圈之后也没有一个好的解决方案。看来需要自己实现该功能了。

由于之前解析doc文件时使用poi，想着应该poi这么强大的工具包会支持格式转换，还是仔细看看官方相关文档吧。果然不出所料，只不过poi不直接支持doc到pdf文件的转换，需要用到apache的另外一个工具包——fop。难道想这样来支持自家的项目？或者说由于自家项目有该功能，就不需要集成到POI中了？真是搞不懂。

[Apache POI - HWPF and XWPF - Java API to Handle Microsoft Word Files](http://poi.apache.org/document/index.html)中的An overview of the code一段有对该功能的明确描述
> Source code in the org.apache.poi.hwpf.extractor tree is a wrapper of this to facilitate easy extraction of interesting things (eg the Text), and org.apache.poi.hwpf.converter package contains Word-to-HTML and Word-to-FO converters (* latest can be used to generate PDF from Word files when using with Apache FOP *)

从FOP官网[The Apache™ FOP Project](http://xmlgraphics.apache.org/fop/)上可以看到FOP的简介，功能很强大的感觉。
> Apache™ FOP (Formatting Objects Processor) is a print formatter driven by XSL formatting objects (XSL-FO) and an output independent formatter. It is a Java application that reads a formatting object (FO) tree and renders the resulting pages to a specified output. Output formats currently supported include PDF, PS, PCL, AFP, XML (area tree representation), Print, AWT and PNG, and to a lesser extent, RTF and TXT. The primary output target is PDF.

既然这种变通的方式可以做，那应该不会有错吧。

不过在使用的时候却走了很多弯路。主要是FOP 2.1版本的`FopFactory.newInstance`方法需要一个`fopConf`文件的参数，但哪里来的`fopConf`文件呢？官网给出的例子也没有说这个`fopCOnf`文件是什么格式，做什么用的，感觉真是有点迷惑。

官网示例参见[Apache™ FOP: Embedding](http://xmlgraphics.apache.org/fop/2.1/embedding.html)

最终在下载的fop-2.1-bin.zip包包含的例子中找到了官方给出的范例，如下所示
具体类路径fop-2.1\examples\embedding\java\embedding\ExampleFO2PDF.java

重点是下面这句代码。这代码，真是坑啊！弄了半天，fopConf还是不知道干嘛用的
> **private final FopFactory fopFactory = FopFactory.newInstance(new File(".").toURI());**

```java
/*
 * Licensed to the Apache Software Foundation (ASF) under one or more
 * contributor license agreements.  See the NOTICE file distributed with
 * this work for additional information regarding copyright ownership.
 * The ASF licenses this file to You under the Apache License, Version 2.0
 * (the "License"); you may not use this file except in compliance with
 * the License.  You may obtain a copy of the License at
 *
 *      http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */

/* $Id: ExampleFO2PDF.java 1356646 2012-07-03 09:46:41Z mehdi $ */

package embedding;

// Java
import java.io.BufferedOutputStream;
import java.io.File;
import java.io.FileOutputStream;
import java.io.IOException;
import java.io.OutputStream;

import javax.xml.transform.Result;
import javax.xml.transform.Source;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerFactory;
import javax.xml.transform.sax.SAXResult;
import javax.xml.transform.stream.StreamSource;

import org.apache.fop.apps.FOPException;
import org.apache.fop.apps.FOUserAgent;
import org.apache.fop.apps.Fop;
import org.apache.fop.apps.FopFactory;
import org.apache.fop.apps.FormattingResults;
import org.apache.fop.apps.MimeConstants;
import org.apache.fop.apps.PageSequenceResults;

/**
 * This class demonstrates the conversion of an FO file to PDF using FOP.
 */
public class ExampleFO2PDF {

    // configure fopFactory as desired
    private final FopFactory fopFactory = FopFactory.newInstance(new File(".").toURI());

    /**
     * Converts an FO file to a PDF file using FOP
     * @param fo the FO file
     * @param pdf the target PDF file
     * @throws IOException In case of an I/O problem
     * @throws FOPException In case of a FOP problem
     */
    public void convertFO2PDF(File fo, File pdf) throws IOException, FOPException {

        OutputStream out = null;

        try {
            FOUserAgent foUserAgent = fopFactory.newFOUserAgent();
            // configure foUserAgent as desired

            // Setup output stream.  Note: Using BufferedOutputStream
            // for performance reasons (helpful with FileOutputStreams).
            out = new FileOutputStream(pdf);
            out = new BufferedOutputStream(out);

            // Construct fop with desired output format
            Fop fop = fopFactory.newFop(MimeConstants.MIME_PDF, foUserAgent, out);

            // Setup JAXP using identity transformer
            TransformerFactory factory = TransformerFactory.newInstance();
            Transformer transformer = factory.newTransformer(); // identity transformer

            // Setup input stream
            Source src = new StreamSource(fo);

            // Resulting SAX events (the generated FO) must be piped through to FOP
            Result res = new SAXResult(fop.getDefaultHandler());

            // Start XSLT transformation and FOP processing
            transformer.transform(src, res);

            // Result processing
            FormattingResults foResults = fop.getResults();
            java.util.List pageSequences = foResults.getPageSequences();
            for (java.util.Iterator it = pageSequences.iterator(); it.hasNext();) {
                PageSequenceResults pageSequenceResults = (PageSequenceResults)it.next();
                System.out.println("PageSequence "
                        + (String.valueOf(pageSequenceResults.getID()).length() > 0
                                ? pageSequenceResults.getID() : "<no id>")
                        + " generated " + pageSequenceResults.getPageCount() + " pages.");
            }
            System.out.println("Generated " + foResults.getPageCount() + " pages in total.");

        } catch (Exception e) {
            e.printStackTrace(System.err);
            System.exit(-1);
        } finally {
            out.close();
        }
    }


    /**
     * Main method.
     * @param args command-line arguments
     */
    public static void main(String[] args) {
        try {
            System.out.println("FOP ExampleFO2PDF\n");
            System.out.println("Preparing...");

            //Setup directories
            File baseDir = new File(".");
            File outDir = new File(baseDir, "out");
            outDir.mkdirs();

            //Setup input and output files
            File fofile = new File(baseDir, "xml/fo/helloworld.fo");
            //File fofile = new File(baseDir, "../fo/pagination/franklin_2pageseqs.fo");
            File pdffile = new File(outDir, "ResultFO2PDF.pdf");

            System.out.println("Input: XSL-FO (" + fofile + ")");
            System.out.println("Output: PDF (" + pdffile + ")");
            System.out.println();
            System.out.println("Transforming...");

            ExampleFO2PDF app = new ExampleFO2PDF();
            app.convertFO2PDF(fofile, pdffile);

            System.out.println("Success!");
        } catch (Exception e) {
            e.printStackTrace(System.err);
            System.exit(-1);
        }
    }
}
```

最终代码如下：
```java
package word2pdf;

import java.io.BufferedInputStream;
import java.io.BufferedOutputStream;
import java.io.File;
import java.io.FileInputStream;
import java.io.FileOutputStream;
import java.io.StringReader;
import java.io.StringWriter;

import javax.xml.transform.OutputKeys;
import javax.xml.transform.Transformer;
import javax.xml.transform.TransformerFactory;
import javax.xml.transform.dom.DOMSource;
import javax.xml.transform.sax.SAXResult;
import javax.xml.transform.stream.StreamResult;
import javax.xml.transform.stream.StreamSource;

import org.apache.fop.apps.Fop;
import org.apache.fop.apps.FopFactory;
import org.apache.fop.apps.FormattingResults;
import org.apache.fop.apps.MimeConstants;
import org.apache.poi.hwpf.HWPFDocument;
import org.apache.poi.hwpf.converter.WordToFoConverter;
import org.apache.poi.util.XMLHelper;

public class Main {

	public static void main(String[] args) throws Exception {
		BufferedInputStream bis = new BufferedInputStream(new FileInputStream("d:\\work\\test1.doc"));
		HWPFDocument hwpfDocument = new HWPFDocument(bis);
		WordToFoConverter wordToFoConverter = new WordToFoConverter(
				XMLHelper.getDocumentBuilderFactory().newDocumentBuilder().newDocument());
		wordToFoConverter.processDocument(hwpfDocument);

		StringWriter stringWriter = new StringWriter();

        Transformer transformer = TransformerFactory.newInstance()
                .newTransformer();
        transformer.setOutputProperty( OutputKeys.INDENT, "yes" );
        transformer.transform(
                new DOMSource( wordToFoConverter.getDocument() ),
                new StreamResult( stringWriter ) );

        FopFactory fopFactory = FopFactory.newInstance(new File(".").toURI());
        BufferedOutputStream bos = new BufferedOutputStream(
        		new FileOutputStream(new File("D:\\work\\test.pdf")));
        Fop fop = fopFactory.newFop(MimeConstants.MIME_PDF, bos);
        transformer = TransformerFactory.newInstance()
                .newTransformer();
        transformer.transform(
        		new StreamSource(new StringReader(stringWriter.toString())),
        		new SAXResult(fop.getDefaultHandler()));
        FormattingResults results = fop.getResults();
//        List<PageSequenceResults> pageSequences = results.getPageSequences();
//        for (PageSequenceResults pageSequenceResults : pageSequences) {
//        	System.out.println("PageSequence "
//	        	+ (String.valueOf(pageSequenceResults.getID()).length() > 0
//	        	        ? pageSequenceResults.getID() : "<no id>")
//	        	+ " generated " + pageSequenceResults.getPageCount() + " pages.");
//		}
        System.out.println("Generated " + results.getPageCount() + " pages in total.");
        bos.close();
	}

}
```

其实这只能说是一个例子，只是实现了最基础功能的代码，但测试却发现一般情况下转换都不会出现乱码。
后续需要补充的功能:
- [ ] 如果遇到word中的字体在ptf中不支持，则会出现乱码，需要找到一种方法设置pdf的默认字体，或者在生成fo格式时将所有中文字体替换成宋体或者别的字体。
- [ ] 由于word功能强大，测试发现有些功能由于在转换成fo格式时都无法输出，所以生成pdf时也不会展示。比如使用域代码展示的效果，最普遍的应用就是带圈文字。
