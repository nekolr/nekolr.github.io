---
title: XML 解析
date: 2017/7/2 12:2:0
tags: [XML]
categories: [XML]
---
在 Java 中解析 XML，主流常用的方法有四种：**DOM、SAX、JDOM** 和 **DOM4J**。		
		
其中，DOM 和 SAX 是两种不同的 XML 解析方式，JDK 已经提供了支持 DOM 方式和 SAX 方式的具体实现，因此可以不引入第三方 jar 包来使用这两种解析方式。		
<!--more-->		
		
JAXP（`Java API for XML Processing`）：JAXP 类库是 JDK 的一部分，它由以下几个包及其子包组成：  

- `org.w3c.dom`  
提供 DOM 方式解析 XML 的标准接口。		
- `org.xml.sax`  
提供 SAX 方式解析 XML 的标准接口。		
- `javax.xml`  
提供 StAX 方式解析 XML 的标准接口和解析 XML 的类。		
		
JDOM 和 DOM4J 是在这两种方式的基础上发展出的 XML 解析类库，看名字就知道这是只能在 Java 平台使用的类库。		
		
**需要注意的是：DOM 和 SAX 是两种解析 XML 的方式，这两种方式与编程语言无关，针对不同的编程语言都有具体的实现。而在 Java 中，针对这两种方式及衍生方式实现的类库有很多，有 JDK 自带的 DOM 解析类库、SAX 解析类库和 StAX 解析类库（和 SAX 类似但有区别），还有第三方的 DOM4J 解析类库和 JDOM 解析类库等。**		
		
## DOM（Document Object Model）		
DOM 是与平台和编程语言无关的 W3C 的标准，基于 DOM 文档树，在使用时需要加载和构建整个文档形成一棵 DOM 树放在内存当中。		
- **优点**  
> 允许应用程序对数据和结构做出更改，即 CRUD 操作。  
> 访问是双向的，可以在任何时候在树中上下导航，获取和操作任意部分的数据。		
		
- **缺点**  
> 需要加载整个 XML 文档来构造层次结构，文档越大，消耗资源越多。		
		
在使用 DOM 的方式解析 XML 之前，需要了解 DOM 中的节点类型：		
		
> W3C XML DOM 节点类型：[http://www.w3school.com.cn/xmldom/dom_nodetype.asp](http://www.w3school.com.cn/xmldom/dom_nodetype.asp)		
		
常用的节点：		

| 节点类型 | 描述 | NodeType | nodeName | nodeValue |
| ------------ | ------------ |	------------ | ------------ | ------------ |
| Document | 表示整个文档（DOM 树的根节点） | 9 | #document | null |
| Element | 表示 element（元素）元素 |	1 | element name | null |
| Attr | 属性名称 |	2 | 属性名称 | 属性值 |
| Text | 表示元素或属性中的文本内容 | 3 | #text | 节点内容 |
| Comment | 表示注释 | 8 | #comment | 注释文本 |

		
DOM 解析过程及常用方法：		
		
![DOM_XML1](https://img.nekolr.com/images/2018/04/14/ROd.png)
![DOM_XML2](https://img.nekolr.com/images/2018/04/14/W0W.png)
		
使用 JAXP 的 DOM 解析方式：		
		
```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!-- this is comment -->
<root>
    <animate id="0">
        <name> 希德尼娅的骑士 </name>
        <published>2014.04</published>
    </animate>
    <animate id="1">
        <name> 恶之华 </name>
        <published>2013.04</published>
    </animate>
    <animate id="2">
        <name> 化物语 </name>
        <published>2009.07</published>
    </animate>
    <animate id="3">
        <name> 穿越时空的少女 </name>
        <published>2006.07</published>
    </animate>
</root>
```

```java
package com.nekolr;

import org.w3c.dom.*;

import javax.xml.parsers.DocumentBuilder;
import javax.xml.parsers.DocumentBuilderFactory;
import java.util.Objects;

/**
 * Created by nekolr on 2017/7/2.
 */
public class DOMResolver {

    public static final String URI = System.getProperty("user.dir") + "/target/classes/nekolr.xml";

    public static void main(String[] args) throws Exception {

        // 1 创建一个 DocumentBuilderFactory 对象
        DocumentBuilderFactory dbf = DocumentBuilderFactory.newInstance();
        // 2 通过 factory 构建一个 DocumentBuilder 对象
        DocumentBuilder db = dbf.newDocumentBuilder();
        // 3 解析 uri 构建 Document
        Document doc = db.parse(URI);

        // 获取 document 的根节点
        Element root = doc.getDocumentElement();
        // 获取 document 的第一个节点，在这个例子里是注释
        Node comment = doc.getFirstChild();

        // 根据 TagName 获取 NodeList
        NodeList nodeList = doc.getElementsByTagName("animate");
        for (int i = 0, len = nodeList.getLength(); i < len; i++) {
            Node node = nodeList.item(i);

            if (node.hasAttributes())
                // 节点的全部属性
                printNodeAttrs(node);

            // 如果没有子节点 打印 Text
            if (!node.hasChildNodes())
                System.out.println("nodeName=" + node.getNodeName() + ", Text=" + node.getTextContent());

            // 节点的所有子节点
            getChildNodes(node);
        }
    }

    /**
     * 打印节点的所有属性
     *
     * @param node
     */
    public static void printNodeAttrs(Node node) {
        NamedNodeMap attrs = node.getAttributes();
        if (!Objects.isNull(attrs)) {
            for (int i = 0; i < attrs.getLength(); i++) {
                Node attr = attrs.item(i);
                String name = attr.getNodeName();// 属性名
                String value = attr.getNodeValue();// 属性值
                System.out.println("nodeName=" + node.getNodeName() + ", attr=" + name + ", value=" + value);
            }
        }
    }

    /**
     * 获取节点的子节点
     *
     * @param node
     */
    public static void getChildNodes(Node node) {
        NodeList childNodes = node.getChildNodes();
        for (int j = 0; j < childNodes.getLength(); j++) {
            Node child = childNodes.item(j);

            if (child.getNodeType() == Node.ELEMENT_NODE) {
                if (child.hasAttributes())
                    // 打印所有属性
                    printNodeAttrs(child);

                if (!child.hasChildNodes())
                    // 节点的 Text
                    System.out.println("nodeName=" + child.getNodeName() + ", Text=" + child.getTextContent());
                else
                    getChildNodes(child);
            } else if (child.getNodeType() == Node.TEXT_NODE && !"".equals(child.getNodeValue().trim())) {
                System.out.println("nodeName=" + node.getNodeName() + ", Text=" + child.getTextContent());
            }
        }
    }
}
```

## SAX（Simple API for XML）
与 DOM 的方式不同，SAX 方式不需要构建整个文档，基于流模型中的推（PUSH）模型，由事件驱动，从文档开始顺序扫描，每发现一个节点就引发一个事件，事件推给事件处理器，通过回调方法完成解析，因此读取文档的过程就是解析文档的过程。使用 SAX 需要两部分：SAX 解析器和事件处理器，其中事件处理器需要开发者提供，一般通过继承默认的事件处理器 DefaultHandler，重写需要的方法即可。		
- **优点**		
> 不需要等待所有数据都被处理，分析就能立即开始。		
> 只在读取数据时检查数据，不需要保存在内存中。		
> 可以在某个条件得到满足时停止解析，不必解析整个文档。		
> 效率和性能较高，能解析大于系统内存的文档。		
		
- **缺点**  
> 需要应用程序自己负责节点的处理逻辑（例如维护父/子关系等），文档越复杂程序就越复杂。
> 单向导航，无法定位文档层次，很难同时访问同一文档的不同部分数据，不支持 XPath。	
> 只能读取文档，不能进行其他操作（新增、修改和删除）。
		
SAX 解析过程及使用：		
		
![SAX_XML1](https://img.nekolr.com/images/2018/04/14/LyR.png)
![SAX_XML2](https://img.nekolr.com/images/2018/04/14/Ay3.png)		
		
使用 JAXP 的 SAX 解析方式（解析的 xml 同上）：
		
```java
package com.nekolr;

import org.xml.sax.Attributes;
import org.xml.sax.SAXException;
import org.xml.sax.helpers.DefaultHandler;

/**
 * Created by nekolr on 2017/7/4.
 */
public class SAXParserHandler extends DefaultHandler {

    boolean isName = false;
    boolean isPublished = false;

    @Override
    public void startDocument() throws SAXException {
        System.out.println("******文档解析开始******");
    }

    @Override
    public void endDocument() throws SAXException {
        System.out.println("******文档解析结束******");
    }

    @Override
    public void startElement(String uri, String localName, String qName, Attributes attributes) throws SAXException {
        if ("animate".equals(qName)) {
            System.out.println("nodeName=" + qName + ", id=" + attributes.getValue("id"));
        } else if ("name".equals(qName)) {
            System.out.println("start element " + qName);
            isName = true;
        } else if ("published".equals(qName)) {
            System.out.println("start element " + qName);
            isPublished = true;
        }
    }

    @Override
    public void endElement(String uri, String localName, String qName) throws SAXException {
        System.out.println("end element " + qName);
    }

	/**
     * ch 代表节点中的所有内容，即每次遇到一个标签调用 characters 方法时，数组 ch 实际就是整个 XML 文档的内容
     * 如何每次去调用 characters 方法时我们都可以获取不同的节点属性？
     * 这时就必须结合 start（开始节点）和 length（长度）
     */
    @Override
    public void characters(char[] ch, int start, int length) throws SAXException {
        if (isName) {
            System.out.println(new String(ch, start, length));
            isName = false;
        } else if (isPublished) {
            System.out.println(new String(ch, start, length));
            isPublished = false;
        }
    }
```

```java
package com.nekolr;

import javax.xml.parsers.SAXParser;
import javax.xml.parsers.SAXParserFactory;

/**
 * Created by nekolr on 2017/7/4.
 */
public class SAXDemo {
    public static final String URI = System.getProperty("user.dir") + "/target/classes/nekolr.xml";

    public static void main(String[] args) throws Exception {
        // 1 获取一个 SAXParserFactory 对象
        SAXParserFactory factory = SAXParserFactory.newInstance();
        // 2 获取一个 SAXParser 解析类对象
        SAXParser parser = factory.newSAXParser();
        // 3 将处理器对象放入解析器中
        parser.parse(URI, new SAXParserHandler());
    }
}
```

## JDOM（Java-based Document Object Model）
为了减少 DOM 和 SAX 的编码量出现了 JDOM，JDOM 致力于成为 Java 特定文档模型，它遵循二八定律，底层仍然使用 DOM 和 SAX（使用 SAX 解析文档），并大量使用了 JDK 的 Collections。与 DOM 类似，JDOM 也将解析的 XML 以 DOM 树的方式放入内存中。		
- **优点**
> 不使用接口，使用具体实现类，简化了 DOM 的 API。		
> 大量使用 Collections，对 Java 开发友好。		
		
- **缺点**
> 由于与 DOM 类似的生成 DOM 树结构，使得 JDOM 在处理大型 XML 文档时性能较差。		
> 灵活性差，不支持 DOM 中某些遍历。
		
JDOM 的使用：		
		
![JDOM_XML1](https://img.nekolr.com/images/2018/04/14/4KA.png)
		
![JDOM_XML2](https://img.nekolr.com/images/2018/04/14/drg.png)
		
使用 JDOM 解析之前的 XML 文档，因为是第三方类库，需要引入 jar 包：		
```xml
  <dependencies>
    
    <!-- jdom2 -->
    <dependency>
      <groupId>org.jdom</groupId>
      <artifactId>jdom2</artifactId>
      <version>2.0.6</version>
    </dependency>

</dependencies>
```

```java
package com.nekolr;

import org.jdom2.Attribute;
import org.jdom2.Document;
import org.jdom2.Element;
import org.jdom2.input.SAXBuilder;
import java.io.FileInputStream;
import java.io.InputStreamReader;
import java.util.List;

/**
 * Created by nekolr on 2017/7/4.
 */
public class JDOMDemo {
    public static final String URI = System.getProperty("user.dir") + "/target/classes/nekolr.xml";

    public static void main(String[] args) throws Exception {
        // 1 构建 SAXBuilder 对象
        SAXBuilder saxBuilder = new SAXBuilder();
        // 2 使用包装流读取文件时指定编码，防止乱码
        InputStreamReader in = new InputStreamReader(new FileInputStream(URI), "utf-8");
        // 3 构建 Document
        Document doc = saxBuilder.build(in);


        //获取根节点
        Element root = doc.getRootElement();
        //获取所有子节点
        List<Element> animates = root.getChildren();
        for (Element animate : animates) {
            Attribute id = animate.getAttribute("id");
            System.out.println("nodeName=" + animate.getName() + " ,id=" + id.getValue());

            List<Element> children = animate.getChildren();
            for (Element child : children) {
                System.out.println("nodeName=" + child.getName() + ", Text=" + child.getText());
            }

        }
    }
}
```

## DOM4J（Document Object Model for Java）
DOM4J 大量使用 JDK 的 Collections，同时也提供了一些高性能的替代方法。支持 DOM、SAX 和 JAXP，支持 XPath。使用接口和抽象类，牺牲了部分 API 的简易性来获得更高的灵活性，同时在解析大型文档时内存占用较低。具有性能优异、功能强大、灵活性高、易用性强等优点，被现在很多开源项目使用。		
		
使用 DOM4J 解析之前的 XML 文档，引入第三方 jar：
		
```xml
<dependencies>

    <!-- dom4j -->
    <dependency>
      <groupId>dom4j</groupId>
      <artifactId>dom4j</artifactId>
      <version>1.6.1</version>
    </dependency>

</dependencies>
```

```java
package com.nekolr;

import org.dom4j.Attribute;
import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

import java.util.Iterator;

/**
 * Created by nekolr on 2017/7/4.
 */
public class DOM4JDemo {
    public static final String URI = System.getProperty("user.dir") + "/target/classes/nekolr.xml";

    public static void main(String[] args) throws Exception {
        // 1 构建 SAXReader 对象
        SAXReader saxReader = new SAXReader();
        // 2 读取 XML 文档
        Document doc = saxReader.read(URI);

        // 获取根节点
        Element root = doc.getRootElement();

        for (Iterator iterator = root.elementIterator(); iterator.hasNext(); ) {
            Element animate = (Element) iterator.next();
            Attribute id = animate.attribute("id");
            System.out.println("nodeName=" + animate.getName() + " ,id=" + id.getValue());
            for (Iterator iter = animate.elementIterator(); iter.hasNext(); ) {
                Element ele = (Element) iter.next();
                System.out.println("nodeName=" + ele.getName() + ", Text=" + ele.getText());
            }
        }
    }
}
```
		
> 本文部分图片引用自：<http://www.cnblogs.com/Qian123/p/5231303.html>