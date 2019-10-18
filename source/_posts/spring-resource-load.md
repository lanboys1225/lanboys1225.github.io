---
title: Spring容器加载过程源码解析之Resource定位加载
date: 2019-10-18 15:03:02
category: spring
tags: [spring]
---

>在上文 [如何手动启动Spring容器](https://me.oopcoder.cn/2019/10/12/spring-startup-java/) 中，我们知道了可以通过传入资源文件来启动容器，如果将`applicationContext.xml`替换为绝对路径就启动不了，报错。

```
//启动容器
ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml")

//更换为绝对路径，启动失败
ApplicationContext context = new ClassPathXmlApplicationContext(
                "E:\\IDEA_workspace\\SpringLearningDemo\\spring-java\\src\\main\\resources\\applicationContext.xml");

//报 FileNotFoundException：Caused by: java.io.FileNotFoundException: class path resource [E:/IDEA_workspace/SpringLearningDemo/spring-java/src/main/resources/applicationContext.xml] cannot be opened because it does not exist

//使用 FileSystemXmlApplicationContext 正常启动
ApplicationContext context = new FileSystemXmlApplicationContext(
                "E:\\IDEA_workspace\\SpringLearningDemo\\spring-java\\src\\main\\resources\\applicationContext.xml");
        
// 启动成功
ApplicationContext context = new FileSystemXmlApplicationContext(
                "classpath:applicationContext.xml");
// 启动成功           
ApplicationContext context = new ClassPathXmlApplicationContext(
                "file:E:\\IDEA_workspace\\SpringLearningDemo\\spring-java\\src\\main\\resources\\applicationContext.xml");
```
>通过字面意思，可以看出`ClassPathXmlApplicationContext`, 传入的是classpath目录下的资源文件, `FileSystemXmlApplicationContext`, 传入的是文件系统下资源文件(即文件绝对路径), 我们在资源文件前面加上 classpath 或 file，启动试试，发现也都成功了，看来资源的定位没有上面说的那么简单，还跟资源路径前缀有关。

> 提醒：本文是基于Spring 3.0.0.RELEASE 版本进行讲解的，其他版本可能稍有差异，在贴源码的时候，部分不影响流程的代码也在本文省略了

下面带着疑问来看看Spring资源到底是怎么定位和加载进来的。进入ClassPathXmlApplicationContext源码，按下面路径走, 找到getResources方法：
```
    // 路径
    refresh() -> obtainFreshBeanFactory() -> refreshBeanFactory() -> loadBeanDefinitions(DefaultListableBeanFactory beanFactory)  
 
    // AbstractXmlApplicationContext.java
 	@Override
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// 创建 XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);
		// 传入ResourceLoader
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));
    	initBeanDefinitionReader(beanDefinitionReader);

		loadBeanDefinitions(beanDefinitionReader);
	}
	
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		// 获取容器启动时，传入的资源文件路径
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
	
	// AbstractBeanDefinitionReader.java
	public int loadBeanDefinitions(String location, Set<Resource> actualResources) throws BeanDefinitionStoreException {
		ResourceLoader resourceLoader = getResourceLoader();
		//..........此处省略部分源码
		if (resourceLoader instanceof ResourcePatternResolver) {
			// Resource pattern matching available.
			try {
				// 这里才是真正将 资源路径 解析为Resource的地方
				Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
				int loadCount = loadBeanDefinitions(resources);
		        //..........此处省略部分源码
		    }
		}
		//..........此处省略部分源码
	}
```

从上面可以看出，`AbstractXmlApplicationContext` 将`Bean`的装载任务委派给了`XmlBeanDefinitionReader`, 而`ResourceLoader`负责将`xml`解析为`Resource`。

先看看类结构图

![](https://pic.superbed.cn/item/5da86a9b451253d1780df669.png)

从上面可以看到，Spring为了统一不同类型的资源访问，把所有资源都抽象成`Resource`接口, 屏蔽不同资源之间的差异，其实现了`InputStreamSource`, 这样所有`Resource`都可以通过`getInputStream()`获取`InputStream`。

![](https://pic.superbed.cn/item/5da86eb4451253d1780e7ce3.png)

其中`资源定位路径 --> Resource` 这个过程是由 `ResourceLoader` 这个类来完成的，我们暂且称之为: 资源定位器, 其默认实现是 `DefaultResourceLoader`。

`ResourcePatternResolver` 继承自 `ResourceResolver` ，扩展了一个能通过`路径模式`匹配定位的方法： `Resource[] getResources(String locationPattern)`，其路径模式支持以  `classpath / classpath*` 或 URI协议名（例如 `http、file、jar:file`）为其前缀，还支持 `Ant风格` 的匹配模式。

看看 `ResourcePatternResolver` 的实现类 `PathMatchingResourcePatternResolver`

```	
    // 构造方法创建的 DefaultResourceLoader
    private final ResourceLoader resourceLoader;
    // Ant风格 路径匹配器
	private PathMatcher pathMatcher = new AntPathMatcher();
 
	public Resource[] getResources(String locationPattern) throws IOException {
		if (locationPattern.startsWith("classpath*:")) {
			// "classpath*:" 后的路径是否含有通配符 “*” 或 “?”，即是否为路径模式
			if (getPathMatcher().isPattern(locationPattern.substring("classpath*:".length()))) {
			    // eg: classpath*:a/b/applicationContext-*.xml
				// 根据路径模式 查找所有匹配的资源
				return findPathMatchingResources(locationPattern);
			} else {
			    // eg: classpath*:a/b/applicationContext-dao.xml 或 classpath*:a/b/
				// 根据确定路径 在所有classpath中(包含所有jar包)查找资源
				return findAllClassPathResources(locationPattern.substring("classpath*:".length()));
			}
		} else {
			// 第一个 ":" 后的路径是否含有通配符 “*” 或 “?”，即是否为路径模式
			int prefixEnd = locationPattern.indexOf(":") + 1;
			if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
				// 根据路径模式 查找所有匹配的资源
				return findPathMatchingResources(locationPattern);
			} else {
				// 加载单个资源，后面再详细讲讲 getResource 这个方法
				return new Resource[] {getResourceLoader().getResource(locationPattern)};
			}
		}
	}
	
	// 根据路径模式 查找所有匹配的资源
	protected Resource[] findPathMatchingResources(String locationPattern) throws IOException {
	    // 获取到一个不含通配符的根目录，如 classpath*:a/b/*/applicationContext-*.xml 返回 classpath*:a/b/
		String rootDirPath = determineRootDir(locationPattern);
		// 返回 */applicationContext-*.xml
		String subPattern = locationPattern.substring(rootDirPath.length());
		// 递归调用getResources(), 获取的符合根目录的所有资源
		Resource[] rootDirResources = getResources(rootDirPath);
		Set<Resource> result = new LinkedHashSet<Resource>(16);
		// 遍历根目录资源，将匹配的资源添加到 result 中
		for (Resource rootDirResource : rootDirResources) {
		    // jar:file:开头的为jar包资源
			if (isJarResource(rootDirResource)) {
				result.addAll(doFindPathMatchingJarResources(rootDirResource, subPattern));
			} else if (rootDirResource.getURL().getProtocol().startsWith(ResourceUtils.URL_PROTOCOL_VFS)) {
				result.addAll(VfsResourceMatchingDelegate.findMatchingResources(rootDirResource, subPattern, getPathMatcher()));
			} else {
			    // 其他类型资源走这里
				result.addAll(doFindPathMatchingFileResources(rootDirResource, subPattern));
			}
		}
		return result.toArray(new Resource[result.size()]);
	}
	
	protected Set<Resource> doFindPathMatchingFileResources(Resource rootDirResource, String subPattern)
			throws IOException {
		File rootDir = rootDirResource.getFile().getAbsoluteFile();
		//..........此处省略部分源码
		return doFindMatchingFileSystemResources(rootDir, subPattern);
	}
	
	protected Set<Resource> doFindMatchingFileSystemResources(File rootDir, String subPattern) throws IOException {
        // retrieveMatchingFiles 流程比较长就不贴出来了，有兴趣的朋友可以debug进去看看
		Set<File> matchingFiles = retrieveMatchingFiles(rootDir, subPattern);
		Set<Resource> result = new LinkedHashSet<Resource>(matchingFiles.size());
		for (File file : matchingFiles) {
		    // 符合条件的路径 此时就解析为Resource了
			result.add(new FileSystemResource(file));
		}
		return result;
	}
```
上面就是`PathMatchingResourcePatternResolver`路径模式匹配的基本逻辑。加载单个资源的任务是委托给了构造方法里面创建的 `DefaultResourceLoader`

DefaultResourceLoader
```
	public Resource getResource(String location) {
        // 以 classpath: 开头的返回 ClassPathResource
		if (location.startsWith("classpath:")) {
			return new ClassPathResource(location.substring("classpath:".length()), getClassLoader());
		} else {
			try {
				// 能够解析为URL的返回 UrlResource
				URL url = new URL(location);
				return new UrlResource(url);
			} catch (MalformedURLException ex) {
				// 不能够解析为URL的 由getResourceByPath返回
				return getResourceByPath(location);
			}
		}
	}
	
	protected Resource getResourceByPath(String path) {
	    // 默认返回 ClassPathContextResource
		return new ClassPathContextResource(path, getClassLoader());
	}
```
从上面的类结构图，我们看到 `FileSystemXmlApplicationContext` 和 `ClassPathXmlApplicationContext` 都是 `DefaultResourceLoader` 的子类，区别在于前者重写了 `getResourceByPath` 方法
```
	protected Resource getResourceByPath(String path) {
		if (path != null && path.startsWith("/")) {
			path = path.substring(1);
		}
		return new FileSystemResource(path);
	}
```
那么 不带前缀的 路径或者 自定义前缀 的路径能否定位成功，关键在于`getResourceByPath`方法了，回到文章开始 传入的路径:

`applicationContext.xml` 和 `E:\\IDEA_workspace\\SpringLearningDemo\\spring-java\\src\\main\\resources\\applicationContext.xml`
> 走的是 DefaultResourceLoader 默认的 getResourceByPath方法，返回的classpath里面的资源，所以后者在ClassPathXmlApplicationContext里面是启动不了的，反之前者在FileSystemXmlApplicationContext也启动不了
 
`classpath:applicationContext.xml` 和 `file:E:\\IDEA_workspace\\SpringLearningDemo\\spring-java\\src\\main\\resources\\applicationContext.xml`
> 都带有前缀，所以跟getResourceByPath无关，也就是跟 FileSystemXmlApplicationContext 和 ClassPathXmlApplicationContext 无关，传入哪个Context都能启动成功

总结起来, Spring的资源路径填写方式如下：

前缀 | 示例 | 说明
---|---|---
classpath: | classpath:a/c.xml | 从classpath中加载，存在多个资源则返回最先加载的那个资源，易导致资源加载不进来的问题
classpath*: | classpath*:a/c.xml | 从classpath中加载，返回全部符合条件的资源，需要遍历所有classpath, 因此加载效率低
file: 或 jar:file: 等URI协议名 | file:d:\\b\\f.xml |作为URL进行加载
没有前缀 | a/c.xml 或 d:\\b\\f.xml | 根据context的getResourceByPath方法判断

 Ant风格 的匹配模式
>  “ ? ”：匹配一个字符, 如 a?.xml 匹配 ab.xml

>  “ * ”：匹配零个或多个字符串，如“a/ * /c.xml”将匹配“a/b/c.xml”，但不匹配匹配“a/c.xml”；而“a/c-*.xml”将匹配“a/c-dao.xml”

> “ ** ”：匹配路径中的零个或多个目录，如“a/ ** /c.xml”将匹配“a /c.xml”，也匹配“a/b/b/c.xml”；而“a/b/c- ** .xml”将匹配“a/b/c-dao.xml”，即把“ ** ”当做两个“ * ”处理。
 
 这就是Spring容器中资源的定位，然后通过 `Resource` 的 `getInputStream()` 加载进内存进行解析，至于如何解析，就请听下回分解啦。
 