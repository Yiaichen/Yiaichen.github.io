---
title: Java读取xml——dom4j篇
date: 2017-08-27 10:33:58
tags: [Java,xml]
categories: Java
---

首先引入dom4j的jar

``` java
package com.hucheng.xmlParse;

import java.io.File;
import java.util.ArrayList;
import java.util.Iterator;
import java.util.List;

import org.dom4j.Attribute;
import org.dom4j.Document;
import org.dom4j.Element;
import org.dom4j.io.SAXReader;

public class XmlUtils
{
    
    public static void main(String[] args)
    {
        long lasting = System.currentTimeMillis();
        try
        {
            //获取xml文件
            File f = new File("data.xml");
            SAXReader reader = new SAXReader();
            Document doc = reader.read(f);
            //获取到根节点
            Element node = doc.getRootElement();
            //判断跟节点下面是否还有子节点
            if (node.elementIterator().hasNext())
            {
                Element next = (Element)node.elementIterator().next();
            }
            Iterator<Element> iterator = node.elementIterator();
            //开始循环子节点
            while(iterator.hasNext()){  
                Element e = iterator.next();  
                listNodes(e);
            }  
        }
        catch (Exception e)
        {
            e.printStackTrace();
        }
        System.out.println("运行时间：" + (System.currentTimeMillis() - lasting) + " 毫秒");
    }
    
    /**
     * @描述：循环子节点
     * @作者：chen
     * @时间：2017年8月17日 上午10:53:06
     * @param node
     */
    protected static void listNodes(Element node)
    {
        //首先获取当前节点的所有属性节点  
        List<Attribute> list = node.attributes();  
        //遍历属性节点  
        for(Attribute attribute : list){  
            System.out.println("属性"+attribute.getName() +":" + attribute.getValue());  
        }  
        //如果当前节点内容不为空，则输出  
        if(!(node.getTextTrim().equals(""))){
             System.out.println( node.getName() + "：" + node.getText());
        }  
        
        //同时迭代当前节点下面的所有子节点  
        //使用递归  
        Iterator<Element> iterator = node.elementIterator();  
        while(iterator.hasNext()){  
            Element e = iterator.next();  
            listNodes(e);  
        }  
    }
}

```

下面是 `data.xml`
``` xml
<?xml version="1.0" encoding="UTF-8"?>
<VALUES>
<VALUE id = "1" name = "第一个">
<NO>鄂B74110</NO>
<ADDR>湖北省黄石市白沙镇</ADDR>
</VALUE>
<VALUE id = "2" name = "第二个">
<NO>鄂B74111</NO>
<ADDR>湖北省黄石市白沙镇白沙老街</ADDR>
</VALUE>
<VALUE id = "3" name = "第三个">
<NO>鄂B74112</NO>
<ADDR>湖北省黄石市白沙镇白沙中学</ADDR>
</VALUE>
</VALUES>
```

控制台会输出：
``` cmd
属性id:1
属性name:第一个
NO：鄂B74110
ADDR：湖北省黄石市白沙镇
属性id:2
属性name:第二个
NO：鄂B74111
ADDR：湖北省黄石市白沙镇白沙老街
属性id:3
属性name:第三个
NO：鄂B74112
ADDR：湖北省黄石市白沙镇白沙中学
运行时间：71 毫秒
```
