---
title: Spring容器加载过程之源码解析之Resource解析流程
date: 2019-10-21 22:55:42
category: spring
tags: [spring]
---

>在上文 [Spring容器加载过程之源码解析之Resource定位加载](https://me.oopcoder.cn/2019/10/18/spring-resource-load/) 中，我们已经将资源路径解析为Resource了，今天我们来分析下整个解析流程。

> 提醒：本文是基于Spring 3.0.0.RELEASE 版本进行讲解的，其他版本可能稍有差异，在贴源码的时候，部分不影响流程的代码也在本文省略了

##### 1. XmlBeanDefinitionReader 分析
```
public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
    ResourceLoader resourceLoader = getResourceLoader();
    //..........此处省略部分源码
    if (resourceLoader instanceof ResourcePatternResolver) {
        try {
            // 这里才是真正将 资源路径 解析为Resource的地方
            Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
            // 开始解析
            int loadCount = loadBeanDefinitions(resources);
            //..........此处省略部分源码
        }
    }
    //..........此处省略部分源码
}

public int loadBeanDefinitions(Resource[] resources) throws BeanDefinitionStoreException {
    Assert.notNull(resources, "Resource array must not be null");
    int counter = 0;
    for (Resource resource : resources) {
        // 遍历所有资源 进行解析
        counter += loadBeanDefinitions(resource);
    }
    return counter;
}

public int loadBeanDefinitions(Resource resource) throws BeanDefinitionStoreException {
    return loadBeanDefinitions(new EncodedResource(resource));
}

public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
    //..........此处省略部分源码
    try {
        InputStream inputStream = encodedResource.getResource().getInputStream();
        try {
            InputSource inputSource = new InputSource(inputStream);
            if (encodedResource.getEncoding() != null) {
                inputSource.setEncoding(encodedResource.getEncoding());
            }
            // 从Resource获取输入流进行解析
            return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
        }
        finally {
            inputStream.close();
        }
    }
    //..........此处省略部分源码
}

protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
        throws BeanDefinitionStoreException {
    try {
        int validationMode = getValidationModeForResource(resource);
        // 通过 documentLoader 将资源转换为 Document 对象
        Document doc = this.documentLoader.loadDocument(
                inputSource, getEntityResolver(), this.errorHandler, validationMode, isNamespaceAware());
        return registerBeanDefinitions(doc, resource);
    }
    //..........此处省略部分源码
}

// EntityResolver 由 BeansDtdResolver 和 PluggableSchemaResolver 组成
// 用来在 classpath 下搜寻 schema 和 DTD 文件
protected EntityResolver getEntityResolver() {
    if (this.entityResolver == null) {
        // Determine default EntityResolver to use.
        ResourceLoader resourceLoader = getResourceLoader();
        if (resourceLoader != null) {
            this.entityResolver = new ResourceEntityResolver(resourceLoader);
        } else {
            this.entityResolver = new DelegatingEntityResolver(getBeanClassLoader());
        }
    }
    return this.entityResolver;
}

public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
    // Read document based on new BeanDefinitionDocumentReader SPI.
    BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
    int countBefore = getRegistry().getBeanDefinitionCount();
    // 委托 BeanDefinitionDocumentReader 解析document
    documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
    return getRegistry().getBeanDefinitionCount() - countBefore;
}

protected XmlReaderContext createReaderContext(Resource resource) {
    if (this.namespaceHandlerResolver == null) {
        // 创建默认命名空间处理解析器
        this.namespaceHandlerResolver = createDefaultNamespaceHandlerResolver();
    }
    return new XmlReaderContext(resource, this.problemReporter, this.eventListener,
            this.sourceExtractor, this, this.namespaceHandlerResolver);
}

protected NamespaceHandlerResolver createDefaultNamespaceHandlerResolver() {
    // 用来解析 META-INF/spring.handlers 目录下对应的处理器，自定义标签的时候也会使用到，我们后面会详细分析
    return new DefaultNamespaceHandlerResolver(getResourceLoader().getClassLoader());
}
```
从上述代码中，我们看到XmlBeanDefinitionReader完成了如下工作：
1. 由内部的`ResourceLoader`去获取所有符合条件的`Resource` (上文已重点分析过 [Spring容器加载过程之源码解析之Resource定位加载](https://me.oopcoder.cn/2019/10/18/spring-resource-load/))；
2. 从`Resource`中获取流，转化为方便解析的`Document`对象；
3. 委托`BeanDefinitionDocumentReader`来解析 `Document`，所以实际的bean注册工作也是由它来完成。

##### 2. DefaultBeanDefinitionDocumentReader 分析
```
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
    this.readerContext = readerContext;
    // 获取根元素，这里一般就是 beans 标签
    Element root = doc.getDocumentElement();
    // 创建解析bean的代理类
    BeanDefinitionParserDelegate delegate = createHelper(readerContext, root);
    // 与下方的 postProcessXml 一样，在这里都没有实现，可用来扩展自定义标签
    preProcessXml(root);
    // 解析根元素
    parseBeanDefinitions(root, delegate);
    postProcessXml(root);
}

protected BeanDefinitionParserDelegate createHelper(XmlReaderContext readerContext, Element root) {
    BeanDefinitionParserDelegate delegate = new BeanDefinitionParserDelegate(readerContext);
    // 初始化默认的设置，即 beans 标签的属性
    // Initialize the default lazy-init, autowire, dependency check settings, init-method, destroy-method and merge settings
    delegate.initDefaults(root);
    return delegate;
}

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

private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
    if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
        // 解析 import 标签
        importBeanDefinitionResource(ele);
    } else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
        // 解析 alias 标签
        processAliasRegistration(ele);
    } else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
        // 解析 bean 标签
        processBeanDefinition(ele, delegate);
    }
}
```
来到这里，`Resource` 解析的整个流程就清晰了许多，默认标签和自定义标签的具体解析我们将在后面的文章再来分析。
