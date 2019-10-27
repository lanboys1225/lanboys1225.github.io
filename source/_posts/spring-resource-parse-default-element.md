---
title: Spring容器加载过程源码解析之默认标签解析
date: 2019-10-27 18:33:12
category: spring
tags: [spring]
---

>在上文 [Spring容器加载过程源码解析之Resource解析流程](https://me.oopcoder.cn/2019/10/21/spring-resource-parse/) 中，我们已经了解了整个解析流程，今天我们来具体分析下默认标签的解析。

> 提醒：本文是基于Spring 3.0.0.RELEASE 版本进行讲解的，其他版本可能稍有差异，在贴源码的时候，部分不影响流程的代码也在本文省略了

##### 1.什么是默认标签和自定义标签
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

// 判断是否为默认命名空间下的标签
public boolean isDefaultNamespace(String namespaceUri) {
	return (!StringUtils.hasLength(namespaceUri) || "http://www.springframework.org/schema/beans".equals(namespaceUri));
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
从 `isDefaultNamespace()` 方法可以看出，在命名空间 `http://www.springframework.org/schema/beans` 下面定义的标签就是我们所说的默认标签，比如顶级标签 `beans`, 以及下面的 `bean`,`import`,`alias`等等其他标签。

而不在默认命名空间下的标签就是自定义标签了，比如我们最常见的带 `context` 前缀的标签：`<context:annotation-config/>`,`<context:component-scan base-package="com.xx.xx"/>`等。

##### 2.bean 标签解析方法 processBeanDefinition

委托给了`BeanDefinitionParserDelegate#parseBeanDefinitionElement`进行解析，解析完成后再进行注册。

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
接下看看代理类 `BeanDefinitionParserDelegate`
```
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
    // beans 标签下的 bean, containingBean 为空
    return parseBeanDefinitionElement(ele, null);
}

public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, BeanDefinition containingBean) {
    // 获取属性 id 值，用作 beanName
    String id = ele.getAttribute("id");
    // 获取属性 name 值，用作别名, 多个别名可用‘，’或‘；’分割开来
    String nameAttr = ele.getAttribute("name");
    // 分割多个别名
    List<String> aliases = new ArrayList<String>();
    if (StringUtils.hasLength(nameAttr)) {
        String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, BEAN_NAME_DELIMITERS);
        aliases.addAll(Arrays.asList(nameArr));
    }
    // 属性id值作为 bean 的名称
    String beanName = id;
    if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
        // 如果未定义id, 则将 第一个别名作为 beanName
        beanName = aliases.remove(0);
    }
    // beans 标签下的bean, 需要检查beanName和aliases的在当前xml配置文件中的唯一性
    // bean 标签中嵌套的 bean 不需要检查
    if (containingBean == null) {
        checkNameUniqueness(beanName, aliases, ele);
    }
    // 解析标签的具体操作
    AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
    if (beanDefinition != null) {
        // 下面主要是没有定义 beanName 时，名字自动生成的逻辑
        if (!StringUtils.hasText(beanName)) {
            try {
                if (containingBean != null) {
                    beanName = BeanDefinitionReaderUtils.generateBeanName(
                            beanDefinition, this.readerContext.getRegistry(), true);
                } else {
                    beanName = this.readerContext.generateBeanName(beanDefinition);

                    String beanClassName = beanDefinition.getBeanClassName();
                    if (beanClassName != null &&
                            beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
                            !this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
                        aliases.add(beanClassName);
                    }
                }
            }
            //..........此处省略部分源码
        }
        String[] aliasesArray = StringUtils.toStringArray(aliases);
        // 返回 holder
        return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
    }
    return null;
}
```
上面这部分主要是处理名字和别名的逻辑，下面看下解析的具体操作
```
public AbstractBeanDefinition parseBeanDefinitionElement(
        Element ele, String beanName, BeanDefinition containingBean) {
    this.parseState.push(new BeanEntry(beanName));

    String className = null;
    // 解析 class 属性
    if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
        className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
    }
    try {
        String parent = null;
        // 解析 parent 属性
        if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
            parent = ele.getAttribute(PARENT_ATTRIBUTE);
        }
        // 创建GenericBeanDefinition实例
        AbstractBeanDefinition bd = createBeanDefinition(className, parent);
        // 解析其他属性 
        parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
        bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));
        // 解析meta标签
        parseMetaElements(ele, bd);
        // 解析lookup-method标签
        parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
        // 解析replaced-method标签
        parseReplacedMethodSubElements(ele, bd.getMethodOverrides());
        // 解析constructor-arg标签
        parseConstructorArgElements(ele, bd);
        // 解析property标签
        parsePropertyElements(ele, bd);
        // 解析qualifier标签
        parseQualifierElements(ele, bd);

        bd.setResource(this.readerContext.getResource());
        bd.setSource(extractSource(ele));
        return bd;
    }
    //..........此处省略部分源码
    finally {
        this.parseState.pop();
    }
    return null;
}
```
上面就是解析的具体过程了，更具体的解析细节比较繁琐，不过逻辑简单，这里就不一一贴出来了，小伙伴们可以自己跟进去看看。

##### 3.import 标签解析方法 importBeanDefinitionResource
```
protected void importBeanDefinitionResource(Element ele) {
    String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
    if (!StringUtils.hasText(location)) {
        return;
    }
    // Resolve system properties: e.g. "${user.dir}"
    // 解析占位符
    location = SystemPropertyUtils.resolvePlaceholders(location);
    Set<Resource> actualResources = new LinkedHashSet<Resource>(4);
    // 判断是绝对还是相对路径
    boolean absoluteLocation = false;
    try {
        absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
    } catch (URISyntaxException ex) {}

    // Absolute or relative?
    if (absoluteLocation) {
        try {
            // 确定是绝对路径后，直接交给 XmlBeanDefinitionReader 进行定位，加载，解析
            int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
        } catch (BeanDefinitionStoreException ex) {}
    } else {
        try {
            int importCount;
            // 创建与当前资源同一父路径的相对资源
            Resource relativeResource = getReaderContext().getResource().createRelative(location);
            if (relativeResource.exists()) {
                // 资源存在，直接加载
                importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
                actualResources.add(relativeResource);
            } else {
                // 资源不存在，表明路径是含有通配符的路径
                String baseLocation = getReaderContext().getResource().getURL().toString();
                importCount = getReaderContext().getReader().loadBeanDefinitions(
                        StringUtils.applyRelativePath(baseLocation, location), actualResources);
            }
        }
        //..........此处省略部分源码
    }
    Resource[] actResArray = actualResources.toArray(new Resource[actualResources.size()]);
    // 发送import事件
    getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
}
```
`import`标签解析比较简单，上面提到的资源 定位，加载方法: `loadBeanDefinitions`, 在[Spring容器加载过程源码解析之Resource定位加载](https://me.oopcoder.cn/2019/10/18/spring-resource-load/)一文中有详细分析，想了解的小伙伴可以进去看看。

##### 4.alias 标签解析方法 processAliasRegistration

```
protected void processAliasRegistration(Element ele) {
    String name = ele.getAttribute(NAME_ATTRIBUTE);
    String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
    boolean valid = true;
    if (!StringUtils.hasText(name)) {
        valid = false;
    }
    if (!StringUtils.hasText(alias)) {
        valid = false;
    }
    if (valid) {
        try {
            // 注册别名
            getReaderContext().getRegistry().registerAlias(name, alias);
        } catch (Exception ex) {}
        // 发送alias注册事件
        getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
    }
}
```

`alias`标签跟`bean`标签下的`alias`属性都一样是`bean`的别名，那有什么作用呢？最简单的例子，比如，配置文件存在第三方`jar`包中，我们无法修改，但是又想通过特定的别名进行访问，这时候alias标签就派上用场了。
```
// jar包中配置
<bean id="hello"  name="hello-alias-1, hello-alias-2"  class=“com.xx.xx"/>
// 自己的配置
<alias name="hello" alias="hello-alias-3"/>
<alias name="hello" alias="hello-alias-4"/>
```
按照上面这么配置，`hello` 就有4个别名了。

好了，今天的默认标签解析就分析到这里了，下篇文章我们再分析自定义标签是如何解析的。