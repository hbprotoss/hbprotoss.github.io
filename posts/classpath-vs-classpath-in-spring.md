<!-- 
.. title: classpath* vs classpath in spring
.. slug: classpath-vs-classpath-in-spring
.. date: 2016-07-06 09:12:52 UTC+08:00
.. tags: Spring, Java
.. category: 
.. link: 
.. description: 
.. type: text
-->

记录一下spring里读取资源时的classpath*和classpath表达式的区别。不想看细节的可以直接跳到最后直接看结论。

读取资源的主要逻辑在`PathMatchingResourcePatternResolver.getResources`

```java
public Resource[] getResources(String locationPattern) throws IOException {
	Assert.notNull(locationPattern, "Location pattern must not be null");
	if (locationPattern.startsWith(CLASSPATH_ALL_URL_PREFIX)) {
		// a class path resource (multiple resources for same name possible)
		if (getPathMatcher().isPattern(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()))) {
			// a class path resource pattern
			return findPathMatchingResources(locationPattern);
		}
		else {
			// all class path resources with the given name
			return findAllClassPathResources(locationPattern.substring(CLASSPATH_ALL_URL_PREFIX.length()));
		}
	}
	else {
		// Only look for a pattern after a prefix here
		// (to not get fooled by a pattern symbol in a strange prefix).
		int prefixEnd = locationPattern.indexOf(":") + 1;
		if (getPathMatcher().isPattern(locationPattern.substring(prefixEnd))) {
			// a file pattern
			return findPathMatchingResources(locationPattern);
		}
		else {
			// a single resource with the given name
			return new Resource[] {getResourceLoader().getResource(locationPattern)};
		}
	}
}

```

* classpath*:resource

	代码中的if逻辑，通过findAllClassPathResources，遍历整个classpath，搜索所有名字匹配的文件返回


* classpath:resource

	代码中的else逻辑，`getResourceLoader().getResource(locationPattern)`最终调用tomcat提供的WebappClassLoader，从classpath中遍历每个目录（包括jar），寻找指定的resource直到找到指定的第一个匹配文件返回
	
读取完resource列表之后，`AbstractBeanDefinitionReader.loadBeanDefinitions`会载入得到resource列表

```java
Resource[] resources = ((ResourcePatternResolver) resourceLoader).getResources(location);
int loadCount = loadBeanDefinitions(resources);
```

### 结论
从`classpath*:resource`中读取资源，就相当于把整个classpath下能找到的同名resource按照classpath中的次序拼接成一个大文件后载入，位于classpath后部的resource定义会覆盖前面的。由于tomcat会保证`WEB-INF/class`下的class会放在classpath中的第一位，所以导致jar中的同名resource定义覆盖app中的定义。

`classpath:resource`仅载入classpath路径中碰到的第一个resource。