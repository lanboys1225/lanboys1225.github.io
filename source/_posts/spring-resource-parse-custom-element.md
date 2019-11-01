---
title: Spring容器加载过程源码解析之自定义标签解析
date: 2019-11-01 14:03:12
category: spring
tags: [spring]
---

>在上文 [Spring容器加载过程源码解析之默认标签解析](https://me.oopcoder.cn/2019/10/27/spring-resource-parse-default-element/index.html) 中，我们学习了默认标签的解析，今天我们来看看自定义的标签和属性是如何解析的。


> 提醒：本文是基于Spring 3.0.0.RELEASE 版本进行讲解的，其他版本可能稍有差异，在贴源码的时候，部分不影响流程的代码也在本文省略了

##### 1. 常用的自定义标签和属性

自定义标签：`<context:annotation-config/>`, `<context:component-scan base-package="com.xx.xx"/>`, `<mvc:annotation-driven>`, `<aop:aspectj-autoproxy/>`等等都是自定义的标签；

自定义属性见得比较少，不过也有，如带 "p" 前缀的属性，是在`spring-beans`jar包中。

```
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:context="http://www.springframework.org/schema/context"
       xmlns:p="http://www.springframework.org/schema/p"
       xsi:schemaLocation="http://www.springframework.org/schema/beans
						http://www.springframework.org/schema/beans/spring-beans.xsd
						http://www.springframework.org/schema/context
						http://www.springframework.org/schema/context/spring-context.xsd">
    <!--    启用注解    -->
    <context:annotation-config/>
    <!--    设置扫描的包    -->
    <context:component-scan base-package="com.xxx"/>
    <bean id="hello" class="com.xxx.Hello"/>
    <!-- p:hello-ref 定义 HelloWorld 属性 hello，有后缀 -ref 表示bean的id -->
    <!-- p:world 定义 HelloWorld 属性 world 的值 -->
    <!-- p:world 会与下方的 property world 冲突 -->
    <bean id="helloWorld" class="com.xxx.HelloWorld"
          p:hello-ref="hello" p:world="world-value-p">
        <property name="world" value="world-value"/>
    </bean>
</beans>
```

##### 2. parseCustomElement分析

我们先从自定义标签解析方法的起始位置 `DefaultBeanDefinitionDocumentReader#parseBeanDefinitions` 看起

```
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
    // 判断是否为默认命名空间下的标签
    if (delegate.isDefaultNamespace(delegate.getNamespaceURI(root))) {
        // 循环遍历 解析默认标签
        NodeList nl = root.getChildNodes();
        for (int i = 0; i < nl.getLength(); i++) {
            Node node = nl.item(i);
            if (node instanceof Element) {
                Element ele = (Element) node;
                String namespaceUri = delegate.getNamespaceURI(ele);
                // 判断是否为默认命名空间下的标签
                if (delegate.isDefaultNamespace(namespaceUri)) {
                    // 默认标签解析方法
                    parseDefaultElement(ele, delegate);
                } else {
                    // 自定义标签解析方法
                    delegate.parseCustomElement(ele);
                }
            }
        }
    } else {
        // 自定义标签解析方法
        delegate.parseCustomElement(root);
    }
}
```
委托给了`BeanDefinitionParserDelegate#parseCustomElement`进行解析
```
public BeanDefinition parseCustomElement(Element ele) {
    return parseCustomElement(ele, null);
}

public BeanDefinition parseCustomElement(Element ele, BeanDefinition containingBd) {
    String namespaceUri = getNamespaceURI(ele);
    // 根据命名空间获取对应的处理器 (命名空间处理解析器,下面会分析)
    NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
    if (handler == null) {
        error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
        return null;
    }
    // 处理器进行解析
    return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
}
```

##### 3. NamespaceHandler分析

![](https://pic.superbed.cn/item/5dba48a6bd461d945a7503e6.png)

结合上面类图，查看源码发现，命名空间处理器的主要逻辑是放在其子类`NamespaceHandlerSupport`中，而`NamespaceHandlerSupport`的子类主要是实现`init`方法，在里面注册该命名空间下标签对应的解析器，如`ContextNamespaceHandler`的`init`方法，一个标签对应一个解析器。
```
public void init() {
    // 注册解析器 BeanDefinitionParser
    registerBeanDefinitionParser("property-placeholder", new PropertyPlaceholderBeanDefinitionParser());
    registerBeanDefinitionParser("property-override", new PropertyOverrideBeanDefinitionParser());
    registerBeanDefinitionParser("annotation-config", new AnnotationConfigBeanDefinitionParser());
    registerBeanDefinitionParser("component-scan", new ComponentScanBeanDefinitionParser());
    registerBeanDefinitionParser("load-time-weaver", new LoadTimeWeaverBeanDefinitionParser());
    registerBeanDefinitionParser("spring-configured", new SpringConfiguredBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-export", new MBeanExportBeanDefinitionParser());
    registerBeanDefinitionParser("mbean-server", new MBeanServerBeanDefinitionParser());
}
```
`NamespaceHandler`调用解析器进行解析的方法是 `parse`

```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    // 查找对应的解析器进行解析
    return findParserForElement(element, parserContext).parse(element, parserContext);
}

private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
    String localName = parserContext.getDelegate().getLocalName(element);
    // 根据标签名获取解析器
    BeanDefinitionParser parser = this.parsers.get(localName);
    if (parser == null) {
        parserContext.getReaderContext().fatal(
                "Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
    }
    return parser;
}
```

##### 4. BeanDefinitionParser分析

我们拿常用的`context:component-scan`标签解析器看看
```
public BeanDefinition parse(Element element, ParserContext parserContext) {
    // 获取 base-package 值
    String[] basePackages = StringUtils.tokenizeToStringArray(element.getAttribute(BASE_PACKAGE_ATTRIBUTE),
            ConfigurableApplicationContext.CONFIG_LOCATION_DELIMITERS);

    // Actually scan for bean definitions and register them.
    // 扫描并注册
    ClassPathBeanDefinitionScanner scanner = configureScanner(parserContext, element);
    Set<BeanDefinitionHolder> beanDefinitions = scanner.doScan(basePackages);
    registerComponents(parserContext.getReaderContext(), beanDefinitions, element);

    return null;
}
```
不同标签的具体解析逻辑就在`parse`方法中了。

##### 5. NamespaceHandler的注册

我们在前面提过的命名空间处理解析器，用来解析命名空间对应的处理器
```
NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
```
它的默认实现是`DefaultNamespaceHandlerResolver`
```
public NamespaceHandler resolve(String namespaceUri) {
    // 获取所有处理器
    Map<String, Object> handlerMappings = getHandlerMappings();
    Object handlerOrClassName = handlerMappings.get(namespaceUri);
    if (handlerOrClassName == null) {
        return null;
    } else if (handlerOrClassName instanceof NamespaceHandler) {
        return (NamespaceHandler) handlerOrClassName;
    } else {
        String className = (String) handlerOrClassName;
        try {
            Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
            if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
                throw new FatalBeanException(".....");
            }
            // 实例化处理器
            NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
            // 初始化处理器，里面就是注册解析器
            namespaceHandler.init();
            handlerMappings.put(namespaceUri, namespaceHandler);
            return namespaceHandler;
        }
        //..........此处省略部分源码
    }
}

private Map<String, Object> getHandlerMappings() {
    if (this.handlerMappings == null) {
        synchronized (this) {
            if (this.handlerMappings == null) {
                try {
                    // 加载 META-INF/spring.handlers 属性文件
                    // this.handlerMappingsLocation = "META-INF/spring.handlers"
                    Properties mappings = PropertiesLoaderUtils
                            .loadAllProperties(this.handlerMappingsLocation, this.classLoader);

                    Map<String, Object> handlerMappings = new ConcurrentHashMap<String, Object>();
                    CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
                    this.handlerMappings = handlerMappings;
                }
                //..........此处省略部分源码
            }
        }
    }
    return this.handlerMappings;
}
```

代码中出现了` "META-INF/spring.handlers" `这么一个文件，打开`spring-beans`,`spring-context`jar包下该文件，有如下内容：
```
spring-beans下

http\://www.springframework.org/schema/p=org.springframework.beans.factory.xml.SimplePropertyNamespaceHandler
http\://www.springframework.org/schema/util=org.springframework.beans.factory.xml.UtilNamespaceHandler

spring-context下

http\://www.springframework.org/schema/context=org.springframework.context.config.ContextNamespaceHandler
http\://www.springframework.org/schema/jee=org.springframework.ejb.config.JeeNamespaceHandler
http\://www.springframework.org/schema/lang=org.springframework.scripting.config.LangNamespaceHandler
http\://www.springframework.org/schema/task=org.springframework.scheduling.config.TaskNamespaceHandler
```
我们看到命名空间 `http\://www.springframework.org/schema/context`的处理器`org.springframework.context.config.ContextNamespaceHandler`的对应关系就是在这里确定的。

##### 6. 自定义属性解析

我们来回顾下 `BeanDefinitionParserDelegate#parseBeanDefinitionElement`方法

```
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
    // bean标签比较复杂，委托给了BeanDefinitionParserDelegate进行解析，返回封装了含 BeanDefinition，beanName，aliases 的BeanDefinitionHolder
    BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
    if (bdHolder != null) {
        // 解析自定义属性或标签
        bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
        try {
            // Register the final decorated instance.
            // 注册到 BeanDefinitionRegistry，也就是DefaultListableBeanFactory中的 beanDefinitionMap 中
            BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
        }catch (BeanDefinitionStoreException ex) {}
        // Send registration event. 发送bean注册事件
        getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
    }
}
```
`delegate.decorateBeanDefinitionIfRequired(ele, bdHolder)`就是自定义属性的入口了
```
public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder definitionHolder) {
    return decorateBeanDefinitionIfRequired(ele, definitionHolder, null);
}

public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
        Element ele, BeanDefinitionHolder definitionHolder, BeanDefinition containingBd) {
    BeanDefinitionHolder finalDefinition = definitionHolder;
    // Decorate based on custom attributes first.
    NamedNodeMap attributes = ele.getAttributes();
    for (int i = 0; i < attributes.getLength(); i++) {
        Node node = attributes.item(i);
        // 遍历属性
        finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
    }
    // Decorate based on custom nested elements.
    NodeList children = ele.getChildNodes();
    for (int i = 0; i < children.getLength(); i++) {
        Node node = children.item(i);
        if (node.getNodeType() == Node.ELEMENT_NODE) {
            // 遍历子标签
            finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
        }
    }
    return finalDefinition;
}

private BeanDefinitionHolder decorateIfRequired(
        Node node, BeanDefinitionHolder originalDef, BeanDefinition containingBd) {
    String namespaceUri = getNamespaceURI(node);
    if (!isDefaultNamespace(namespaceUri)) {
        // 这里跟前面的逻辑类似，就不重复讲了
        NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
        if (handler != null) {
            return handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
        }
        //..........此处省略部分源码
    }
    return originalDef;
}
```
看看装饰方法`NamespaceHandlerSupport#decorate`
```
public BeanDefinitionHolder decorate(
        Node node, BeanDefinitionHolder definition, ParserContext parserContext) {

    return findDecoratorForNode(node, parserContext).decorate(node, definition, parserContext);
}

private BeanDefinitionDecorator findDecoratorForNode(Node node, ParserContext parserContext) {
    BeanDefinitionDecorator decorator = null;
    String localName = parserContext.getDelegate().getLocalName(node);
    if (node instanceof Element) {
        // 获取元素装饰器
        decorator = this.decorators.get(localName);
    } else if (node instanceof Attr) {
        // 获取属性装饰器
        decorator = this.attributeDecorators.get(localName);
    }
    //..........此处省略部分源码
    return decorator;
}
```
再看看另外一个处理器 `SimplePropertyNamespaceHandler`的实现，这个处理器对应的是带 "p" 前缀的属性
```
public BeanDefinitionHolder decorate(Node node, BeanDefinitionHolder definition, ParserContext parserContext) {
    if (node instanceof Attr) {
        Attr attr = (Attr) node;
        String propertyName = parserContext.getDelegate().getLocalName(attr);
        String propertyValue = attr.getValue();
        MutablePropertyValues pvs = definition.getBeanDefinition().getPropertyValues();
        // 判断是否已经存在，默认标签优先解析
        if (pvs.contains(propertyName)) {
            parserContext.getReaderContext().error("Property '" + propertyName + "' is already defined using " +
                    "both <property> and inline syntax. Only one approach may be used per property.", attr);
        }
        // 以 "-ref" 结尾，表示的是值为beanName，非具体值
        if (propertyName.endsWith("-ref")) {
            propertyName = propertyName.substring(0, propertyName.length() - REF_SUFFIX.length());
            pvs.add(Conventions.attributeNameToPropertyName(propertyName), new RuntimeBeanReference(propertyValue));
        } else {
            // 将值添加到definition中
            pvs.add(Conventions.attributeNameToPropertyName(propertyName), propertyValue);
        }
    }
    return definition;
}
```

相信大家看到这里应该明白自定义的标签或属性是怎么解析的了，大家可以尝试去定义一下自己的属性标签，这里就不带大家去实践了。