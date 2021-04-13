### 1.spring容器的基本实现

#### 1.1 spring 的结构组成

##### 1.1.1 核心类

- DefaultListableBeanFactory：Spring 注册及加载 Bean 的默认实现。DefaultListableBeanFactory 继承了AbstractAutowireCapableBeanFactory 并实现了 ConfigurableListableBeanFactory 以及 BeanDefinitionRegistry 接口。
  - XMLBeanFactory 继承自 DefaultListableBeanFactory，而对于 XMLBeanFactory 与 DefaultListableBeanFactory 不同的地方是在 XMLBeanFactory 中使用了自定义的 XML 读取器 XMLBeanDefinitionReader，实现了个性化的 BeanDefinitionReader 读取。



- XmlBeanDefinitionReader：XMLBeanDefinitionReader提供了 xml 资源文件读取、解析及注册的大致脉络。

  

#### 1.2 容器的基础 XmlBeanFactory

- 数据准备阶段的逻辑：首先对传入的 resource 参数做封装，目的是考虑到 Resource 可能存在编码要求的情况，其次，通过 SAX 读取 XML 文件的方式来准备 InputSource 对象，最后将准备的数据通过参数传入真正的核心处理部分 doLoadBeanDefinitions(inputSource, encodedResource.getResource())。
- **doLoadBeanDefinitions 的逻辑，基本上做了三件事情**：
  - Document doc = doLoadDocument(inputSource, resource);	
    - 1、getValidationModeForResource(resource)：获取对 XML 文件的验证模式；
    - 2、加载 XML 文件，并得到对应的 Document 对象。
  - registerBeanDefinitions(doc, resource)：根据返回的 Document 对象注册 Bean 信息。

```java
/*

*/
public int loadBeanDefinitions(EncodedResource encodedResource) throws BeanDefinitionStoreException {
		Assert.notNull(encodedResource, "EncodedResource must not be null");
		if (logger.isInfoEnabled()) {
			logger.info("Loading XML bean definitions from " + encodedResource);
		}
		// 通过属性来记录已经加载的资源
		Set<EncodedResource> currentResources = this.resourcesCurrentlyBeingLoaded.get();
		if (currentResources == null) {
			currentResources = new HashSet<>(4);
			this.resourcesCurrentlyBeingLoaded.set(currentResources);
		}
		if (!currentResources.add(encodedResource)) {
			throw new BeanDefinitionStoreException(
					"Detected cyclic loading of " + encodedResource + " - check your import definitions!");
		}
		try {
			// 从 encodedResource 中获取已经封装的 Resource 对象
			// 并再次从 Resource 中获取其中的 inputStream
			InputStream inputStream = encodedResource.getResource().getInputStream();
			try {
				// InputSource 这个类不来自于 Spring，它的全路径是 org.xml.sax.InputSource
				InputSource inputSource = new InputSource(inputStream);
				if (encodedResource.getEncoding() != null) {
					inputSource.setEncoding(encodedResource.getEncoding());
				}
				// 逻辑核心部分
				return doLoadBeanDefinitions(inputSource, encodedResource.getResource());
			}
			finally {
				inputStream.close();
			}
		}
		catch (IOException ex) {
			throw new BeanDefinitionStoreException(
					"IOException parsing XML document from " + encodedResource.getResource(), ex);
		}
		finally {
			currentResources.remove(encodedResource);
			if (currentResources.isEmpty()) {
				this.resourcesCurrentlyBeingLoaded.remove();
			}
		}
	}




protected int doLoadBeanDefinitions(InputSource inputSource, Resource resource)
			throws BeanDefinitionStoreException {
		try {
			// 加载 XML 文件，并得到对应的 Document 对象。
			Document doc = doLoadDocument(inputSource, resource);
			// 根据返回的 Document 对象注册 Bean 信息
			return registerBeanDefinitions(doc, resource);
		}
```



#### 1.3 获取 XML 的验证模式

- Spring 通过 getValidationModeForResource 方法来获取对应资源的验证模式；
  - 如果设定了验证模式则使用设定的验证模式（可以通过对调用 XmlBeanDefinitionReader 中的 setValidationMode 方法进行设定），否则是哟好难过自动检测的方式。
  - 自动检测验证模式的功能是在函数 detectValidationMode 方法中实现的，在 detectValidationMode 函数中又将自动检测验证模式的工作委托给专门处理类 XmlValidationModeDetector，调用了 XmlValidationModeDetector 的 validationModeDetector 方法

```java
protected int getValidationModeForResource(Resource resource) {
		int validationModeToUse = getValidationMode();
		// 如果手动指定了验证模式则使用指定的验证模式
		if (validationModeToUse != VALIDATION_AUTO) {
			return validationModeToUse;
		}
		// 如果未指定则使用自动检测
		int detectedMode = detectValidationMode(resource);
		if (detectedMode != VALIDATION_AUTO) {
			return detectedMode;
		}
		return VALIDATION_XSD;
	}


protected int detectValidationMode(Resource resource) {
		if (resource.isOpen()) {
			// 异常处理
		}

		InputStream inputStream;
		try {
			inputStream = resource.getInputStream();
		}
		catch (IOException ex) {
			// 异常处理
		}

		try {
			return this.validationModeDetector.detectValidationMode(inputStream);
		}
		catch (IOException ex) {
			// 异常处理
		}
	}

```



XmlValidationModeDetector.java

```java
public int detectValidationMode(InputStream inputStream) throws IOException {
		// Peek into the file to look for DOCTYPE.
		BufferedReader reader = new BufferedReader(new InputStreamReader(inputStream));
		try {
			boolean isDtdValidated = false;
			String content;
			while ((content = reader.readLine()) != null) {
				content = consumeCommentTokens(content);
				// 如果读取的行是空或者是注释则略过
				if (this.inComment || !StringUtils.hasText(content)) {
					continue;
				}
				if (hasDoctype(content)) {
					isDtdValidated = true;
					break;
				}
				// 读取到 < 开始符号，验证模式一定会在开始符号之前
				if (hasOpeningTag(content)) {
					// End of meaningful data...
					break;
				}
			}
			return (isDtdValidated ? VALIDATION_DTD : VALIDATION_XSD);
		}
		catch (CharConversionException ex) {
			// Choked on some character encoding...
			// Leave the decision up to the caller.
			return VALIDATION_AUTO;
		}
		finally {
			reader.close();
		}
	}
```

#### 1.4 获取 Document

- 经过了验证模式准备的步骤后进行 Document 加载， XMLBeanFactoryReader 将**文档读取**委托给了 DocumentLoader 去执行，DocumentLoader 是个接口，真正调用的是 DefaultDocumentLoader，解析代码如下：

DefaultDocumentLoader.java

```java
@Override
	public Document loadDocument(InputSource inputSource, EntityResolver entityResolver,
			ErrorHandler errorHandler, int validationMode, boolean namespaceAware) throws Exception {
		// 首先创建 DocumentBuilderFactory
		DocumentBuilderFactory factory = createDocumentBuilderFactory(validationMode, namespaceAware);
		if (logger.isDebugEnabled()) {
			logger.debug("Using JAXP provider [" + factory.getClass().getName() + "]");
		}
		// 再通过 DocumentBuilderFactory 创建 DocumentBuilder
		// 进而解析 InputSource 来返回 Document 对象
		DocumentBuilder builder = createDocumentBuilder(factory, entityResolver, errorHandler);
		return builder.parse(inputSource);
	}
```

- EntityResolver 作用：项目本身就可以提供一个如何寻找 DTD 声明的方法，即由程序来实现寻找 DTD声明的过程，我们可以将 DTD 文件放到项目中某处，在实现是直接将此文档读取并返回给 SAX 即可。这样就避免了通过网络来寻找相应的声明。

  

#### 1.5 解析并注册 BeanDefinitions

- 当程序已经拥有 XML 文档文件的 Document 实例对象时，就会被引入 registerBeanDefinitions 方法：
  - doc ：通过loadDocument 加载转换出来的；

```java
public int registerBeanDefinitions(Document doc, Resource resource) throws BeanDefinitionStoreException {
		// 使用 DefaultBeanDefinitionDocumentReader 实例化 BeanDefinitionDocumentReader
		BeanDefinitionDocumentReader documentReader = createBeanDefinitionDocumentReader();
		// 在实例化 BeanDefinitionReader 时候会将 BeanDefinitionRegistry 传入，
		// 默认使用继承自 DefaultListableBeanFactory 的子类
		// 记录统计前 BeanDefinition 的加载个数
		int countBefore = getRegistry().getBeanDefinitionCount();
		// 加载及注册 bean（核心）
		documentReader.registerBeanDefinitions(doc, createReaderContext(resource));
		// 记录本次加载的 BeanDefinition 个数
		return getRegistry().getBeanDefinitionCount() - countBefore;
	}
```

- 通过 DefaultBeanDefinitionDocumentReader 的 registerBeanDefinitions 方法处理，提取 root ，以便于再次将 root 作为参数继续 BeanDefinition 的注册。

```java
public void registerBeanDefinitions(Document doc, XmlReaderContext readerContext) {
		this.readerContext = readerContext;
		logger.debug("Loading bean definitions");
		Element root = doc.getDocumentElement();
		// 真正地开始进行解析
		doRegisterBeanDefinitions(root);
	}
```

- 之前一直是 XML 加载解析的准备阶段，而  doRegisterBeanDefinitions(root) 算是真正地开始解析：
  - preProcessXml 和 postProcessXml方法体是空的，这是模板方法模式，如果继承自 DefaultBeanDefinitionDocumentReader 的子类需要在 Bean 解析前后做一些处理的话，那么只需要重写这两个方法即可。

```java
protected void doRegisterBeanDefinitions(Element root) {
		
		// 专门处理解析
		BeanDefinitionParserDelegate parent = this.delegate;
		this.delegate = createDelegate(getReaderContext(), root, parent);
		
		if (this.delegate.isDefaultNamespace(root)) {
			// 处理 profile 属性，PROFILE_ATTRIBUTE
			String profileSpec = root.getAttribute(PROFILE_ATTRIBUTE);
			if (StringUtils.hasText(profileSpec)) {
				String[] specifiedProfiles = StringUtils.tokenizeToStringArray(
						profileSpec, BeanDefinitionParserDelegate.MULTI_VALUE_ATTRIBUTE_DELIMITERS);
				if (!getReaderContext().getEnvironment().acceptsProfiles(specifiedProfiles)) {
					if (logger.isInfoEnabled()) {
						logger.info("Skipped XML bean definition file due to specified profiles [" + profileSpec +
								"] not matching: " + getReaderContext().getResource());
					}
					return;
				}
			}
		}
		// 解析前处理，留给子类实现
		preProcessXml(root);
		// 解析的核心逻辑
		parseBeanDefinitions(root, this.delegate);
		// 解析后处理，留给子类实现
		postProcessXml(root);

		this.delegate = parent;
	}
```

- 解析并注册 BeanDefinition

处理了 profile 后就进行 XML 的读取，跟踪代码进入 parseBeanDefinitions(root, this.delegate)

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		// 对 beans 的处理
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						// 对 bean 的处理，默认标签
						parseDefaultElement(ele, delegate);
					}
					else {
						// 对 bean 的处理，自定义标签
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```



Spring 的 XML 配置里面有两大类 Bean 声明，一个是默认的，如：

```xml
<bean id = "test" class = "test.TestBean" />
```

另一类就是自定义的，如：

```xml
<tx:annotation-driven />
```

这两种方式的读取及解析差别是非常大的，对于根节点或者子节点如果是默认命名空间的话则采用 parseDefaultElement 方法进行解析，否则使用 delegate.parseCustomElement(ele); 方法对自定义命名空间进行解析。



### 2.默认标签的解析

- 通过 parseDefaultElement 方法对 4 种不同的标签（import、alias、bean 和 beans）进行相应的解析。

```java
private void parseDefaultElement(Element ele, BeanDefinitionParserDelegate delegate) {
		// 对 Import 标签的处理
		if (delegate.nodeNameEquals(ele, IMPORT_ELEMENT)) {
			importBeanDefinitionResource(ele);
		}
		// 对 alias 标签的处理
		else if (delegate.nodeNameEquals(ele, ALIAS_ELEMENT)) {
			processAliasRegistration(ele);
		}
		// 对 bean 标签的处理
		else if (delegate.nodeNameEquals(ele, BEAN_ELEMENT)) {
			processBeanDefinition(ele, delegate);
		}
		// 对 beans 标签的处理
		else if (delegate.nodeNameEquals(ele, NESTED_BEANS_ELEMENT)) {
			// recurse
			doRegisterBeanDefinitions(ele);
		}
	}
```

#### 2.1 bean 标签的解析及注册

- 对 bean 标签的解析最为复杂也最为重要。

```java
protected void processBeanDefinition(Element ele, BeanDefinitionParserDelegate delegate) {
		// 首先委托 BeanDefinitionParserDelegate 类的 parseBeanDefinitionElement 方法进行元素解析，返回 BeanDefinitionHolder 类型的实例 bdHolder，经过这个方法后，bdHolder 实例已经包含配置文件中配置的各种属性了，如 Class、name、id、alias 之类的属性。
		BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
		if (bdHolder != null) {
			// 当返回的 bdHolder 不为空的情况下若存在默认标签的子节点下再有自定义属性，还需要再次对自定义标签进行解析
			bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder);
			try {
				// Register the final decorated instance.
				// 解析完成后，需要对解析后的 bdHolder 进行注册，同样，注册操作委托给了 BeanDefinitionReaderUtils 的 registerBeanDefinition 方法。
				BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry());
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to register bean definition with name '" +
						bdHolder.getBeanName() + "'", ele, ex);
			}
			// Send registration event.
			// 最后发出响应事件，通知相关的监听器，这个 bean 已经加载完成了。
			getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder));
		}
	}
```

（时序图）



##### 2.1.1 解析 BeanDefinition

- 元素解析及信息提取操作：BeanDefinitionHolder bdHolder = delegate.parseBeanDefinitionElement(ele);
  - （1）提取元素中的 id 以及 name 属性
  - （2）进一步解析其他所有属性并统一封装至 GenericBeanDefinition 类型的实例中。
  - （3）如果检测到 bean 没有指定 beanName，那么使用默认规则为此 bean 生成 beanName。
  - （4）将获取到的信息封装到 BeanDefinitionHolder 的实例中。

```java
public BeanDefinitionHolder parseBeanDefinitionElement(Element ele) {
		return parseBeanDefinitionElement(ele, null);
	}

@Nullable
	public BeanDefinitionHolder parseBeanDefinitionElement(Element ele, @Nullable BeanDefinition containingBean) {
		// （1）解析 id 和 name 属性
		String id = ele.getAttribute(ID_ATTRIBUTE);
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);

		// 分割 name 属性
		List<String> aliases = new ArrayList<>();
		if (StringUtils.hasLength(nameAttr)) {
			String[] nameArr = StringUtils.tokenizeToStringArray(nameAttr, MULTI_VALUE_ATTRIBUTE_DELIMITERS);
			aliases.addAll(Arrays.asList(nameArr));
		}

		String beanName = id;
		if (!StringUtils.hasText(beanName) && !aliases.isEmpty()) {
			beanName = aliases.remove(0);
			if (logger.isDebugEnabled()) {
				logger.debug("No XML 'id' specified - using '" + beanName +
						"' as bean name and " + aliases + " as aliases");
			}
		}

		if (containingBean == null) {
			checkNameUniqueness(beanName, aliases, ele);
		}
		
		// （2）进一步解析其他所有属性并同意封装至 GenericBeanDefinition 类型的实例中。
		AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);
		if (beanDefinition != null) {
			if (!StringUtils.hasText(beanName)) {
				try {
					// （3）如果不存在 beanName 那么根据 Spring 中提供的命名规则为当前 bean 生成对应的 beanName
					if (containingBean != null) {
						beanName = BeanDefinitionReaderUtils.generateBeanName(
								beanDefinition, this.readerContext.getRegistry(), true);
					}
					else {
						beanName = this.readerContext.generateBeanName(beanDefinition);
						// Register an alias for the plain bean class name, if still possible,
						// if the generator returned the class name plus a suffix.
						// This is expected for Spring 1.2/2.0 backwards compatibility.
						String beanClassName = beanDefinition.getBeanClassName();
						if (beanClassName != null &&
								beanName.startsWith(beanClassName) && beanName.length() > beanClassName.length() &&
								!this.readerContext.getRegistry().isBeanNameInUse(beanClassName)) {
							aliases.add(beanClassName);
						}
					}
					if (logger.isDebugEnabled()) {
						logger.debug("Neither XML 'id' nor 'name' specified - " +
								"using generated bean name [" + beanName + "]");
					}
				}
				catch (Exception ex) {
					error(ex.getMessage(), ele);
					return null;
				}
			}
			String[] aliasesArray = StringUtils.toStringArray(aliases);
			// （4）将获取到的信息封装到 BeanDefinitionHolder 的实例中
			return new BeanDefinitionHolder(beanDefinition, beanName, aliasesArray);
		}

		return null;
	}
```

- 步骤（2）对标签其他属性的解析过程：AbstractBeanDefinition beanDefinition = parseBeanDefinitionElement(ele, beanName, containingBean);

```java
	@Nullable
	public AbstractBeanDefinition parseBeanDefinitionElement(
			Element ele, String beanName, @Nullable BeanDefinition containingBean) {

		this.parseState.push(new BeanEntry(beanName));

		String className = null;
		// 解析 Class 属性
		if (ele.hasAttribute(CLASS_ATTRIBUTE)) {
			className = ele.getAttribute(CLASS_ATTRIBUTE).trim();
		}
		String parent = null;
		// 解析 parent 属性
		if (ele.hasAttribute(PARENT_ATTRIBUTE)) {
			parent = ele.getAttribute(PARENT_ATTRIBUTE);
		}

		try {
			// 创建用于承载属性的 AbstractBeanDefinition 类型的 GenericBeanDefinition
			AbstractBeanDefinition bd = createBeanDefinition(className, parent);

			// 硬编码解析默认 bean 的各种属性
			parseBeanDefinitionAttributes(ele, beanName, containingBean, bd);
			// 提取 description
			bd.setDescription(DomUtils.getChildElementValueByTagName(ele, DESCRIPTION_ELEMENT));

			// 解析元数据
			parseMetaElements(ele, bd);
			// 解析 lookup-method 属性
			parseLookupOverrideSubElements(ele, bd.getMethodOverrides());
			// 解析 replaced-method 属性
			parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

			// 解析构造函数参数
			parseConstructorArgElements(ele, bd);
			// 解析 property 子元素
			parsePropertyElements(ele, bd);
			// 解析 qualifier 子元素
			parseQualifierElements(ele, bd);

			bd.setResource(this.readerContext.getResource());
			bd.setSource(extractSource(ele));

			return bd;
		}
		catch (ClassNotFoundException ex) {
			error("Bean class [" + className + "] not found", ele, ex);
		}
		catch (NoClassDefFoundError err) {
			error("Class that bean class [" + className + "] depends on not found", ele, err);
		}
		catch (Throwable ex) {
			error("Unexpected failure during bean definition parsing", ele, ex);
		}
		finally {
			this.parseState.pop();
		}

		return null;
	}
```

###### 1.创建用于属性承载的 BeanDefinition

- BeanDefinition 是一个接口，在 Spring 中存在三种实现：RootBeanDefinition、ChildBeanDefinition 和 GenericBeanDefinition，三种实现均继承了 AbstractBeanDefinition。
- Spring 通过 BeanDefinition 将配置文件中的 <bean> 配置信息转换为容器的内部表示，并将这些 BeanDefinition 注册到 BeanDefinitionRegistry 中。Spring 容器的 BeanDefinitionRegistry 就像是 Spring 配置信息的内存数据库，主要是以 map 的形式保存，后续操作直接从 BeanDefinitionRegistry 中读取配置信息。
- 要解析属性首先要创建用于承载属性的实例，也就是创建 GenericBeanDefinition 类型的实例。而 AbstractBeanDefinition bd = createBeanDefinition(className, parent); 的作用就是实现此功能。

```java
protected AbstractBeanDefinition createBeanDefinition(@Nullable String className, @Nullable String parentName)
			throws ClassNotFoundException {

		return BeanDefinitionReaderUtils.createBeanDefinition(
				parentName, className, this.readerContext.getBeanClassLoader());
	}
```

BeanDefinitionReaderUtils.java

```java
	public static AbstractBeanDefinition createBeanDefinition(
			@Nullable String parentName, @Nullable String className, @Nullable ClassLoader classLoader) throws ClassNotFoundException {

		GenericBeanDefinition bd = new GenericBeanDefinition();
		// parentName 可能为空
		bd.setParentName(parentName);
		if (className != null) {
			if (classLoader != null) {
				// 如果 classLoader 不为空，则使用与传入的 classLoader 同一虚拟机加载类对象，
				// 否则只是记录 className
				bd.setBeanClass(ClassUtils.forName(className, classLoader));
			} else {
				bd.setBeanClassName(className);
			}
		}
		return bd;
	}
```



###### 2.解析各种属性

- 当创建了 bean 信息的承载实例后，即可以通过 parseBeanDefinitionAttributes(ele, beanName,
  			containingBean, bd) 方法进行 bean 信息的各种属性解析。属性包括：
  - scope
  - singleton
  - abstract
  - lazy-init
  - autowire
  - depends-on
  - autowire-candidate
  - primary
  - init-method
  - destroy-method
  - factory-method
  - factory-bean

```java
	public AbstractBeanDefinition parseBeanDefinitionAttributes(Element ele, String beanName,
			@Nullable BeanDefinition containingBean, AbstractBeanDefinition bd) {
		// 解析 singleton 属性与之前不同，直接报错
		if (ele.hasAttribute(SINGLETON_ATTRIBUTE)) {
			error("Old 1.x 'singleton' attribute in use - upgrade to 'scope' declaration", ele);
		}
		// 解析 Scope 属性
		else if (ele.hasAttribute(SCOPE_ATTRIBUTE)) {
			bd.setScope(ele.getAttribute(SCOPE_ATTRIBUTE));
		}
		else if (containingBean != null) {
			// Take default from containing bean in case of an inner bean definition.
			// 在嵌入 BeanDefinition 情况下且没有单独指定 scope 属性则使用父类默认的属性
			bd.setScope(containingBean.getScope());
		}

		// 解析 abstract 属性
		if (ele.hasAttribute(ABSTRACT_ATTRIBUTE)) {
			bd.setAbstract(TRUE_VALUE.equals(ele.getAttribute(ABSTRACT_ATTRIBUTE)));
		}

		// 解析 lazy-init 属性
		String lazyInit = ele.getAttribute(LAZY_INIT_ATTRIBUTE);
		if (isDefaultValue(lazyInit)) {
			lazyInit = this.defaults.getLazyInit();
		}
		// 若没有设置或设置成其他字符都会被设置为 false
		bd.setLazyInit(TRUE_VALUE.equals(lazyInit));

		// 解析 autowire 属性
		String autowire = ele.getAttribute(AUTOWIRE_ATTRIBUTE);
		bd.setAutowireMode(getAutowireMode(autowire));

		// 解析 depends-on 属性
		if (ele.hasAttribute(DEPENDS_ON_ATTRIBUTE)) {
			String dependsOn = ele.getAttribute(DEPENDS_ON_ATTRIBUTE);
			bd.setDependsOn(StringUtils.tokenizeToStringArray(dependsOn, MULTI_VALUE_ATTRIBUTE_DELIMITERS));
		}

		// 解析 autowire-candidate 属性
		String autowireCandidate = ele.getAttribute(AUTOWIRE_CANDIDATE_ATTRIBUTE);
		if (isDefaultValue(autowireCandidate)) {
			String candidatePattern = this.defaults.getAutowireCandidates();
			if (candidatePattern != null) {
				String[] patterns = StringUtils.commaDelimitedListToStringArray(candidatePattern);
				bd.setAutowireCandidate(PatternMatchUtils.simpleMatch(patterns, beanName));
			}
		}
		else {
			bd.setAutowireCandidate(TRUE_VALUE.equals(autowireCandidate));
		}

		// 解析 primary 属性
		if (ele.hasAttribute(PRIMARY_ATTRIBUTE)) {
			bd.setPrimary(TRUE_VALUE.equals(ele.getAttribute(PRIMARY_ATTRIBUTE)));
		}

		// 解析 init-method 属性
		if (ele.hasAttribute(INIT_METHOD_ATTRIBUTE)) {
			String initMethodName = ele.getAttribute(INIT_METHOD_ATTRIBUTE);
			bd.setInitMethodName(initMethodName);
		}
		else if (this.defaults.getInitMethod() != null) {
			bd.setInitMethodName(this.defaults.getInitMethod());
			bd.setEnforceInitMethod(false);
		}
	
		// 解析 destroy-method 属性
		if (ele.hasAttribute(DESTROY_METHOD_ATTRIBUTE)) {
			String destroyMethodName = ele.getAttribute(DESTROY_METHOD_ATTRIBUTE);
			bd.setDestroyMethodName(destroyMethodName);
		}
		else if (this.defaults.getDestroyMethod() != null) {
			bd.setDestroyMethodName(this.defaults.getDestroyMethod());
			bd.setEnforceDestroyMethod(false);
		}

		// 解析 factory-method 属性
		if (ele.hasAttribute(FACTORY_METHOD_ATTRIBUTE)) {
			bd.setFactoryMethodName(ele.getAttribute(FACTORY_METHOD_ATTRIBUTE));
		}
		// 解析 factory-bean 属性
		if (ele.hasAttribute(FACTORY_BEAN_ATTRIBUTE)) {
			bd.setFactoryBeanName(ele.getAttribute(FACTORY_BEAN_ATTRIBUTE));
		}

		return bd;
	}
```

###### 3.解析子元素 meta

- meta 属性的使用：不会体现在 bean 的属性中，而是一个额外的声明，当需要使用里面的信息的时候可以通过 BeanDefinition 的 getAttribute(key) 方法进行获取。

```java
	public void parseMetaElements(Element ele, BeanMetadataAttributeAccessor attributeAccessor) {
		// 获取当前节点的所有子元素
		NodeList nl = ele.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			// 提取 meta
			if (isCandidateElement(node) && nodeNameEquals(node, META_ELEMENT)) {
				Element metaElement = (Element) node;
				String key = metaElement.getAttribute(KEY_ATTRIBUTE);
				String value = metaElement.getAttribute(VALUE_ATTRIBUTE);
				// 使用 key、value 构造 BeanMetadataAttribute
				BeanMetadataAttribute attribute = new BeanMetadataAttribute(key, value);
				attribute.setSource(extractSource(metaElement));
				// 记录信息
				attributeAccessor.addMetadataAttribute(attribute);
			}
		}
	}
```

###### 4.解析子元素 lookup-method

- lookup-method：获取器注入。获取器注入是一种特殊的方法注入，它是把一个方法声明为返回某种类型的 bean，但实际要返回的 bean 是在配置文件里面配置的，此方法可用于在设计有些可插拔的功能上，接触程序依赖。
- 解析 lookup-method 属性的方法是 parseLookupOverrideSubElements(ele, bd.getMethodOverrides());   与 parseMetaElements 的代码大同小异。

```java
	public void parseLookupOverrideSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			// 仅当在 Spring 默认 bean 的子元素下且为 lookup-method 时有效
			if (isCandidateElement(node) && nodeNameEquals(node, LOOKUP_METHOD_ELEMENT)) {
				Element ele = (Element) node;
				// 获取要修饰的方法
				String methodName = ele.getAttribute(NAME_ATTRIBUTE);
				// 获取配置返回的 bean
				String beanRef = ele.getAttribute(BEAN_ELEMENT);
				LookupOverride override = new LookupOverride(methodName, beanRef);
				override.setSource(extractSource(ele));
				// 记录信息
				overrides.addOverride(override);
			}
		}
	}
```

###### 5.解析子元素 replaced-method

- replaced-method：方法替换。可以在运行时运用新的方法替换现有的方法。与 look-up 不同的是，replaced-method 不但可以动态地替换返回实体 bean，而且还能动态地更改原有方法的逻辑。
- replaced-method 元素的提取方法为：parseReplacedMethodSubElements(ele, bd.getMethodOverrides());

```java
	public void parseReplacedMethodSubElements(Element beanEle, MethodOverrides overrides) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			// 仅当在 Spring 默认 bean 的子元素下且为 <replaced-method 时有效
			if (isCandidateElement(node) && nodeNameEquals(node, REPLACED_METHOD_ELEMENT)) {
				Element replacedMethodEle = (Element) node;
				// 提取要替换的旧的方法
				String name = replacedMethodEle.getAttribute(NAME_ATTRIBUTE);
				// 提取对应的新的替换方法
				String callback = replacedMethodEle.getAttribute(REPLACER_ATTRIBUTE);
				ReplaceOverride replaceOverride = new ReplaceOverride(name, callback);
				// Look for arg-type match elements.
				List<Element> argTypeEles = DomUtils.getChildElementsByTagName(replacedMethodEle, ARG_TYPE_ELEMENT);
				for (Element argTypeEle : argTypeEles) {
					// 记录参数
					String match = argTypeEle.getAttribute(ARG_TYPE_MATCH_ATTRIBUTE);
					match = (StringUtils.hasText(match) ? match : DomUtils.getTextValue(argTypeEle));
					if (StringUtils.hasText(match)) {
						replaceOverride.addTypeIdentifier(match);
					}
				}
				replaceOverride.setSource(extractSource(replacedMethodEle));
				overrides.addOverride(replaceOverride);
			}
		}
	}
```

- 无论是 look-up 还是 replaced-method 都是构造了一个 MethodOverride，并最终记录在了 AbstractBeanDefinition 中的 methodOverrides 属性中。



###### 6.解析子元素 constructor-arg

- 构造函数的配置示例：

```xml
<beans>
    <!-- 默认的情况下是按照参数的顺序注入，当指定 index 索引后就可以改变注入参数的顺序 -->
    <bean id= "helloBean" class = "com.HelloBean">
    	<constructor-arg index = "0">
            <value>张三</value>
        </constructor-arg>
        <constructor-arg index = "0">
            <value>你好</value>
        </constructor-arg>
    </bean>
    ....
</beans>
```



- 对于 constructor-arg 子元素的解析，Spring 是通过 parseConstructorArgElements(ele, bd); 方法实现的。

```java
	public void parseConstructorArgElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, CONSTRUCTOR_ARG_ELEMENT)) {
				// 解析 constructor-arg
				parseConstructorArgElement((Element) node, bd);
			}
		}
	}
```

- 上面的方法提取所有的 constructor-arg，然后进行解析，而具体的解析被放置在了另一个函数parseConstructorArgElement((Element) node, bd); 中。
- 代码逻辑
  - 提取 constructor-arg 上必要的属性（index、type、name）
  - 如果配置中指定了 index 属性
    - 解析 constructor-arg 的子元素
    - 使用 ConstructorArgumentValues.ValueHolder 来封装解析出来的元素。
    - 将 type、name 和 index 属性一并封装在 ConstructorArgumentValues.ValueHolder 类型中并添加至当前 BeanDefinition 的 constructorArgumentValues 的 **indexedArgumentValues** 属性中。
  - 如果没有指定 index 属性
    - 解析 constructor-arg 的子元素
    - 使用 ConstructorArgumentValues.ValueHolder 来封装解析出来的元素。
    - 将 type、name 和 index 属性一并封装在 ConstructorArgumentValues.ValueHolder 类型中并添加至当前 BeanDefinition 的 constructorArgumentValues 的 **genericArgumentValues** 属性中。

```java
public void parseConstructorArgElement(Element ele, BeanDefinition bd) {
		// 提取 index 属性
		String indexAttr = ele.getAttribute(INDEX_ATTRIBUTE);
		// 提取 type 属性
		String typeAttr = ele.getAttribute(TYPE_ATTRIBUTE);
		// 提取 name 属性
		String nameAttr = ele.getAttribute(NAME_ATTRIBUTE);
		if (StringUtils.hasLength(indexAttr)) {
			try {
				int index = Integer.parseInt(indexAttr);
				if (index < 0) {
					error("'index' cannot be lower than 0", ele);
				}
				else {
					try {
						this.parseState.push(new ConstructorArgumentEntry(index));
						// 解析 ele 对应的属性元素（）
						Object value = parsePropertyValue(ele, bd, null);
						ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
						if (StringUtils.hasLength(typeAttr)) {
							valueHolder.setType(typeAttr);
						}
						if (StringUtils.hasLength(nameAttr)) {
							valueHolder.setName(nameAttr);
						}
						valueHolder.setSource(extractSource(ele));
						// 不允许重复指定相同参数
						if (bd.getConstructorArgumentValues().hasIndexedArgumentValue(index)) {
							error("Ambiguous constructor-arg entries for index " + index, ele);
						}
						else {
							// 添加信息
							bd.getConstructorArgumentValues().addIndexedArgumentValue(index, valueHolder);
						}
					}
					finally {
						this.parseState.pop();
					}
				}
			}
			catch (NumberFormatException ex) {
				error("Attribute 'index' of tag 'constructor-arg' must be an integer", ele);
			}
		}
		else {
			// 没有 index 属性则忽略去属性，自动寻找
			try {
				this.parseState.push(new ConstructorArgumentEntry());
				Object value = parsePropertyValue(ele, bd, null);
				ConstructorArgumentValues.ValueHolder valueHolder = new ConstructorArgumentValues.ValueHolder(value);
				if (StringUtils.hasLength(typeAttr)) {
					valueHolder.setType(typeAttr);
				}
				if (StringUtils.hasLength(nameAttr)) {
					valueHolder.setName(nameAttr);
				}
				valueHolder.setSource(extractSource(ele));
				// 添加信息
                bd.getConstructorArgumentValues().addGenericArgumentValue(valueHolder);
			}
			finally {
				this.parseState.pop();
			}
		}
	}
```



- 解析构造函数配置中子元素的过程，通过 parsePropertyValue(ele, bd, null); 方法实现。

```java
	@Nullable
	public Object parsePropertyValue(Element ele, BeanDefinition bd, @Nullable String propertyName) {
		String elementName = (propertyName != null ?
				"<property> element for property '" + propertyName + "'" :
				"<constructor-arg> element");

		// Should only have one child element: ref, value, list, etc.
		// 一个属性只能对应一种类型：ref、value、list 等
		NodeList nl = ele.getChildNodes();
		Element subElement = null;
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			// 对应 description 或者 meta 不处理
			if (node instanceof Element && !nodeNameEquals(node, DESCRIPTION_ELEMENT) &&
					!nodeNameEquals(node, META_ELEMENT)) {
				// Child element is what we're looking for.
				if (subElement != null) {
					error(elementName + " must not contain more than one sub-element", ele);
				}
				else {
					subElement = (Element) node;
				}
			}
		}
		
		// 解析 constructor-arg 上的 ref 属性
		boolean hasRefAttribute = ele.hasAttribute(REF_ATTRIBUTE);
		// 解析 constructor-arg 上的 value 属性
		boolean hasValueAttribute = ele.hasAttribute(VALUE_ATTRIBUTE);
		if ((hasRefAttribute && hasValueAttribute) ||
				((hasRefAttribute || hasValueAttribute) && subElement != null)) {
			/*
				在 constructor-arg 上不存在：
					1、同时既有 ref 属性又有 value 属性；
					2、存在 ref 属性或者 value 属性且又有子元素。
			 */
			error(elementName +
					" is only allowed to contain either 'ref' attribute OR 'value' attribute OR sub-element", ele);
		}

		if (hasRefAttribute) {
			// ref 属性的处理
			String refName = ele.getAttribute(REF_ATTRIBUTE);
			if (!StringUtils.hasText(refName)) {
				error(elementName + " contains empty 'ref' attribute", ele);
			}
			// 使用 RuntimeBeanReference 封装对应的 ref 名称
			RuntimeBeanReference ref = new RuntimeBeanReference(refName);
			ref.setSource(extractSource(ele));
			return ref;
		}
		else if (hasValueAttribute) {
			// value 属性的处理，使用 TypedStringValue 封装 value
			TypedStringValue valueHolder = new TypedStringValue(ele.getAttribute(VALUE_ATTRIBUTE));
			valueHolder.setSource(extractSource(ele));
			return valueHolder;
		}
		else if (subElement != null) {
			// 解析子元素
			return parsePropertySubElement(subElement, bd);
		}
		else {
			// Neither child element nor "ref" or "value" attribute found.
			// 既没有 ref 也没有 value 也没有子元素，Spring 报错
			error(elementName + " must specify a ref or value", ele);
			return null;
		}
	}
```

- 解析 constructor-arg 的子元素，示例如下。解析函数为 parsePropertySubElement(subElement, bd);

```xml
<constructor-arg>
    <map>
        <entry key = "key" value = "value" />
    </map>
</constructor-arg>
```

```java
	@Nullable
	public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd) {
		return parsePropertySubElement(ele, bd, null);
	}


	@Nullable
	public Object parsePropertySubElement(Element ele, @Nullable BeanDefinition bd, @Nullable String defaultValueType) {
		if (!isDefaultNamespace(ele)) {
			return parseNestedCustomElement(ele, bd);
		}
		else if (nodeNameEquals(ele, BEAN_ELEMENT)) {
			BeanDefinitionHolder nestedBd = parseBeanDefinitionElement(ele, bd);
			if (nestedBd != null) {
				nestedBd = decorateBeanDefinitionIfRequired(ele, nestedBd, bd);
			}
			return nestedBd;
		}
		else if (nodeNameEquals(ele, REF_ELEMENT)) {
			// A generic reference to any name of any bean.
			String refName = ele.getAttribute(BEAN_REF_ATTRIBUTE);
			boolean toParent = false;
			if (!StringUtils.hasLength(refName)) {
				// A reference to the id of another bean in a parent context.
				// 解析 parent
				refName = ele.getAttribute(PARENT_REF_ATTRIBUTE);
				toParent = true;
				if (!StringUtils.hasLength(refName)) {
					error("'bean' or 'parent' is required for <ref> element", ele);
					return null;
				}
			}
			if (!StringUtils.hasText(refName)) {
				error("<ref> element contains empty target attribute", ele);
				return null;
			}
			RuntimeBeanReference ref = new RuntimeBeanReference(refName, toParent);
			ref.setSource(extractSource(ele));
			return ref;
		}
		// 对 idref 元素的解析
		else if (nodeNameEquals(ele, IDREF_ELEMENT)) {
			return parseIdRefElement(ele);
		}
		// 对 value 子元素的解析
		else if (nodeNameEquals(ele, VALUE_ELEMENT)) {
			return parseValueElement(ele, defaultValueType);
		}
		// 对 null 子元素的解析
		else if (nodeNameEquals(ele, NULL_ELEMENT)) {
			// It's a distinguished null value. Let's wrap it in a TypedStringValue
			// object in order to preserve the source location.
			TypedStringValue nullHolder = new TypedStringValue(null);
			nullHolder.setSource(extractSource(ele));
			return nullHolder;
		}
		else if (nodeNameEquals(ele, ARRAY_ELEMENT)) {
			// 解析 array 子元素
			return parseArrayElement(ele, bd);
		}
		else if (nodeNameEquals(ele, LIST_ELEMENT)) {
			// 解析 list 子元素
			return parseListElement(ele, bd);
		}
		else if (nodeNameEquals(ele, SET_ELEMENT)) {
			// 解析 set 子元素
			return parseSetElement(ele, bd);
		}
		else if (nodeNameEquals(ele, MAP_ELEMENT)) {
			// 解析 map 子元素
			return parseMapElement(ele, bd);
		}
		else if (nodeNameEquals(ele, PROPS_ELEMENT)) {
			// 解析 props 子元素
			return parsePropsElement(ele);
		}
		else {
			error("Unknown property sub-element: [" + ele.getNodeName() + "]", ele);
			return null;
		}
	}
```



###### 7.解析子元素 property

- property 属性的用法：

```xml
<bean id = "test" class = "test.TestClass">
	<property name = "testStr" value = "aaa" />
</bean>

<bean id = "test" class = "test.TestClass">
	<property name = "p">
        <list>
            <value>aa</value>
            <value>bb</value>
        </list>
    </property>
</bean>
```

- Spring 使用 parsePropertyElements(ele, bd); 方法解析 property 属性，具体的解析过程如下：提取所有的 property 属性，然后调用 parsePropertyElement((Element) node, bd); 方法处理。

```java
	public void parsePropertyElements(Element beanEle, BeanDefinition bd) {
		NodeList nl = beanEle.getChildNodes();
		for (int i = 0; i < nl.getLength(); i++) {
			Node node = nl.item(i);
			if (isCandidateElement(node) && nodeNameEquals(node, PROPERTY_ELEMENT)) {
				parsePropertyElement((Element) node, bd);
			}
		}
	}
```

```java
public void parsePropertyElement(Element ele, BeanDefinition bd) {
		// 获取配置元素中 name 的值
		String propertyName = ele.getAttribute(NAME_ATTRIBUTE);
		if (!StringUtils.hasLength(propertyName)) {
			error("Tag 'property' must have a 'name' attribute", ele);
			return;
		}
		this.parseState.push(new PropertyEntry(propertyName));
		try {
			// 不允许多次对同一属性配置
			if (bd.getPropertyValues().contains(propertyName)) {
				error("Multiple 'property' definitions for property '" + propertyName + "'", ele);
				return;
			}
			// 使用 parsePropertyValue(ele, bd, propertyName); 解析 value 属性
			Object val = parsePropertyValue(ele, bd, propertyName);
			// 将返回值使用 PropertyValue 封装
			PropertyValue pv = new PropertyValue(propertyName, val);
			parseMetaElements(ele, pv);
			pv.setSource(extractSource(ele));
			// 将 PropertyValue 记录在 BeanDefinition 中的 propertyValues 属性中
			bd.getPropertyValues().addPropertyValue(pv);
		}
		finally {
			this.parseState.pop();
		}
	}
```





###### 8.解析子元素 qualifier 

- 对于 Qualifier 元素的获取，更多的是通过注解的形式。解析过程与之前大同小异。



##### 2.1.2 AbstractBeanDefinition 属性

- **上述完成了 XML 文档到 GenericBeanDefinition 的转换，即从这里开始，XML 中所有的配置都可以在 GenericBeanDefinition 的实例类中找到对应的配置。**	GenericBeanDefinition 只是子类实现，大部分的通用属性都保存在了 AbstractBeanDefinition 中，AbstractBeanDefinition 的属性如下：

```java
    public abstract class AbstractBeanDefinition extends BeanMetadataAttributeAccessor
            implements BeanDefinition, Cloneable {

        // 静态变量及 final 常量
        public static final String SCOPE_DEFAULT = "";
        public static final int AUTOWIRE_NO = AutowireCapableBeanFactory.AUTOWIRE_NO;
        public static final int AUTOWIRE_BY_NAME = AutowireCapableBeanFactory.AUTOWIRE_BY_NAME;
        public static final int AUTOWIRE_BY_TYPE = AutowireCapableBeanFactory.AUTOWIRE_BY_TYPE;
        public static final int AUTOWIRE_CONSTRUCTOR = AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR;
        @Deprecated
        public static final int AUTOWIRE_AUTODETECT = AutowireCapableBeanFactory.AUTOWIRE_AUTODETECT;
        public static final int DEPENDENCY_CHECK_NONE = 0;
        public static final int DEPENDENCY_CHECK_OBJECTS = 1;
        public static final int DEPENDENCY_CHECK_SIMPLE = 2;
        public static final int DEPENDENCY_CHECK_ALL = 3;
        public static final String INFER_METHOD = "(inferred)";


        @Nullable
        private volatile Object beanClass;

        // bean 的作用范围，对应 bean 属性 scope
        @Nullable
        private String scope = SCOPE_DEFAULT;

        // 是否是抽象，对应 bean 属性 abstract
        private boolean abstractFlag = false;

        // 是否延迟加载，对应 bean 属性 lazy-init
        private boolean lazyInit = false;

        // 自动注入模式，对应 bean 属性 autowire
        private int autowireMode = AUTOWIRE_NO;

        // 依赖检查， Spring 3.0 后弃用这个属性
        private int dependencyCheck = DEPENDENCY_CHECK_NONE;

        // 用来表示一个 bean 的实例化依靠另一个 bean 先实例化，对应 bean 属性 depends-on
        @Nullable
        private String[] dependsOn;

        // autowire-candidate 属性设置为 false，这样容器在查找自动装配对象时，将不考虑该 bean，即它不会被考虑作为其他 bean 自动装配的候选者，但是该 bean 本身还是可以使用自动装配来注入其他 bean 的。
        private boolean autowireCandidate = true;

        // 自动装配时当出现多个 bean 候选者时，将作为首选者，对应 bean 属性 primary
        private boolean primary = false;

        // 用于记录 Qualifier，对应子元素 qualifier
        private final Map<String, AutowireCandidateQualifier> qualifiers = new LinkedHashMap<>();

        @Nullable
        private Supplier<?> instanceSupplier;

        // 允许访问非公开的构造器和方法，程序设置
        private boolean nonPublicAccessAllowed = true;

        // 是否以一种宽松的模式解析构造函数，默认为 true，程序设置
        private boolean lenientConstructorResolution = true;

        // 对应 bean 属性 factory-bean，用法：
        // <bean id = "instanceFactoryBean" class = "example.InstanceFactoryBean"/>
        // <bean id = "currentTime" factory-bean = "instanceFactoryBean" factory-method="createTime" />
        @Nullable
        private String factoryBeanName;

        // 对应 bean 属性 factory-method
        @Nullable
        private String factoryMethodName;

        // 记录构造函数注入属性，对应 bean 属性 constructor-arg
        @Nullable
        private ConstructorArgumentValues constructorArgumentValues;

        // 普通属性集合，记录 property 属性的 value
        @Nullable
        private MutablePropertyValues propertyValues;

        // 方法重写的持有者，记录 lookup-method、replaced-method 元素
        private MethodOverrides methodOverrides = new MethodOverrides();

        // 初始化方法，对应 bean 属性 init-method
        @Nullable
        private String initMethodName;

        // 销毁方法，对应 bean 属性 destroy-method
        @Nullable
        private String destroyMethodName;

        // 是否执行 init-method，程序设置
        private boolean enforceInitMethod = true;

        // 是否执行 destroy-method，程序设置
        private boolean enforceDestroyMethod = true;

        // 是否是用户定义的而不是应用程序本身定义的，创建 AOP 时候为 true，程序设置
        private boolean synthetic = false;

        // 定义这个 bean 的应用，APPLICATION：用户，INFRASTRUCTURE：完全内部使用，SUPPORT：某些复杂配置的一部分。程序设置
        private int role = BeanDefinition.ROLE_APPLICATION;

        // bean 的描述信息
        @Nullable
        private String description;

        // 这个 bean 定义的资源
        @Nullable
        private Resource resource;

		...省略 set/get 方法
    }
```

##### 2.1.3 解析默认标签中的自定义标签元素

- Spring 的解析方法：bdHolder = delegate.decorateBeanDefinitionIfRequired(ele, bdHolder)。使用场景如下：

```xml
<bean id = "test" class = "test.MyClass">
    <mybean:user username = "aaa" />
</bean>
```

- 这里的自定义类型针对的是属性，而在下一章中的自定义标签是针对 bean 的。代码逻辑如下：

```java
    public BeanDefinitionHolder decorateBeanDefinitionIfRequired(Element ele, BeanDefinitionHolder originalDef) {
        	// 第三个参数为空；而第三个参数是父类 bean（BeanDefinition containingBd），当对某个嵌套配置进行分析时，这里需要传递父类 BeanDefinition。由源码得知这里是为了使用父类的 scope 属性，以备子类若没有设置 scope 时默认使用父类的属性。
            return decorateBeanDefinitionIfRequired(ele, originalDef, null);
        }



    public BeanDefinitionHolder decorateBeanDefinitionIfRequired(
                Element ele, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

            BeanDefinitionHolder finalDefinition = originalDef;

            // Decorate based on custom attributes first.
            NamedNodeMap attributes = ele.getAttributes();
            // 遍历所有的属性，看看是否有适用于修饰的属性
            for (int i = 0; i < attributes.getLength(); i++) {
                Node node = attributes.item(i);
                // 处理代码
                finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
            }

            // Decorate based on custom nested elements.
            NodeList children = ele.getChildNodes();
            // 遍历所有的子节点，看看是否有适用于修饰的子元素
            for (int i = 0; i < children.getLength(); i++) {
                Node node = children.item(i);
                if (node.getNodeType() == Node.ELEMENT_NODE) {
                    // 处理代码
                    finalDefinition = decorateIfRequired(node, finalDefinition, containingBd);
                }
            }
            return finalDefinition;
        }
```

- 分别对元素的所有属性以及子节点进行了 decorateIfRequired(node, finalDefinition, containingBd) 函数的调用

```java
    public BeanDefinitionHolder decorateIfRequired(
        Node node, BeanDefinitionHolder originalDef, @Nullable BeanDefinition containingBd) {

        // 获取自定义标签的命名空间
        String namespaceUri = getNamespaceURI(node);
        // 对于非默认标签进行修饰
        if (namespaceUri != null && !isDefaultNamespace(namespaceUri)) {
            // 根据命名空间找到对应的处理器
            NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
            if (handler != null) {
                // 进行修饰
                BeanDefinitionHolder decorated =
                    handler.decorate(node, originalDef, new ParserContext(this.readerContext, this, containingBd));
                if (decorated != null) {
                    return decorated;
                }
            }
            else if (namespaceUri.startsWith("http://www.springframework.org/")) {
                error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", node);
            }
            else {
                // A custom namespace, not to be handled by Spring - maybe "xml:...".
                if (logger.isDebugEnabled()) {
                    logger.debug("No Spring NamespaceHandler found for XML schema namespace [" + namespaceUri + "]");
                }
            }
        }
        return originalDef;
    }

    @Nullable
    public String getNamespaceURI(Node node) {
        return node.getNamespaceURI();
    }

	public boolean isDefaultNamespace(@Nullable String namespaceUri) {
		return (!StringUtils.hasLength(namespaceUri) || BEANS_NAMESPACE_URI.equals(namespaceUri));
	}
```



##### 2.1.4 注册解析后的 BeanDefinition

![](images\SpringIOC的初始化过程.png)

- 对于解析完得到的 BeanDefinition 唯一剩下的工作就是注册，也就是 processBeanDefinition 函数中的BeanDefinitionReaderUtils.registerBeanDefinition(bdHolder, getReaderContext().getRegistry()); 代码解析。

```java
	public static void registerBeanDefinition(
			BeanDefinitionHolder definitionHolder, BeanDefinitionRegistry registry)
			throws BeanDefinitionStoreException {

		// Register bean definition under primary name.
		// 使用 beanName 做唯一标识注册
		String beanName = definitionHolder.getBeanName();
		registry.registerBeanDefinition(beanName, definitionHolder.getBeanDefinition());

		// Register aliases for bean name, if any.
		// 注册所有别名
		String[] aliases = definitionHolder.getAliases();
		if (aliases != null) {
			for (String alias : aliases) {
				registry.registerAlias(beanName, alias);
			}
		}
	}
```

- 解析的 BeanDefinition 都会被注册到 BeanDefinitionRegistry 类型的 registry 中，而对于 BeanDefinition 的注册分成了两部分：通过 beanName 的注册以及通过别名的注册。



###### 1.通过 beanName 注册 BeanDefinition

DefaultListableBeanFactory.java

```java
	@Override
	public void registerBeanDefinition(String beanName, BeanDefinition beanDefinition)
			throws BeanDefinitionStoreException {

		Assert.hasText(beanName, "Bean name must not be empty");
		Assert.notNull(beanDefinition, "BeanDefinition must not be null");

		if (beanDefinition instanceof AbstractBeanDefinition) {
			try {
				/*
					注册前的最后一次校验，这里的校验不同于之前的 XMl 文件校验，
					主要是对于 AbstractBeanDefinition 属性中的 methodOverrides 校验，
					校验 methodOverrides 是否与工厂方法并存或者 methodOverrides 对应的方法根本不存在
				 */
				((AbstractBeanDefinition) beanDefinition).validate();
			}
			catch (BeanDefinitionValidationException ex) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Validation of bean definition failed", ex);
			}
		}

		BeanDefinition existingDefinition = this.beanDefinitionMap.get(beanName);
		// 处理注册已经注册的 beanName 情况
		if (existingDefinition != null) {
			// 如果对应的 beanName 已经注册且在配置中配置了 bean 不允许被覆盖，则抛出异常。
			if (!isAllowBeanDefinitionOverriding()) {
				throw new BeanDefinitionStoreException(beanDefinition.getResourceDescription(), beanName,
						"Cannot register bean definition [" + beanDefinition + "] for bean '" + beanName +
						"': There is already [" + existingDefinition + "] bound.");
			}
			else if (existingDefinition.getRole() < beanDefinition.getRole()) {
				// e.g. was ROLE_APPLICATION, now overriding with ROLE_SUPPORT or ROLE_INFRASTRUCTURE
				if (logger.isWarnEnabled()) {
					logger.warn("Overriding user-defined bean definition for bean '" + beanName +
							"' with a framework-generated bean definition: replacing [" +
							existingDefinition + "] with [" + beanDefinition + "]");
				}
			}
			else if (!beanDefinition.equals(existingDefinition)) {
				if (logger.isInfoEnabled()) {
					logger.info("Overriding bean definition for bean '" + beanName +
							"' with a different definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			else {
				if (logger.isDebugEnabled()) {
					logger.debug("Overriding bean definition for bean '" + beanName +
							"' with an equivalent definition: replacing [" + existingDefinition +
							"] with [" + beanDefinition + "]");
				}
			}
			// 注册 BeanDefinition
			this.beanDefinitionMap.put(beanName, beanDefinition);
		}
		else {
			if (hasBeanCreationStarted()) {
				// Cannot modify startup-time collection elements anymore (for stable iteration)
				// 因为 beanDefinitionMap 是全局变量，这里定会存在并发访问的情况
				synchronized (this.beanDefinitionMap) {
                    // 注册 BeanDefinition
					this.beanDefinitionMap.put(beanName, beanDefinition);
                    // 更新 this.beanDefinitionNames，记录 beanName
					List<String> updatedDefinitions = new ArrayList<>(this.beanDefinitionNames.size() + 1);
					updatedDefinitions.addAll(this.beanDefinitionNames);
					updatedDefinitions.add(beanName);
					this.beanDefinitionNames = updatedDefinitions;、
                    // 如果 this.manualSingletonNames中存在 beanName，则更新 this.manualSingletonNames
					if (this.manualSingletonNames.contains(beanName)) {
						Set<String> updatedSingletons = new LinkedHashSet<>(this.manualSingletonNames);
						updatedSingletons.remove(beanName);
						this.manualSingletonNames = updatedSingletons;
					}
				}
			}
			else {
				// Still in startup registration phase
				this.beanDefinitionMap.put(beanName, beanDefinition);
				this.beanDefinitionNames.add(beanName);
				this.manualSingletonNames.remove(beanName);
			}
			this.frozenBeanDefinitionNames = null;
		}

		if (existingDefinition != null || containsSingleton(beanName)) {
			resetBeanDefinition(beanName);
		}
	}
```



###### 2.通过别名注册 BeanDefinition

- 注册 alias 的步骤：
  - alias 与 beanName 相同情况处理。如果 beanName 与 alias 相同的话，不记录 alias，并删除对应的 alias。
  - 如果 name 已经存在，则已经存在 alias，不需要重复注册，直接返回
  - alias 覆盖处理。当 aliasName 已经使用并已经指向了另一 beanName，则需要用户的设置进行处理。
  - alias 循环检查。当 A ->B 存在时，，若再次出现 A->C->B 时候则会抛出异常。
  - 注册 alias。

SimpleAliasRegistry.java

```java
	@Override
	public void registerAlias(String name, String alias) {
		Assert.hasText(name, "'name' must not be empty");
		Assert.hasText(alias, "'alias' must not be empty");
		synchronized (this.aliasMap) {
			// 如果 beanName 与 alias 相同的话，不记录 alias，并删除对应的 alias
			if (alias.equals(name)) {
				this.aliasMap.remove(alias);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' ignored since it points to same name");
				}
			}
			else {
				String registeredName = this.aliasMap.get(alias);
				if (registeredName != null) {
					// alias 已经存在，不需要重复注册
					if (registeredName.equals(name)) {
						// An existing alias - no need to re-register
						return;
					}
					// 如果 alias 不允许被覆盖则抛出异常
					if (!allowAliasOverriding()) {
						throw new IllegalStateException("Cannot define alias '" + alias + "' for name '" +
								name + "': It is already registered for name '" + registeredName + "'.");
					}
					if (logger.isInfoEnabled()) {
						logger.info("Overriding alias '" + alias + "' definition for registered name '" +
								registeredName + "' with new target name '" + name + "'");
					}
				}
				// 当 A ->B 存在时，，若再次出现 A->C->B 时候则会抛出异常
				checkForAliasCircle(name, alias);
				this.aliasMap.put(alias, name);
				if (logger.isDebugEnabled()) {
					logger.debug("Alias definition '" + alias + "' registered for name '" + name + "'");
				}
			}
		}
	}
```



##### 2.1.5 通知监听器解析及注册完成

- 通过代码：getReaderContext().fireComponentRegistered(new BeanComponentDefinition(bdHolder)); 完成此工作，当需要对注册 BeanDefinition 时间进行监听时可以通过注册监听器的方式并将处理逻辑写入监听器中。





#### 2.2 alias 标签的解析

- alias 标签的解析过程和在 bean 中的 alias 解析大同小异，都是将别名与 beanName 组成一对注册至 registry 中。

DefaultBeanDefinitionDocumentReader.java

```java
protected void processAliasRegistration(Element ele) {
		// 获取 beanName
		String name = ele.getAttribute(NAME_ATTRIBUTE);
		// 获取 alias
		String alias = ele.getAttribute(ALIAS_ATTRIBUTE);
		boolean valid = true;
		if (!StringUtils.hasText(name)) {
			getReaderContext().error("Name must not be empty", ele);
			valid = false;
		}
		if (!StringUtils.hasText(alias)) {
			getReaderContext().error("Alias must not be empty", ele);
			valid = false;
		}
		if (valid) {
			try {
				// 注册 alias
				getReaderContext().getRegistry().registerAlias(name, alias);
			}
			catch (Exception ex) {
				getReaderContext().error("Failed to register alias '" + alias +
						"' for bean with name '" + name + "'", ele, ex);
			}
			// 别名注册后通知监听器做相应处理
			getReaderContext().fireAliasRegistered(name, alias, extractSource(ele));
		}
	}
```



#### 2.3 import 标签的解析

- import 标签的使用：applicationContext.xml 文件中使用 import 的方式导入有模块配置文件，以后若有新模块的加入，那就可以简单修改这个文件了。这样大大简化了配置后期维护的复杂度，并使配置模块化，易于管理。

```xml
<beans>
    <import resource = "customerContext.xml" />
    <import resource = "systemContet.xml" />
    ...
</beans>
```

- Spring 通过 importBeanDefinitionResource(Element ele) 方法解析 import 标签；解析步骤如下：
  - 获取 resource 属性所表示的路径；
  - 解析路径中的系统属性，格式如"${user.dir}"；
  - 判定 location 是绝对路径还是相对路径
  - 如果是绝对路径则递归调用 bean 的解析过程，进行另一次的解析。
  - 如果是相对路径则计算出绝对路径并进行解析。
  - 通知监听器，解析完成。

DefaultBeanDefinitionDocumentReader.java

```java
protected void importBeanDefinitionResource(Element ele) {
		// 获取 resource 属性
		String location = ele.getAttribute(RESOURCE_ATTRIBUTE);
		// 如果不存在 resource 属性则不作任何处理
		if (!StringUtils.hasText(location)) {
			getReaderContext().error("Resource location must not be empty", ele);
			return;
		}

		// Resolve system properties: e.g. "${user.dir}"
		// 解析系统属性，格式如："${user.dir}"
		location = getReaderContext().getEnvironment().resolveRequiredPlaceholders(location);

		Set<Resource> actualResources = new LinkedHashSet<>(4);

		// Discover whether the location is an absolute or relative URI
		// 判断 location 是绝对 URI 还是相对 URI
		boolean absoluteLocation = false;
		try {
			absoluteLocation = ResourcePatternUtils.isUrl(location) || ResourceUtils.toURI(location).isAbsolute();
		}
		catch (URISyntaxException ex) {
			// cannot convert to an URI, considering the location relative
			// unless it is the well-known Spring prefix "classpath*:"
		}

		// Absolute or relative?
		if (absoluteLocation) {
			// 如果是绝对 URI 则直接根据地址加载对应的配置文件
			try {
				int importCount = getReaderContext().getReader().loadBeanDefinitions(location, actualResources);
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from URL location [" + location + "]");
				}
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error(
						"Failed to import bean definitions from URL location [" + location + "]", ele, ex);
			}
		}
		else {
			// No URL -> considering resource location as relative to the current file.
			// 如果是相对地址则根据相对地址计算出绝对地址
			try {
				int importCount;
				// Resource 存在多个子实现类，如 VfsResource、FileSystemResource 等，
				// 而每个resource 的 createRelative 方法实现都不一样，
				// 所以这里先使用子类的方法尝试解析
				Resource relativeResource = getReaderContext().getResource().createRelative(location);
				if (relativeResource.exists()) {
					importCount = getReaderContext().getReader().loadBeanDefinitions(relativeResource);
					actualResources.add(relativeResource);
				}
				else {
					// 如果解析不成功，则使用默认的解析器 ResourcePatternResolver 进行解析
					String baseLocation = getReaderContext().getResource().getURL().toString();
					importCount = getReaderContext().getReader().loadBeanDefinitions(
							StringUtils.applyRelativePath(baseLocation, location), actualResources);
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Imported " + importCount + " bean definitions from relative location [" + location + "]");
				}
			}
			catch (IOException ex) {
				getReaderContext().error("Failed to resolve current resource location", ele, ex);
			}
			catch (BeanDefinitionStoreException ex) {
				getReaderContext().error("Failed to import bean definitions from relative location [" + location + "]",
						ele, ex);
			}
		}
		// 解析后进行监听器激活处理
		Resource[] actResArray = actualResources.toArray(new Resource[0]);
		getReaderContext().fireImportProcessed(location, actResArray, extractSource(ele));
	}
```



#### 2.4 嵌入式 beans 标签的解析

- 递归调用 beans 的解析过程。



### 3 自定义标签的解析

```java
protected void parseBeanDefinitions(Element root, BeanDefinitionParserDelegate delegate) {
		// 对 beans 的处理
		if (delegate.isDefaultNamespace(root)) {
			NodeList nl = root.getChildNodes();
			for (int i = 0; i < nl.getLength(); i++) {
				Node node = nl.item(i);
				if (node instanceof Element) {
					Element ele = (Element) node;
					if (delegate.isDefaultNamespace(ele)) {
						// 对 bean 的处理，默认标签的解析
						parseDefaultElement(ele, delegate);
					}
					else {
						// 对 bean 的处理，自定义标签的解析
						delegate.parseCustomElement(ele);
					}
				}
			}
		}
		else {
			delegate.parseCustomElement(root);
		}
	}
```



#### 3.1 自定义标签使用

- 需要为系统提供可配置化支持时，Spring 提供了可扩展的 Schema 的支持，扩展 Spring 自定义标签配置大致需要以下几个步骤（前提是把 Spring 的 Core 包加入项目中）：
  - 创建一个需要扩展的组件。
  - 定义一个 XSD 文件描述组件内容。
  - 创建一个文件，实现 BeanDefinitionParser 接口，用来解析 XSD 文件中的定义和组件定义。
  - 创建一个 Handler 文件，扩展自 NamespaceHandlerSupport，目的是将组件注册到 Spring 容器。
  - 编写 Spring.handlers 和 Spring.schemas 文件。
- 在 Spring 中自定义标签非常常用，如事务标签：tx(<tx:annotation-driven>)。

#### 3.2 自定义标签解析

- 自定义标签的解析过程如下：

```java
	@Nullable
	public BeanDefinition parseCustomElement(Element ele) {
		return parseCustomElement(ele, null);
	}


	// containingBd 为父类 bean，对顶层元素的解析应设置为 null
	@Nullable
	public BeanDefinition parseCustomElement(Element ele, @Nullable BeanDefinition containingBd) {
		// 获取对应的命名空间
		String namespaceUri = getNamespaceURI(ele);
		if (namespaceUri == null) {
			return null;
		}
		// 根据命名空间找到对应的（自定义的） NamespaceHandler
		NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri);
		if (handler == null) {
			error("Unable to locate Spring NamespaceHandler for XML schema namespace [" + namespaceUri + "]", ele);
			return null;
		}
		// 调用自定义的 NamespaceHandler 进行解析
		return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd));
	}
```



##### 3.2.1 获取标签的命名空间

- 如何提取对应元素的命名空间不需要我们亲自实现，在 org.w3c.dom.Node 中已经提供了方法供我们直接调用：

```java
	@Nullable
	public String getNamespaceURI(Node node) {
		return node.getNamespaceURI();
	}
```



##### 3.2.2 提取自定义标签处理器

- 有了命名空间就可以进行 NamespaceHandler 的提取了。通过 NamespaceHandler handler = this.readerContext.getNamespaceHandlerResolver().resolve(namespaceUri); 进行提取。在 readerContext 初始化的时候其属性 namespaceHandlerResolver 已经被初始化为了 DefaultNamespaceHandlerResolver 的实例，所以，这里调用的 resolve 方法其实调用的是 DefaultNamespaceHandlerResolver 类中的方法。

DefaultNamespaceHandlerResolver.java

```java
	@Override
	@Nullable
	public NamespaceHandler resolve(String namespaceUri) {
		// 获取所有已经配置的 Handler 映射
		Map<String, Object> handlerMappings = getHandlerMappings();
		// 根据命名空间找到对应的信息
		Object handlerOrClassName = handlerMappings.get(namespaceUri);
		if (handlerOrClassName == null) {
			return null;
		}
		else if (handlerOrClassName instanceof NamespaceHandler) {
			// 已经做过解析的情况，直接从缓存读取
			return (NamespaceHandler) handlerOrClassName;
		}
		else {
			// 没有做过解析，返回的是类路径
			String className = (String) handlerOrClassName;
			try {
				// 使用反射将类路径转化为类
				Class<?> handlerClass = ClassUtils.forName(className, this.classLoader);
				if (!NamespaceHandler.class.isAssignableFrom(handlerClass)) {
					throw new FatalBeanException("Class [" + className + "] for namespace [" + namespaceUri +
							"] does not implement the [" + NamespaceHandler.class.getName() + "] interface");
				}
				// 初始化类
				NamespaceHandler namespaceHandler = (NamespaceHandler) BeanUtils.instantiateClass(handlerClass);
				// 调用自定义的 NamespaceHandler 的 init 方法
				namespaceHandler.init();
				// 记录在 handlerMappings 缓存中
				handlerMappings.put(namespaceUri, namespaceHandler);
				return namespaceHandler;
			}
			catch (ClassNotFoundException ex) {
				throw new FatalBeanException("Could not find NamespaceHandler class [" + className +
						"] for namespace [" + namespaceUri + "]", ex);
			}
			catch (LinkageError err) {
				throw new FatalBeanException("Unresolvable class definition for NamespaceHandler class [" +
						className + "] for namespace [" + namespaceUri + "]", err);
			}
		}
	}
```

- 如果要使用自定义标签，其中一项必不可少的操作就是在 Spring.handlers 文件中配置命名空间与命名空间处理器的映射关系。只有这样， Spring 才能根据映射关系找到匹配的处理器。

```java
public class MyNamespaceHandler extends NamespaceHandlerSupport {
	public void init() {
		registerBeanDefinitionParser("user", new UserBeanDefinitionParser());
    }
}
```



- 当得到自定义命名空间处理后会马上执行 NamespaceHandler.init() 来进行自定义 BeanDefinitionParser 的注册。可以注册多个标签解析器。注册后，命名空间处理器就可以根据标签的不同来调用不同的解析器进行解析。
- getHandlerMappings() 的主要功能是读取 Spring.handlers 配置文件并将配置温家安缓存在 map 中。借助了工具类 PropertiesLoaderUtils 对属性 handlerMappingsLocation进行了配置文件的读取，handlerMappingsLocation 被默认初始化为“META-INF/Spring.handlers”。

```java
	private Map<String, Object> getHandlerMappings() {
		Map<String, Object> handlerMappings = this.handlerMappings;
		if (handlerMappings == null) {
			// 如果没有被缓存则开始进行缓存
			synchronized (this) {
				handlerMappings = this.handlerMappings;
				if (handlerMappings == null) {
					if (logger.isDebugEnabled()) {
						logger.debug("Loading NamespaceHandler mappings from [" + this.handlerMappingsLocation + "]");
					}
					try {
						// this.handlerMappingsLocation 在构造函数中已经被初始化为：META-INF/Spring.handlers
						Properties mappings =
								PropertiesLoaderUtils.loadAllProperties(this.handlerMappingsLocation, this.classLoader);
						if (logger.isDebugEnabled()) {
							logger.debug("Loaded NamespaceHandler mappings: " + mappings);
						}
						handlerMappings = new ConcurrentHashMap<>(mappings.size());
						// 将 Properties 格式文件合并到 Map 格式的 handlerMappings 中
						CollectionUtils.mergePropertiesIntoMap(mappings, handlerMappings);
						this.handlerMappings = handlerMappings;
					}
					catch (IOException ex) {
						throw new IllegalStateException(
								"Unable to load NamespaceHandler mappings from location [" + this.handlerMappingsLocation + "]", ex);
					}
				}
			}
		}
		return handlerMappings;
	}
```



##### 3.2.3 标签解析

- 得到了解析器以及要分析的元素后，Spring 就可以将解析工作委托给自定义解析器去解析。在 Spring 中的代码为：return handler.parse(ele, new ParserContext(this.readerContext, this, containingBd))。此时：
  - handler 被实例化成为了自定义的 MyNamespaceHandler 了，而 MyNamespaceHandler 也已经完成了初始化的工作，但是在我们实现的自定义命名空间处理器中并没有实现 parse 方法，所以推断，这个方法是在父类中实现的。
  - 查看父类 NamespaceHandlerSupport 中的 parse 方法：

NamespaceHandlerSupport.java

```java
	@Override
	@Nullable
	public BeanDefinition parse(Element element, ParserContext parserContext) {
		// 寻找解析器
		BeanDefinitionParser parser = findParserForElement(element, parserContext);
		// 进行解析操作
		return (parser != null ? parser.parse(element, parserContext) : null);
	}
```

- 解析过程首先是在寻找元素对应的解析器，进而调用解析器中的 parse 方法。即首先获取在 MyNamespaceHandler 类中的 init 方法中注册的对应的 UserBeanDefinitionParser 实例，然后调用 parse 方法进行进一步解析。

```java
	@Nullable
	private BeanDefinitionParser findParserForElement(Element element, ParserContext parserContext) {
		// 获取原色名称，也就是 <myname:user 中的 user，若在实例中，此时 localName 为 user
		String localName = parserContext.getDelegate().getLocalName(element);
		// 根据 localName 找到对应的解析器，即 UserBeanDefinitionParser
		BeanDefinitionParser parser = this.parsers.get(localName);
		if (parser == null) {
			parserContext.getReaderContext().fatal(
					"Cannot locate BeanDefinitionParser for element [" + localName + "]", element);
		}
		return parser;
	}
```

- 对于 parse 方法的处理：

AbstractBeanDefinitionParser.java

```java
	@Nullable
	public final BeanDefinition parse(Element element, ParserContext parserContext) {
		AbstractBeanDefinition definition = parseInternal(element, parserContext);
		if (definition != null && !parserContext.isNested()) {
			try {
				String id = resolveId(element, definition, parserContext);
				if (!StringUtils.hasText(id)) {
					parserContext.getReaderContext().error(
							"Id is required for element '" + parserContext.getDelegate().getLocalName(element)
									+ "' when used as a top-level tag", element);
				}
				String[] aliases = null;
				if (shouldParseNameAsAliases()) {
					String name = element.getAttribute(NAME_ATTRIBUTE);
					if (StringUtils.hasLength(name)) {
						aliases = StringUtils.trimArrayElements(StringUtils.commaDelimitedListToStringArray(name));
					}
				}
				// 将 AbstractBeanDefinition 转换成 BeanDefinitionHolder 并注册
				BeanDefinitionHolder holder = new BeanDefinitionHolder(definition, id, aliases);
				registerBeanDefinition(holder, parserContext.getRegistry());
				if (shouldFireEvents()) {
					// 需要通知监听器进行处理
					BeanComponentDefinition componentDefinition = new BeanComponentDefinition(holder);
					postProcessComponentDefinition(componentDefinition);
					parserContext.registerComponent(componentDefinition);
				}
			}
			catch (BeanDefinitionStoreException ex) {
				String msg = ex.getMessage();
				parserContext.getReaderContext().error((msg != null ? msg : ex.toString()), element);
				return null;
			}
		}
		return definition;
	}
```

- 上述函数大部分的代码是来处理将解析后的 AbstractBeanDefinition 转换成 BeanDefinitionHolder 并注册的功能，而将真正去做解析的事情委托给了函数 parseInternal(element, parserContext)

```java
	protected final AbstractBeanDefinition parseInternal(Element element, ParserContext parserContext) {
		BeanDefinitionBuilder builder = BeanDefinitionBuilder.genericBeanDefinition();
		String parentName = getParentName(element);
		if (parentName != null) {
			builder.getRawBeanDefinition().setParentName(parentName);
		}
		// 获取自定义标签中的 Class，此时会调用自定义解析器如 UserBeanDefinitionParser 中的 getBeanClass 方法
		Class<?> beanClass = getBeanClass(element);
		if (beanClass != null) {
			builder.getRawBeanDefinition().setBeanClass(beanClass);
		}
		else {
			// 若子类没有重写 getBeanClass 方法则尝试检查子类是否重写 getBeanClassName 方法
			String beanClassName = getBeanClassName(element);
			if (beanClassName != null) {
				builder.getRawBeanDefinition().setBeanClassName(beanClassName);
			}
		}
		builder.getRawBeanDefinition().setSource(parserContext.extractSource(element));
		BeanDefinition containingBd = parserContext.getContainingBeanDefinition();
		if (containingBd != null) {
			// Inner bean definition must receive same scope as containing bean.
			builder.setScope(containingBd.getScope());
		}
		if (parserContext.isDefaultLazyInit()) {
			// Default-lazy-init applies to custom bean definitions as well.
			builder.setLazyInit(true);
		}
		// 调用子类重写的 doParse 方法进行解析
		doParse(element, parserContext, builder);
		return builder.getBeanDefinition();
	}
```



### 4 bean 的加载

- bean 的加载过程所涉及的步骤如下：
  - 转换对应的 beanName。
    - 去除 FactoryBean 的修饰符，如 name = "&aa"，那么会首先去除 & 而使 name = "aa"；
    - 取指定 alias 所表示的最终 beanName，如别名 A 指向名称为 B 的 bean则返回 B；若别名 A 指向别名 B，别名 B 又指向名称为 C 的 bean，则返回 C。
  - 尝试从缓存中加载单例。
    - 单例在 Spring 的同一个容器内只会被创建一次，后续再获取 bean，就直接从单例缓存中获取。顺序如下：尝试从单例缓存（singletonObjects）中加载，如果加载不成功则再次尝试从 singletonFactories 中加载。主要为了避免循环依赖问题。
  - bean  的实例化。
    - 如果从缓存中得到了 bean 的原始状态，则需要对 bean 进行实例化。通过 bean = getObjectForBeanInstance(sharedInstance, name, beanName, null); 方法完成。
  - 原型模式的依赖检查。
    - 只有在单例情况下才会尝试解决循环依赖。当 isPrototypeCurrentlyInCreation(beanName) 为 true 时，抛出异常。
  - 检测 parentBeanFactory。
    -  
  - 将存储 XML 配置文件的 GenericBeanDefinition 转换为 RootBeanDefinition。
    - 因为从 XML 配置文件中读取到的 Bean 信息是存储在 GenericBeanDefinition 中的，而所有的 bean 后续处理都是针对于 RootBeanDefinition 的，所以这里需要进行一个转换，转换的同时如果父类 bean 不为空的话，则会一并合并父类的属性。
  - 寻找依赖。
    - 在 Spring 的加载顺序中，在初始化某一个 bean 的时候首先会初始化这个 bean 所对应的依赖。
  - 针对不同的 scope 进行 bean 的创建。
  - 类型转换。
    - 将返回的 bean 转换为 requiredType 所指定的类型。

AbstractBeanFactory.java

```java
	@Override
	public Object getBean(String name) throws BeansException {
		return doGetBean(name, null, null, false);
	}

	protected <T> T doGetBean(final String name, @Nullable final Class<T> requiredType,
			@Nullable final Object[] args, boolean typeCheckOnly) throws BeansException {

		// 提取对应的 beanName
		final String beanName = transformedBeanName(name);
		Object bean;

		// Eagerly check singleton cache for manually registered singletons.
		/*
		 *	检查缓存中或者实例工厂中是否有对应的实例。
		 * 	为什么首先要使用这段代码呢？
		 * 	因为在创建单例 bean 的时候会存在依赖注入的情况，而在创建依赖的时候为了避免循环依赖，
		 * 	Spring 创建 bean 的原则是不等 bean 创建完成就会将 bean 的 ObjectFactory 提早曝光，
		 * 	也就是将 ObjectFactory 加入到缓存中，一旦下一个创建时候需要上个 bean 则直接使用 ObjectFactory。
		 */
		// 直接尝试从单例缓存池（singletonObjects）获取或者
		// 从 singletonFactories 中的 objectFactory 中获取
		Object sharedInstance = getSingleton(beanName);
		if (sharedInstance != null && args == null) {
			if (logger.isDebugEnabled()) {
				if (isSingletonCurrentlyInCreation(beanName)) {
					logger.debug("Returning eagerly cached instance of singleton bean '" + beanName +
							"' that is not fully initialized yet - a consequence of a circular reference");
				}
				else {
					logger.debug("Returning cached instance of singleton bean '" + beanName + "'");
				}
			}
			// 返回对应的实例，有时候存在诸如 BeanFactory 的情况并不是返回实例本身
			// 而是返回指定方法返回的实例。
			bean = getObjectForBeanInstance(sharedInstance, name, beanName, null);
		}

		else {
			// Fail if we're already creating this bean instance:
			// We're assumably within a circular reference.
			// 只有在单例模式下才会尝试解决循环依赖，原型模式下直接抛出异常。
			if (isPrototypeCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(beanName);
			}

			// Check if bean definition exists in this factory.
			BeanFactory parentBeanFactory = getParentBeanFactory();
			// 如果 beanDefinitionMap 中也就是在所有已经加载的类中不包括 beanName，
			// 则尝试从 parentBeanFactory 中检测
			if (parentBeanFactory != null && !containsBeanDefinition(beanName)) {
				// Not found -> check parent.
				String nameToLookup = originalBeanName(name);
				if (parentBeanFactory instanceof AbstractBeanFactory) {
					return ((AbstractBeanFactory) parentBeanFactory).doGetBean(
							nameToLookup, requiredType, args, typeCheckOnly);
				}
				// 递归到 BeanFactory 中寻找
				else if (args != null) {
					// Delegation to parent with explicit args.
					return (T) parentBeanFactory.getBean(nameToLookup, args);
				}
				else {
					// No args -> delegate to standard getBean method.
					return parentBeanFactory.getBean(nameToLookup, requiredType);
				}
			}

			// 如果不是仅仅做类型检查而是创建 bean，则需要进行记录
			if (!typeCheckOnly) {
				markBeanAsCreated(beanName);
			}

			try {
				// 将存储 XML 配置文件的 GenericBeanDefinition 转换为 RootBeanDefinition。
				// 转换的同时如果父类 bean 不为空的话，则会一并合并父类的属性。
				final RootBeanDefinition mbd = getMergedLocalBeanDefinition(beanName);
				checkMergedBeanDefinition(mbd, beanName, args);

				// Guarantee initialization of beans that the current bean depends on.
				// 若存在依赖则需要递归实例化依赖的 bean
				String[] dependsOn = mbd.getDependsOn();
				if (dependsOn != null) {
					for (String dep : dependsOn) {
						if (isDependent(beanName, dep)) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"Circular depends-on relationship between '" + beanName + "' and '" + dep + "'");
						}
						registerDependentBean(dep, beanName);
						try {
							getBean(dep);
						}
						catch (NoSuchBeanDefinitionException ex) {
							throw new BeanCreationException(mbd.getResourceDescription(), beanName,
									"'" + beanName + "' depends on missing bean '" + dep + "'", ex);
						}
					}
				}

				// Create bean instance.
				// 实例化依赖的 bean 后便可以实例化 mbd 本身了
				// singleton 模式的创建
				if (mbd.isSingleton()) {
					sharedInstance = getSingleton(beanName, () -> {
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});
					bean = getObjectForBeanInstance(sharedInstance, name, beanName, mbd);
				}

				else if (mbd.isPrototype()) {
					// It's a prototype -> create a new instance.
					// prototype 模式的创建（new）
					Object prototypeInstance = null;
					try {
						beforePrototypeCreation(beanName);
						prototypeInstance = createBean(beanName, mbd, args);
					}
					finally {
						afterPrototypeCreation(beanName);
					}
					bean = getObjectForBeanInstance(prototypeInstance, name, beanName, mbd);
				}

				else {
					// 指定的 scope 上实例化 bean
					String scopeName = mbd.getScope();
					final Scope scope = this.scopes.get(scopeName);
					if (scope == null) {
						throw new IllegalStateException("No Scope registered for scope name '" + scopeName + "'");
					}
					try {
						Object scopedInstance = scope.get(beanName, () -> {
							beforePrototypeCreation(beanName);
							try {
								return createBean(beanName, mbd, args);
							}
							finally {
								afterPrototypeCreation(beanName);
							}
						});
						bean = getObjectForBeanInstance(scopedInstance, name, beanName, mbd);
					}
					catch (IllegalStateException ex) {
						throw new BeanCreationException(beanName,
								"Scope '" + scopeName + "' is not active for the current thread; consider " +
								"defining a scoped proxy for this bean if you intend to refer to it from a singleton",
								ex);
					}
				}
			}
			catch (BeansException ex) {
				cleanupAfterBeanCreationFailure(beanName);
				throw ex;
			}
		}

		// Check if required type matches the type of the actual bean instance.
		// 检查需要的类型是否符合 bean 的实际类型
		if (requiredType != null && !requiredType.isInstance(bean)) {
			try {
				T convertedBean = getTypeConverter().convertIfNecessary(bean, requiredType);
				if (convertedBean == null) {
					throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
				}
				return convertedBean;
			}
			catch (TypeMismatchException ex) {
				if (logger.isDebugEnabled()) {
					logger.debug("Failed to convert bean '" + name + "' to required type '" +
							ClassUtils.getQualifiedName(requiredType) + "'", ex);
				}
				throw new BeanNotOfRequiredTypeException(name, requiredType, bean.getClass());
			}
		}
		return (T) bean;
	}
	
```



#### 4.1 FactoryBean 的使用



#### 4.2 缓存中获取单例 bean

- getSingleton(String beanName, boolean allowEarlyReference)  方法首先尝试从 singletonObjects 里面获取实例，如果获取不到再从 earlySingletonObjects 里面获取，如果还获取不到，再尝试从 singletonFactories 中获取 beanName 对应的 ObjectFactory，然后调用这个 ObjectFactory 的 getObject 方法来创建 bean，并收到 earlySingletonObjects 中，并且从 singletonFactories 中 remove 掉这个 ObjectFactory，而对于后续的所有内存操作都只是为了循环依赖检测时候使用，也就是在 allowEarlyReference 为 true 的情况下会使用。
- 存储 bean 的不同的 map（三级缓存）：
  - singletonObjects（一级缓存、单例缓存池）：用于保存 BeanName 和创建 bean 实例之间的关系 beanName ----> bean instance。
  - singletonFactories（二级缓存）：用于保存 beanName 和创建 bean 的工厂之间的关系，      beanName ----> ObjectFactory
  - earlySingletonObjects（三级缓存）：也是保存 beanName 和创建 bean 实例之间的关系，与 singletonObjects 的不同之处在于，当一个单例 bean 被放到这里后，那么当 bean 还在创建过程中，就可以通过 getBean 方法获取到了，其目的是用来检测循环引用。
  - registeredSingletons：用来保存当前所有已注册的 bean。

DefaultSingletonBeanRegistry.java

```java
	public Object getSingleton(String beanName) {
		// 参数 true 设置标识允许早期依赖 allowEarlyReference
		return getSingleton(beanName, true);
	}

	@Nullable
	protected Object getSingleton(String beanName, boolean allowEarlyReference) {
		// 检查单例缓存中是否存在实例
		Object singletonObject = this.singletonObjects.get(beanName);

		if (singletonObject == null && isSingletonCurrentlyInCreation(beanName)) {
			// 如果为空，且 beanName 正在创建中，则锁定全局变量并进行处理
			synchronized (this.singletonObjects) {
				// 如果此 bean 正在加载则不处理
				singletonObject = this.earlySingletonObjects.get(beanName);
				if (singletonObject == null && allowEarlyReference) {
					// 当某些方法需要提前初始化的时候则
					// 将对应的 ObjectFactory 初始化策略存储在 singletonFactories 中
					ObjectFactory<?> singletonFactory = this.singletonFactories.get(beanName);
					if (singletonFactory != null) {
						// 调用预先设定的 getObject 方法
						singletonObject = singletonFactory.getObject();
						// 记录在缓存中，earlySingletonObjects 和 singletonFactories 互斥
						// 二级缓存：singletonFactories 三级缓存：earlySingletonObjects
						this.earlySingletonObjects.put(beanName, singletonObject);
						this.singletonFactories.remove(beanName);
					}
				}
			}
		}
		return singletonObject;
	}
```



#### 4.3 从 bean 的实例中获取对象

- 无论是从缓存中获取到的 bean 还是通过不同的 scope 策略加载的 bean 都只是最原始的 bean 状态，并不一定是最终要的 bean。假如需要对工厂 bean 进行处理，那么这里得到的其实是工厂 bean 的初始状态，但是真正需要的是工厂 bean 中定义的 factory-method 方法中返回的 bean，而 bean = getObjectForBeanInstance(sharedInstance, name, beanName, null); 方法就是完成这个工作的。
- 

```java
	protected Object getObjectForBeanInstance(
			Object beanInstance, String name, String beanName, @Nullable RootBeanDefinition mbd) {

		// Don't let calling code try to dereference the factory if the bean isn't a factory.
		if (BeanFactoryUtils.isFactoryDereference(name)) {
			if (beanInstance instanceof NullBean) {
				return beanInstance;
			}
			// 如果指定的 name 是工厂相关（以 & 为前缀）且 beanInstance 又不是 FactoryBean 类型则验证不通过
			if (!(beanInstance instanceof FactoryBean)) {
				throw new BeanIsNotAFactoryException(beanName, beanInstance.getClass());
			}
		}

		// Now we have the bean instance, which may be a normal bean or a FactoryBean.
		// If it's a FactoryBean, we use it to create a bean instance, unless the
		// caller actually wants a reference to the factory.
		// 现在我们有了个 bean 实例，这个实例可能回事正常的 bean 或者是 FactoryBean
		// 如果是 FactoryBean，我们使用它创建一个 bean 实例，但是如果用户想要直接获取工厂实例
			// 而不是工厂的 getObject 方法对应的实例那么传入的 name 应该加入前缀 &
			if (!(beanInstance instanceof FactoryBean) || BeanFactoryUtils.isFactoryDereference(name)) {
				return beanInstance;
			}

			// 加载 FactoryBean
			Object object = null;
			if (mbd == null) {
				// 尝试从缓存中加载 bean
				object = getCachedObjectForFactoryBean(beanName);
			}
			if (object == null) {
				// Return bean instance from factory.
				// 到这里已经明确知道 beanInstance 一定是 FactoryBean 类型。
				FactoryBean<?> factory = (FactoryBean<?>) beanInstance;
				// Caches object obtained from FactoryBean if it is a singleton.
				// containsBeanDefinition 检测 BeanDefinitionMap（所有已经加载的类中）是否定义 beanName
				if (mbd == null && containsBeanDefinition(beanName)) {
					// 将存储 XML 配置文件的 GenericBeanDefinition 转换为 RootBeanDefinition，
					// 如果指定 BeanName 是子 bean 的话同时会合并父类的相关属性
					mbd = getMergedLocalBeanDefinition(beanName);
				}
			// 是否是用户定义的而不是应用程序本身定义的
			boolean synthetic = (mbd != null && mbd.isSynthetic());
			object = getObjectFromFactoryBean(factory, beanName, !synthetic);
		}
		return object;
	}
```

- getObjectForBeanInstance 所做的工作：
  - 对 FactoryBean 正确性的验证。
  - 对非 FactoryBean 不作任何处理。
  - 对 bean 进行转换。
  - 将从 Factory 中解析 bean 的工作委托给getObjectFromFactoryBean(factory, beanName, !synthetic);。

FactoryBeanRegistrySupport.java

```java
protected Object getObjectFromFactoryBean(FactoryBean<?> factory, String beanName, boolean shouldPostProcess) {
		// 如果是单例模式
		if (factory.isSingleton() && containsSingleton(beanName)) {
			// 单例模式双重检验
			synchronized (getSingletonMutex()) {
				Object object = this.factoryBeanObjectCache.get(beanName);
				if (object == null) {
					object = doGetObjectFromFactoryBean(factory, beanName);
					// Only post-process and store if not put there already during getObject() call above
					// (e.g. because of circular reference processing triggered by custom getBean calls)
					Object alreadyThere = this.factoryBeanObjectCache.get(beanName);
					if (alreadyThere != null) {
						object = alreadyThere;
					}
					else {
						// 创建 Object
						if (shouldPostProcess) {
							if (isSingletonCurrentlyInCreation(beanName)) {
								// Temporarily return non-post-processed object, not storing it yet..
								return object;
							}
							beforeSingletonCreation(beanName);
							try {
                                // 调用 ObjectFactory 的后置处理器
								object = postProcessObjectFromFactoryBean(object, beanName);
							}
							catch (Throwable ex) {
								throw new BeanCreationException(beanName,
										"Post-processing of FactoryBean's singleton object failed", ex);
							}
							finally {
								afterSingletonCreation(beanName);
							}
						}
						if (containsSingleton(beanName)) {
							this.factoryBeanObjectCache.put(beanName, object);
						}
					}
				}
				return object;
			}
		}
		else {
			Object object = doGetObjectFromFactoryBean(factory, beanName);
			if (shouldPostProcess) {
				try {
					// 调用 ObjectFactory 的后置处理器
					object = postProcessObjectFromFactoryBean(object, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(beanName, "Post-processing of FactoryBean's object failed", ex);
				}
			}
			return object;
		}
	}
```

- 原型模式通过 doGetObjectFromFactoryBean(factory, beanName); 方法创建 bean。

```java
	private Object doGetObjectFromFactoryBean(final FactoryBean<?> factory, final String beanName)
			throws BeanCreationException {

		Object object;
		try {
			// 需要权限验证
			if (System.getSecurityManager() != null) {
				AccessControlContext acc = getAccessControlContext();
				try {
					object = AccessController.doPrivileged((PrivilegedExceptionAction<Object>) factory::getObject, acc);
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 直接调用 getObject 方法
				object = factory.getObject();
			}
		}
		catch (FactoryBeanNotInitializedException ex) {
			throw new BeanCurrentlyInCreationException(beanName, ex.toString());
		}
		catch (Throwable ex) {
			throw new BeanCreationException(beanName, "FactoryBean threw exception on object creation", ex);
		}

		// Do not accept a null value for a FactoryBean that's not fully
		// initialized yet: Many FactoryBeans just return null then.
		// Object 为 null 时的处理。
		if (object == null) {
			if (isSingletonCurrentlyInCreation(beanName)) {
				throw new BeanCurrentlyInCreationException(
						beanName, "FactoryBean which is currently in creation returned null from getObject");
			}
			object = new NullBean();
		}
		return object;
	}
```

- 创建或获取了 bean 之后，没有直接返回，而是通过 postProcessObjectFromFactoryBean(object, beanName) 方法进行后置处理。进入 AbstractAutowireCapableBeanFactory 类的 postProcessObjectFromFactoryBean 方法：
  - Spring 获取 bean 的规则：尽可能保证所有 bean 初始化后都会调用注册的 BeanPostProcessor 的 postProcessAfterInitialization 方法进行处理，在实际开发过程中可以对此特性设计自己的业务逻辑。 

```java
	@Override
	protected Object postProcessObjectFromFactoryBean(Object object, String beanName) {
		return applyBeanPostProcessorsAfterInitialization(object, beanName);
	}

	@Override
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
				Object current = processor.postProcessAfterInitialization(result, beanName);
				if (current == null) {
					return result;
				}
			result = current;
		}
		return result;
	}
```



#### 4.4 获取单例

- 如果缓存中不存在已经加载的单例 bean，就需要从头开始 bean 的加载过程，Spring中使用 getSingleton(String beanName, ObjectFactory<?> singletonFactory) 方法实现 bean 的加载过程。
- 

DefaultSingletonBeanRegistry.java

```java
	public Object getSingleton(String beanName, ObjectFactory<?> singletonFactory) {
		Assert.notNull(beanName, "Bean name must not be null");
		// 全局变量需要同步
		// 单例模式，双重检验
		synchronized (this.singletonObjects) {
			Object singletonObject = this.singletonObjects.get(beanName);
			if (singletonObject == null) {
				if (this.singletonsCurrentlyInDestruction) {
					throw new BeanCreationNotAllowedException(beanName,
							"Singleton bean creation not allowed while singletons of this factory are in destruction " +
							"(Do not request a bean from a BeanFactory in a destroy method implementation!)");
				}
				if (logger.isDebugEnabled()) {
					logger.debug("Creating shared instance of singleton bean '" + beanName + "'");
				}
				beforeSingletonCreation(beanName);
				boolean newSingleton = false;
				boolean recordSuppressedExceptions = (this.suppressedExceptions == null);
				if (recordSuppressedExceptions) {
					this.suppressedExceptions = new LinkedHashSet<>();
				}
				try {
					// 初始化 bean
					singletonObject = singletonFactory.getObject();
					newSingleton = true;
				}
				catch (IllegalStateException ex) {
					// Has the singleton object implicitly appeared in the meantime ->
					// if yes, proceed with it since the exception indicates that state.
					singletonObject = this.singletonObjects.get(beanName);
					if (singletonObject == null) {
						throw ex;
					}
				}
				catch (BeanCreationException ex) {
					if (recordSuppressedExceptions) {
						for (Exception suppressedException : this.suppressedExceptions) {
							ex.addRelatedCause(suppressedException);
						}
					}
					throw ex;
				}
				finally {
					if (recordSuppressedExceptions) {
						this.suppressedExceptions = null;
					}
					afterSingletonCreation(beanName);
				}
				// 如果是新的单例 bean，则加入缓存
				if (newSingleton) {
					addSingleton(beanName, singletonObject);
				}
			}
			return singletonObject;
		}
	}
```

- 代码使用了回调方法，使得程序可以在单例创建的前后做一些准备及处理操作，而真正获取单例 bean 的方法是在 ObjectFactory 类型的实例 singletonFactory 中实现的。上述代码的准备及处理操作包括如下内容：
  - （1）检查缓存是否已经加载过。
  - （2）如果没有加载，则记录 beanName 的正在加载状态。
  - （3）加载单例前记录加载状态：beforeSingletonCreation(String beanName)  方法。通过 this.singletonsCurrentlyInCreation.add(beanName) 将当前正要创建的 bean 记录在缓存中，这样便可以对循环依赖进行检测。
  - （4）通过调用参数传入的 ObjectFactory 的个体 Object 方法实例化 bean。singletonFactory.getObject();
  - （5）加载单例后的处理方法调用。同步骤（3）的记录状态相似，当 bean 加载结束后需要移除缓存中对该 bean 的正在加载状态的记录。
  - （6）将结果记录至缓存并删除加载 bean 过程中所记录的各种辅助状态。
  - （7）返回处理结果。

```java
	// 第三步
	protected void beforeSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.add(beanName)) {
			throw new BeanCurrentlyInCreationException(beanName);
		}
	}
	// 第四步
	sharedInstance = getSingleton(beanName, () -> {
						// 传入的是 getObject 方法的实现
						try {
							return createBean(beanName, mbd, args);
						}
						catch (BeansException ex) {
							// Explicitly remove instance from singleton cache: It might have been put there
							// eagerly by the creation process, to allow for circular reference resolution.
							// Also remove any beans that received a temporary reference to the bean.
							destroySingleton(beanName);
							throw ex;
						}
					});

	// 第五步
	protected void afterSingletonCreation(String beanName) {
		if (!this.inCreationCheckExclusions.contains(beanName) && !this.singletonsCurrentlyInCreation.remove(beanName)) {
			throw new IllegalStateException("Singleton '" + beanName + "' isn't currently in creation");
		}
	}

	// 第六步
	protected void addSingleton(String beanName, Object singletonObject) {
		synchronized (this.singletonObjects) {
			this.singletonObjects.put(beanName, singletonObject);
			this.singletonFactories.remove(beanName);
			this.earlySingletonObjects.remove(beanName);
			this.registeredSingletons.add(beanName);
		}
	}
```



#### 4.5 准备创建 bean

- ObjectFactory 的核心部分调用了 createBean 的方法，即真正创建 bean 的逻辑在 createBean(beanName, mbd, args); 方法中。
- Spring 的规律：真正干活的函数是以 do 开头的，比如 doGetObjectFromFactoryBean；而给我们错觉的函数，比如 getObjectFromFactoryBean 只是从全局角度做些统筹工作。

AbstractAutowireCapableBeanFactory.java

```java
	@Override
	protected Object createBean(String beanName, RootBeanDefinition mbd, @Nullable Object[] args)
			throws BeanCreationException {

		if (logger.isDebugEnabled()) {
			logger.debug("Creating instance of bean '" + beanName + "'");
		}
		RootBeanDefinition mbdToUse = mbd;

		// Make sure bean class is actually resolved at this point, and
		// clone the bean definition in case of a dynamically resolved Class
		// which cannot be stored in the shared merged bean definition.
		Class<?> resolvedClass = resolveBeanClass(mbd, beanName);
		if (resolvedClass != null && !mbd.hasBeanClass() && mbd.getBeanClassName() != null) {
			mbdToUse = new RootBeanDefinition(mbd);
			mbdToUse.setBeanClass(resolvedClass);
		}

		// Prepare method overrides.
		// 验证及准备覆盖的方法
		try {
			mbdToUse.prepareMethodOverrides();
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanDefinitionStoreException(mbdToUse.getResourceDescription(),
					beanName, "Validation of method overrides failed", ex);
		}

		try {
			// Give BeanPostProcessors a chance to return a proxy instead of the target bean instance.
			Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
			if (bean != null) {
				return bean;
			}
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbdToUse.getResourceDescription(), beanName,
					"BeanPostProcessor before instantiation of bean failed", ex);
		}

		try {
			Object beanInstance = doCreateBean(beanName, mbdToUse, args);
			if (logger.isDebugEnabled()) {
				logger.debug("Finished creating instance of bean '" + beanName + "'");
			}
			return beanInstance;
		}
		catch (BeanCreationException | ImplicitlyAppearedSingletonException ex) {
			// A previously detected exception with proper bean creation context already,
			// or illegal singleton state to be communicated up to DefaultSingletonBeanRegistry.
			throw ex;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					mbdToUse.getResourceDescription(), beanName, "Unexpected exception during bean creation", ex);
		}
	}
```

- createBean 函数完成的具体步骤及功能。
  - （1）根据设置的 class 属性或者根据 className 来解析 Class。 resolveBeanClass(mbd, beanName);
  - （2）对 override 属性进行标记及验证。针对 lookup-method 和 replaced-method 配置。 mbdToUse.prepareMethodOverrides();
  - （3）应用初始化前的后置处理器，解析指定 bean  是否存在初始化前的短路操作。                          Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
  - （4）创建 bean。Object beanInstance = doCreateBean(beanName, mbdToUse, args);



##### 4.5.1 处理 override 属性。

- 实现原理：在 bean 实例化的时候如果检测到存在 methodOverrides 属性，会动态地为当前 bean 生成代理并使用对应的拦截器为 bean 做增强处理。（相关逻辑实现在 bean 的实例化部分详解。）

AbstractBeanDefinition.java

```java
	public void prepareMethodOverrides() throws BeanDefinitionValidationException {
		// Check that lookup methods exist and determine their overloaded status.
		if (hasMethodOverrides()) {
			getMethodOverrides().getOverrides().forEach(this::prepareMethodOverride);
		}
	}

	protected void prepareMethodOverride(MethodOverride mo) throws BeanDefinitionValidationException {
		// 获取对应类中对应方法名的个数
		int count = ClassUtils.getMethodCountForName(getBeanClass(), mo.getMethodName());
		if (count == 0) {
			throw new BeanDefinitionValidationException(
					"Invalid method override: no method with name '" + mo.getMethodName() +
					"' on class [" + getBeanClassName() + "]");
		}
		else if (count == 1) {
			// Mark override as not overloaded, to avoid the overhead of arg type checking.
			// 标记 MethodOverrides 暂未被覆盖，避免参数类型检查的开销。
			mo.setOverloaded(false);
		}
	}
```



##### 4.5.2 实例化的前置处理

- 在真正调用 doCreate 方法创建 bean 的实例前使用了 resolveBeforeInstantiation(beanName, mbdToUse) 方法对 BeanDefinition 中的属性做些前置处理。
- 而关键部分是提供了一个短路判断。当经过前置处理后返回的结果如果不为空，那么会直接略过后续的 bean 的创建而直接返回结果。这一特性容易被忽略，但起着重要作用，AOP 功能就是基于这里的判断的。

```java
	Object bean = resolveBeforeInstantiation(beanName, mbdToUse);
    // 短路判断
    if (bean != null) {
        return bean;
    }
```

- Object bean = resolveBeforeInstantiation(beanName, mbdToUse);——InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation 方法，和 BeanPostProcessor 的 postProcessAfterInitialization 方法。

```java
	protected Object resolveBeforeInstantiation(String beanName, RootBeanDefinition mbd) {
		Object bean = null;
		// 如果尚未被解析
		if (!Boolean.FALSE.equals(mbd.beforeInstantiationResolved)) {
			// Make sure bean class is actually resolved at this point.
			if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
				Class<?> targetType = determineTargetType(beanName, mbd);
				if (targetType != null) {
					bean = applyBeanPostProcessorsBeforeInstantiation(targetType, beanName);
					if (bean != null) {
						bean = applyBeanPostProcessorsAfterInitialization(bean, beanName);
					}
				}
			}
			mbd.beforeInstantiationResolved = (bean != null);
		}
		return bean;
	}
```



###### 1.实例化前的后置处理器应用

- bean 的实例化前调用，也就是将 AbstractBeanDefinition 转换为 BeanWrapper 前的处理。逻辑：对每一个 InstantiationAwareBeanPostProcessor 的子类使用 postProcessBeforeInstantiation 方法返回结果。

```java
	protected Object applyBeanPostProcessorsBeforeInstantiation(Class<?> beanClass, String beanName) {
		for (BeanPostProcessor bp : getBeanPostProcessors()) {
			if (bp instanceof InstantiationAwareBeanPostProcessor) {
				InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
				Object result = ibp.postProcessBeforeInstantiation(beanClass, beanName);
				if (result != null) {
					return result;
				}
			}
		}
		return null;
	}
```



###### 2.实例化后的后置处理器应用

- Spring 中的规则是：在 bean  的初始化后尽可能保证将注册的后置处理器的 postProcessAfterInitialization 方法应用到该 bean 中，因为如果返回的 bean 不为空，则不会再次经历普通 bean 的创建过程（实例化），所以只能在这里应用后置处理器的 postProcessAfterInitialization 方法。

```java
	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
        // BeanPostProcessor 的 postProcessAfterInitialization 方法
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
				Object current = processor.postProcessAfterInitialization(result, beanName);
				if (current == null) {
					return result;
				}
			result = current;
		}
		return result;
	}
```



#### 4.6 循环依赖



##### 4.6.1 什么是循环依赖

-  循环依赖就是循环引用，是两个或多个 bean 相互之间持有对方。
- 循环调用是方法之间的环调用。循环调用是无法解决的，除非有终结条件，否则就是死循环，最终导致内存溢出错误。



##### 4.6.2 Spring 如何解决循环依赖

- 对于 “singleton” 作用域的 bean，可以通过 “setAllowCircularReferences(false)；”来禁用循环引用。

###### 1.构造器循环依赖

- 表示通过构造器注入构成的循环依赖，此依赖是无法解决的，只能抛出 BeanCurrentlyInCreationException 异常表示循环依赖。
- Spring 容器将每一个正在创建的 bean 标识符放在一个“当前创建 bean 池”中，bean 标识符在创建过程中将一直保持在这个池中，因此如果在创建 bean 的过程中发现自己已经在“当前创建 bean 池”里时，将抛出 BeanCurrentlyInCreationException  异常表示循环依赖；而对于创建完毕的 bean 将从“当前创建 bean 池”中清除掉。

###### 2.setter 循环依赖

- 表示通过 setter 注入方式构成的循环依赖。对于 setter 注入构成的依赖是通过 Spring 容器**提前暴露刚完成构造器注入但未完成其他步骤（如 setter 注入）的 bean 来完成的**，而且只能解决单例作用域的 bean 循环依赖。通过提前暴露一个单例工厂方法，从而使其他 bean 能引用到该 bean，代码如下：

AbstractAutowireCapableBeanFactory.java

```java
	doCreateBean() {
        ...
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
        ...
    }

```



###### 3.prototype 范围的依赖处理

- 对于 “prototype” 作用域 bean，Spring 容器无法完成依赖注入，因为 Spring 容器不进行缓存“prototype”作用域的 bean，因此无法提前暴露一个创建中的 bean。



#### 4.7 创建 bean

- 当经过 resolveBeforeInstantiation(beanName, mbdToUse); 方法后，程序有两个选择，**如果创建了代理或者说重写了 InstantiationAwareBeanPostProcessor 的 postProcessBeforeInstantiation 方法并在方法 postProcessBeforeInstantiation 改变了 bean，则直接返回就可以了**，否则需要进行常规的 bean 的创建。
- 常规 bean 的创建就是在 doCreateBean 中完成的。逻辑如下：
  - （1）如果是单例则需要首先从 factoryBeanInstanceCache 中清除缓存。
  - （2）实例化 bean，将 BeanDefinition 转换成 BeanWrapper。createBeanInstance 方法。转换过程如下：
    - 如果存在工厂方法则使用工厂方法进行初始化。
    - 一个类有多个构造函数，每个构造函数都有不同的参数，所以需要根据参数锁定构造函数并进行初始化。
    - 如果既不存在工厂方法也不存在带有参数的构造函数，则使用默认的构造函数进行 bean 的实例化。
  - （3）MergedBeanDefinitionPostProcessor 的应用。
    - bean 合并后的处理，Autowired 注解正是通过此方法实现诸如类型的预解析。
  - （4）依赖处理。
    - boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
      				isSingletonCurrentlyInCreation(beanName));
  - （5）属性填充。将所有属性填充至 bean 的实例中。populateBean。
  - （6）调用初始化方法，比如 init-method。initializeBean(beanName, exposedObject, mbd);
  - （7）循环依赖检查。
  - （8）注册 DisposableBean。如果配置了 destroy-method ，这里需要注册以便于在销毁的时候调用。registerDisposableBeanIfNecessary(beanName, bean, mbd);
  - （9）完成创建并返回

```java
	protected Object doCreateBean(final String beanName, final RootBeanDefinition mbd, final @Nullable Object[] args)
			throws BeanCreationException {

		// Instantiate the bean.
		BeanWrapper instanceWrapper = null;
		if (mbd.isSingleton()) {
			instanceWrapper = this.factoryBeanInstanceCache.remove(beanName);
		}
		if (instanceWrapper == null) {
			// 根据指定 bean 使用对应的策略创建新的实例，如：工厂方法、构造函数自动注入、简单初始化
			instanceWrapper = createBeanInstance(beanName, mbd, args);
		}
		final Object bean = instanceWrapper.getWrappedInstance();
		Class<?> beanType = instanceWrapper.getWrappedClass();
		if (beanType != NullBean.class) {
			mbd.resolvedTargetType = beanType;
		}

		// Allow post-processors to modify the merged bean definition.
		synchronized (mbd.postProcessingLock) {
			if (!mbd.postProcessed) {
				try {
					// 应用 MergedBeanDefinitionPostProcessor
					applyMergedBeanDefinitionPostProcessors(mbd, beanType, beanName);
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Post-processing of merged bean definition failed", ex);
				}
				mbd.postProcessed = true;
			}
		}

		// Eagerly cache singletons to be able to resolve circular references
		// even when triggered by lifecycle interfaces like BeanFactoryAware.
		// 是否需要提前曝光：单例 & 允许循环依赖 & 当前 bean 正在创建中，检测循环依赖
		boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
				isSingletonCurrentlyInCreation(beanName));
		if (earlySingletonExposure) {
			if (logger.isDebugEnabled()) {
				logger.debug("Eagerly caching bean '" + beanName +
						"' to allow for resolving potential circular references");
			}
			// 为避免后期循环依赖，可以在 bean 初始化完成前将创建实例的 ObjectFactory 加入工厂
			// getEarlyBeanReference：对 bean 再一次依赖引用，主要应用 SmartInstantiationAwareBeanPostProcessor，
			// AOP 就是在这里将 Advice 动态织入 bean 中，若没有则直接返回 bean，不作任何处理
			addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
		}

		// Initialize the bean instance.
		Object exposedObject = bean;
		try {
			// 对 bean 进行填充，将各个属性值注入，其中，可能存在依赖于其他 bean 的属性，则会递归初始依赖的 bean
			populateBean(beanName, mbd, instanceWrapper);
			// 调用初始化方法，比如 init-method
			exposedObject = initializeBean(beanName, exposedObject, mbd);
		}
		catch (Throwable ex) {
			if (ex instanceof BeanCreationException && beanName.equals(((BeanCreationException) ex).getBeanName())) {
				throw (BeanCreationException) ex;
			}
			else {
				throw new BeanCreationException(
						mbd.getResourceDescription(), beanName, "Initialization of bean failed", ex);
			}
		}

		if (earlySingletonExposure) {
			Object earlySingletonReference = getSingleton(beanName, false);
			// earlySingletonReference 只有在检测到有循环依赖的情况下才会不为空
			if (earlySingletonReference != null) {
				// 如果 earlySingletonReference 没有在初始化方法中被改变，也就是没有被增强
				if (exposedObject == bean) {
					exposedObject = earlySingletonReference;
				}
				else if (!this.allowRawInjectionDespiteWrapping && hasDependentBean(beanName)) {
					String[] dependentBeans = getDependentBeans(beanName);
					Set<String> actualDependentBeans = new LinkedHashSet<>(dependentBeans.length);
					for (String dependentBean : dependentBeans) {
						// 检测依赖
						if (!removeSingletonIfCreatedForTypeCheckOnly(dependentBean)) {
							actualDependentBeans.add(dependentBean);
						}
					}
					/*
					 *	因为 bean 创建后期所依赖的 bean 一定是已经创建的，
					 * 	actualDependentBeans 不为空则表示当前 bean 创建后其依赖的 bean
					 * 	却没有全部创建完，也就是说存在循环依赖
					 *
					 */
					if (!actualDependentBeans.isEmpty()) {
						throw new BeanCurrentlyInCreationException(beanName,
								"Bean with name '" + beanName + "' has been injected into other beans [" +
								StringUtils.collectionToCommaDelimitedString(actualDependentBeans) +
								"] in its raw version as part of a circular reference, but has eventually been " +
								"wrapped. This means that said other beans do not use the final version of the " +
								"bean. This is often the result of over-eager type matching - consider using " +
								"'getBeanNamesOfType' with the 'allowEagerInit' flag turned off, for example.");
					}
				}
			}
		}

		// Register bean as disposable.
		try {
			// 根据 scope 注册 bean
			registerDisposableBeanIfNecessary(beanName, bean, mbd);
		}
		catch (BeanDefinitionValidationException ex) {
			throw new BeanCreationException(
					mbd.getResourceDescription(), beanName, "Invalid destruction signature", ex);
		}

		return exposedObject;
	}
```



##### 4.7.1 创建 bean 的实例

- createBeanInstance 方法中 bean 的实例化的逻辑。
  - （1）如果存在 Supplier 回调，则调用 obtainFromSupplier() 进行初始化
  - （2）如果在 RootBeanDefinition 中存在 factoryMethodName 属性，或者说在配置文件中配置了 factory-method，那么 Spring 会尝试使用 instantiateUsingFactoryMethod(beanName, mbd, args); 方法根据 RootBeanDefinition 中的配置生成 bean 的实例。
  - （3）解析构造函数并进行构造函数的实例化。判断的过程是比较消耗性能的步骤，所以使用缓存机制。。
    - 首先判断缓存，如果缓存中存在，即已经解析过了，则直接使用已经解析了的，根据 constructorArgumentsResolved 参数来判断是使用构造函数自动注入还是默认构造函数。
    - 如果缓存中没有，则需要先确定到底使用哪个构造函数来完成解析工作，因为一个类有多个构造函数，每个构造函数都有不同的构造参数，所以需要根据参数来锁定构造函数并完成初始化，如果存在参数则使用相应的带有参数的构造函数，否则使用默认构造函数。

```java
	protected BeanWrapper createBeanInstance(String beanName, RootBeanDefinition mbd, @Nullable Object[] args) {
		// Make sure bean class is actually resolved at this point.
		// 解析 class
		Class<?> beanClass = resolveBeanClass(mbd, beanName);

		if (beanClass != null && !Modifier.isPublic(beanClass.getModifiers()) && !mbd.isNonPublicAccessAllowed()) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean class isn't public, and non-public access not allowed: " + beanClass.getName());
		}

		// 如果存在 Supplier 回调，则使用给定的回调方法初始化策略
		Supplier<?> instanceSupplier = mbd.getInstanceSupplier();
		if (instanceSupplier != null) {
			return obtainFromSupplier(instanceSupplier, beanName);
		}
		// 如果工厂方法不为空则使用工厂方法初始化策略
		if (mbd.getFactoryMethodName() != null) {
			return instantiateUsingFactoryMethod(beanName, mbd, args);
		}

		// Shortcut when re-creating the same bean...
		boolean resolved = false;
		boolean autowireNecessary = false;
		if (args == null) {
			synchronized (mbd.constructorArgumentLock) {
				// 一个类有多个构造函数，每个构造函数有不同的参数，所以调用前需要
				// 先根据参数锁定构造函数或对应的工厂方法
				// 因为需要根据参数确认到底使用哪个构造函数，该过程比较消耗性能，所有采用缓存机制
				if (mbd.resolvedConstructorOrFactoryMethod != null) {
					resolved = true;
					autowireNecessary = mbd.constructorArgumentsResolved;
				}
			}
		}
		if (resolved) {
			if (autowireNecessary) {
				// 构造函数自动注入
				return autowireConstructor(beanName, mbd, null, null);
			}
			else {
				// 使用默认构造函数构造
				return instantiateBean(beanName, mbd);
			}
		}

		// Candidate constructors for autowiring?
		// 需要根据参数解析构造函数
		// 主要是检查已经注册的 SmartInstantiationAwareBeanPostProcessor
		Constructor<?>[] ctors = determineConstructorsFromBeanPostProcessors(beanClass, beanName);
		if (ctors != null || mbd.getResolvedAutowireMode() == AUTOWIRE_CONSTRUCTOR ||
				mbd.hasConstructorArgumentValues() || !ObjectUtils.isEmpty(args)) {
			// 构造函数自动注入
			return autowireConstructor(beanName, mbd, ctors, args);
		}

		// No special handling: simply use no-arg constructor.
		// 使用默认构造函数构造
		return instantiateBean(beanName, mbd);
	}
```



###### 1.autowireConstructor

- 实现的功能：
  - （1）构造函数参数的确定。
    - 根据 explicitArgs 参数判断。     如果传入的参数 explicitArgs 不为空，那么可以直接确定参数，因为 explicitArgs 参数是在调用 Bean 的使用直接指定的，getObject(String name, Object... args); 。在获取 bean 的时候，用户不但可以指定 bean 的名称，**还可以指定 bean 所对应类的构造函数或者工厂方法的方法参数，主要用于静态工厂方法的调用，而这里需要给定完全匹配的参数，所以，如果传入参数 explicitArgs 不为空，则可以确定构造函数参数就是它。**
    - 缓存中获取。     构造函数参数已经记录在缓存中，可以直接拿来使用。而且，在缓存中缓存的可能是参数的最终类型也可能是参数的初始类型，如：构造函数参数要求的是 int 类型，但是原始的参数值可能是 String 类型的 "1" ，那么即使在缓存中得到了参数，也**需要经过类型转换器的过滤以确保参数类型与对应的构造函数参数类型完全对应。**
    - 配置文件获取。     无法从上述两者中获取，则需要开始新一轮的分析。分析从获取配置文件中配置的构造函数信息开始，Spring 配置文件中的信息经过转换都会通过 BeanDefinition 实例承载，也就是参数 mbd 中包含，那么可以通过**调用 mbd.getConstructorArgumentValues(); 来获取配置的构造函数信息**。有了配置中的信息便可以获取对应的参数值信息了，获取参数值的信息包括直接指定值，如：直接指定构造函数中某个值为原始类型 String 类型，或者是一个对其他 bean 的引用，而这一处理委托给 **resolveConstructorArguments** 方法，并返回能解析到的参数的个数。
  - （2）构造函数的确定。
    - 确定了构造函数的参数之后，根据构造函数参数在所有构造函数中锁定对应的构造函数，**匹配的方法是根据参数个数匹配**，所以在匹配之前需要先对构造函数按照 public 构造函数优先参数数量降序、非 public 构造函数参数数量降序。这样，可以在遍历的情况下迅速判断排在后面的构造函数参数个数是否符合条件。
    - 由于在配置文件中并不是唯一限制使用参数位置索引的方式去创建，同时还支持指定参数名称进行设定参数值的情况，如 <constructor-arg name = "aa">，那么**这种情况就需要首先确定构造函数中的参数名称**。
    - 获取参数名称可以有两种方式，一种是通过注解的方式直接获取 ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);，另一种就是使用 Spring 中提供的工具类 ParameterNameDiscoverer 来获取。
    - 构造函数、参数名称、参数类型、参数值都确定后就可以锁定构造函数以及转换对应的参数类型了。
  - （3）根据确定的构造函数转换对应的参数类型。   
    - 主要使用 **Spring 中提供的类型转换器或者用户提供的自定义类型转换器**进行转换。createArgumentArray
  - （4）构造函数不确定性的验证。
    - 有时候即使构造函数、参数名称、参数类型、参数值都确定后也不一定会直接锁定构造函数，不同构造函数的参数为父子关系，所以 Spring 在最后又做了一次验证。int typeDiffWeight = (mbd.isLenientConstructorResolution() ?      argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
  - （5）**根据实例化策略以及得到的构造函数及构造函数参数实例化 bean。**

```java
	protected BeanWrapper autowireConstructor(
			String beanName, RootBeanDefinition mbd, @Nullable Constructor<?>[] ctors, @Nullable Object[] explicitArgs) {

		return new ConstructorResolver(this).autowireConstructor(beanName, mbd, ctors, explicitArgs);
	}
	

	public BeanWrapper autowireConstructor(String beanName, RootBeanDefinition mbd,
			@Nullable Constructor<?>[] chosenCtors, @Nullable Object[] explicitArgs) {

		BeanWrapperImpl bw = new BeanWrapperImpl();
		this.beanFactory.initBeanWrapper(bw);

		Constructor<?> constructorToUse = null;
		ArgumentsHolder argsHolderToUse = null;
		Object[] argsToUse = null;

		// explicitArgs 通过 getBean 方法传入
		// 如果 getBean 方法调用的时候指定方法参数那么直接使用
		if (explicitArgs != null) {
			argsToUse = explicitArgs;
		}
		else {
			// 如果在 getBean 方法时候没有指定则尝试从配置文件中解析
			Object[] argsToResolve = null;
			// 尝试从缓存中获取
			synchronized (mbd.constructorArgumentLock) {
				constructorToUse = (Constructor<?>) mbd.resolvedConstructorOrFactoryMethod;
				if (constructorToUse != null && mbd.constructorArgumentsResolved) {
					// Found a cached constructor...
					// 从缓存中取
					argsToUse = mbd.resolvedConstructorArguments;
					if (argsToUse == null) {
						// 配置的构造函数参数
						argsToResolve = mbd.preparedConstructorArguments;
					}
				}
			}
			// 如果缓存中存在
			if (argsToResolve != null) {
				// 解析参数类型，如给定方法的构造函数 A(int, int) 则通过此方法后就会把配置中的
				// ("1","1") 转换为 (1,1)
				// 缓存中的值可能是原始值也可能是最终值
				argsToUse = resolvePreparedArguments(beanName, mbd, bw, constructorToUse, argsToResolve);
			}
		}

		// 没有被缓存
		if (constructorToUse == null) {
			// Need to resolve the constructor.
			boolean autowiring = (chosenCtors != null ||
					mbd.getResolvedAutowireMode() == AutowireCapableBeanFactory.AUTOWIRE_CONSTRUCTOR);
			ConstructorArgumentValues resolvedValues = null;

			int minNrOfArgs;
			if (explicitArgs != null) {
				minNrOfArgs = explicitArgs.length;
			}
			else {
				// 提取配置文件中的配置的构造函数参数
				ConstructorArgumentValues cargs = mbd.getConstructorArgumentValues();
				// 用于承载解析后的构造函数参数的值
				resolvedValues = new ConstructorArgumentValues();
				// 能解析到的参数个数
				minNrOfArgs = resolveConstructorArguments(beanName, mbd, bw, cargs, resolvedValues);
			}

			// Take specified constructors, if any.
			Constructor<?>[] candidates = chosenCtors;
			if (candidates == null) {
				Class<?> beanClass = mbd.getBeanClass();
				try {
					candidates = (mbd.isNonPublicAccessAllowed() ?
							beanClass.getDeclaredConstructors() : beanClass.getConstructors());
				}
				catch (Throwable ex) {
					throw new BeanCreationException(mbd.getResourceDescription(), beanName,
							"Resolution of declared constructors on bean Class [" + beanClass.getName() +
							"] from ClassLoader [" + beanClass.getClassLoader() + "] failed", ex);
				}
			}
			// 排序给定的构造函数，public 构造函数优先参数数量降序、非 public 构造函数参数数量降序
			AutowireUtils.sortConstructors(candidates);
			int minTypeDiffWeight = Integer.MAX_VALUE;
			Set<Constructor<?>> ambiguousConstructors = null;
			LinkedList<UnsatisfiedDependencyException> causes = null;

			for (Constructor<?> candidate : candidates) {
				Class<?>[] paramTypes = candidate.getParameterTypes();

				if (constructorToUse != null && argsToUse.length > paramTypes.length) {
					// Already found greedy constructor that can be satisfied ->
					// do not look any further, there are only less greedy constructors left.
					// 如果已经找到选用的构造函数或者需要的参数个数小于当前的构造函数参数个数则终止，
					// 因为已经按照参数个数降序排列
					break;
				}
				if (paramTypes.length < minNrOfArgs) {
					// 参数个数不相等
					continue;
				}

				ArgumentsHolder argsHolder;
				if (resolvedValues != null) {
					// 有参数则根据值构造对应参数类型的参数
					try {
						// 注释上获取参数名称
						String[] paramNames = ConstructorPropertiesChecker.evaluate(candidate, paramTypes.length);
						if (paramNames == null) {
							// 获取参数名称探索器
							ParameterNameDiscoverer pnd = this.beanFactory.getParameterNameDiscoverer();
							if (pnd != null) {
								// 获取指定构造函数的参数名称
								paramNames = pnd.getParameterNames(candidate);
							}
						}
						// 根据名称和数据类型创建参数持有者
						argsHolder = createArgumentArray(beanName, mbd, resolvedValues, bw, paramTypes, paramNames,
								getUserDeclaredConstructor(candidate), autowiring);
					}
					catch (UnsatisfiedDependencyException ex) {
						if (logger.isTraceEnabled()) {
							logger.trace("Ignoring constructor [" + candidate + "] of bean '" + beanName + "': " + ex);
						}
						// Swallow and try next constructor.
						if (causes == null) {
							causes = new LinkedList<>();
						}
						causes.add(ex);
						continue;
					}
				}
				else {
					// Explicit arguments given -> arguments length must match exactly.
					if (paramTypes.length != explicitArgs.length) {
						continue;
					}
					// 构造函数没有参数的情况
					argsHolder = new ArgumentsHolder(explicitArgs);
				}

				// 探测是否有不确定的构造函数存在，例如不同构造函数的参数为父子关系
				int typeDiffWeight = (mbd.isLenientConstructorResolution() ?
						argsHolder.getTypeDifferenceWeight(paramTypes) : argsHolder.getAssignabilityWeight(paramTypes));
				// Choose this constructor if it represents the closest match.
				if (typeDiffWeight < minTypeDiffWeight) {
					constructorToUse = candidate;
					argsHolderToUse = argsHolder;
					argsToUse = argsHolder.arguments;
					minTypeDiffWeight = typeDiffWeight;
					ambiguousConstructors = null;
				}
				else if (constructorToUse != null && typeDiffWeight == minTypeDiffWeight) {
					if (ambiguousConstructors == null) {
						ambiguousConstructors = new LinkedHashSet<>();
						ambiguousConstructors.add(constructorToUse);
					}
					ambiguousConstructors.add(candidate);
				}
			}

			if (constructorToUse == null) {
				if (causes != null) {
					UnsatisfiedDependencyException ex = causes.removeLast();
					for (Exception cause : causes) {
						this.beanFactory.onSuppressedException(cause);
					}
					throw ex;
				}
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Could not resolve matching constructor " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities)");
			}
			else if (ambiguousConstructors != null && !mbd.isLenientConstructorResolution()) {
				throw new BeanCreationException(mbd.getResourceDescription(), beanName,
						"Ambiguous constructor matches found in bean '" + beanName + "' " +
						"(hint: specify index/type/name arguments for simple parameters to avoid type ambiguities): " +
						ambiguousConstructors);
			}

			if (explicitArgs == null) {
				// 将解析的构造函数加入到缓存中
				argsHolderToUse.storeCache(mbd, constructorToUse);
			}
		}

		try {
			final InstantiationStrategy strategy = beanFactory.getInstantiationStrategy();
			Object beanInstance;

			if (System.getSecurityManager() != null) {
				final Constructor<?> ctorToUse = constructorToUse;
				final Object[] argumentsToUse = argsToUse;
				beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
						strategy.instantiate(mbd, beanName, beanFactory, ctorToUse, argumentsToUse),
						beanFactory.getAccessControlContext());
			}
			else {
				beanInstance = strategy.instantiate(mbd, beanName, this.beanFactory, constructorToUse, argsToUse);
			}

			// 将构建的实例加入 BeanWrapper 中
			bw.setBeanInstance(beanInstance);
			return bw;
		}
		catch (Throwable ex) {
			throw new BeanCreationException(mbd.getResourceDescription(), beanName,
					"Bean instantiation via constructor failed", ex);
		}
	}
```



###### 2.instantiateBean

- 不带参数的构造函数的实例化过程。

```java
protected BeanWrapper instantiateBean(final String beanName, final RootBeanDefinition mbd) {
        try {
            Object beanInstance;
            final BeanFactory parent = this;
            if (System.getSecurityManager() != null) {
                beanInstance = AccessController.doPrivileged((PrivilegedAction<Object>) () ->
                                                             getInstantiationStrategy().instantiate(mbd, beanName, parent),
                                                             getAccessControlContext());
            }
            else {
                // 直接调用实例化策略进行实例化
                beanInstance = getInstantiationStrategy().instantiate(mbd, beanName, parent);
            }
            BeanWrapper bw = new BeanWrapperImpl(beanInstance);
            initBeanWrapper(bw);
            return bw;
        }
        catch (Throwable ex) {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Instantiation of bean failed", ex);
        }
    }	
```



###### 3.实例化策略

- 前面的分析已经得到实例化的所有相关信息，但是 Spring 不是直接用反射来构造实例对象。

SimpleInstantiationStrategy.java

```java
public Object instantiate(RootBeanDefinition bd, @Nullable String beanName, BeanFactory owner,
			final Constructor<?> ctor, @Nullable Object... args) {

		// 如果有需要覆盖或者动态替换的方法则需要使用 cglib 进行动态代理，
		// 因为可以在创建代理的同时将动态方法织入类中，
		// 但是如果没有需要动态改变的方法，为了方便直接反射就可以了
		if (!bd.hasMethodOverrides()) {
			if (System.getSecurityManager() != null) {
				// use own privileged to change accessibility (when security is on)
				AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
					ReflectionUtils.makeAccessible(ctor);
					return null;
				});
			}
			return (args != null ? BeanUtils.instantiateClass(ctor, args) : BeanUtils.instantiateClass(ctor));
		}
		else {
			return instantiateWithMethodInjection(bd, beanName, owner, ctor, args);
		}
	}
```



##### 4.7.2 记录创建 bean 的 ObjectFactory

- 变量 earlySingletonExposure 是 是否是单例、是否允许循环依赖、是否对应的 bean 正在创建的条件的综合。当这三个条件都满足时会执行 addSingletonFactory 方法。
- **Spring 处理循环依赖的解决方法，在 B 中创建依赖 A 时通过 ObjectFactory 提供的实例化方法来中断 A 中的属性填充，使 B 中持有的 A 仅仅是刚刚初始化并没有填充任何属性的 A，而这正初始化 A 的步骤还是在最开始创建 A 的时候进行的，但是因为 A 与 B 中的 A 所表示的属性地址是一样的，所以在 A 中创建好的属性填充自然可以通过 B 中的 A 获取，这样就解决了循环依赖的问题。

```java
    // 是否需要提前曝光：单例 & 允许循环依赖 & 当前 bean 正在创建中，检测循环依赖
    boolean earlySingletonExposure = (mbd.isSingleton() && this.allowCircularReferences &&
                                      isSingletonCurrentlyInCreation(beanName));
    if (earlySingletonExposure) {
        if (logger.isDebugEnabled()) {
            logger.debug("Eagerly caching bean '" + beanName +
                         "' to allow for resolving potential circular references");
        }
        // 为避免后期循环依赖，可以在 bean 初始化完成前将创建实例的 ObjectFactory 加入工厂
        // getEarlyBeanReference：对 bean 再一次依赖引用，主要应用 SmartInstantiationAwareBeanPostProcessor，
        // AOP 就是在这里将 Advice 动态织入 bean 中，若没有则直接返回 bean，不作任何处理
        addSingletonFactory(beanName, () -> getEarlyBeanReference(beanName, mbd, bean));
    }


	protected Object getEarlyBeanReference(String beanName, RootBeanDefinition mbd, Object bean) {
		Object exposedObject = bean;
		if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
			for (BeanPostProcessor bp : getBeanPostProcessors()) {
				if (bp instanceof SmartInstantiationAwareBeanPostProcessor) {
					SmartInstantiationAwareBeanPostProcessor ibp = (SmartInstantiationAwareBeanPostProcessor) bp;
					exposedObject = ibp.getEarlyBeanReference(exposedObject, beanName);
				}
			}
		}
		return exposedObject;
	}

```



##### 4.7.3 属性注入

- populateBean() 函数的主要功能是属性填充。处理流程：
  - （1）InstantiationAwareBeanPostProcessor 处理器的 postProcessAfterInstantiation 函数的应用，此函数可以控制程序是否继续进行属性填充。
  - （2）根据注入类型（byName/byType），提取依赖的 bean，并同意存入 PropertyValues 类型中。
  - （3）应用 InstantiationAwareBeanPostProcessor 处理器的 postProcessPropertyValues 方法，对属性获取完毕填充前对属性的再次处理，典型应用是 RequiredAnnotationBeanPostProcessor 类中对属性的验证。
  - （4）将所有 PropertyValues 中的属性填充至 BeanWrapper 中。applyPropertyValues(beanName, mbd, bw, pvs);

```java
    protected void populateBean(String beanName, RootBeanDefinition mbd, @Nullable BeanWrapper bw) {
        if (bw == null) {
            if (mbd.hasPropertyValues()) {
                throw new BeanCreationException(
                    mbd.getResourceDescription(), beanName, "Cannot apply property values to null instance");
            }
            else {
                // Skip property population phase for null instance.
                // 没有可填充的属性
                return;
            }
        }

        // Give any InstantiationAwareBeanPostProcessors the opportunity to modify the
        // state of the bean before properties are set. This can be used, for example,
        // to support styles of field injection.
        if (!mbd.isSynthetic() && hasInstantiationAwareBeanPostProcessors()) {
            for (BeanPostProcessor bp : getBeanPostProcessors()) {
                if (bp instanceof InstantiationAwareBeanPostProcessor) {
                    InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                    // 返回值为是否继续填充 bean
                    if (!ibp.postProcessAfterInstantiation(bw.getWrappedInstance(), beanName)) {
                        return;
                    }
                }
            }
        }

        PropertyValues pvs = (mbd.hasPropertyValues() ? mbd.getPropertyValues() : null);

        int resolvedAutowireMode = mbd.getResolvedAutowireMode();
        if (resolvedAutowireMode == AUTOWIRE_BY_NAME || resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
            MutablePropertyValues newPvs = new MutablePropertyValues(pvs);
            // Add property values based on autowire by name if applicable.
            // 根据名称自动注入
            if (resolvedAutowireMode == AUTOWIRE_BY_NAME) {
                autowireByName(beanName, mbd, bw, newPvs);
            }
            // Add property values based on autowire by type if applicable.
            // 根据类型自动注入
            if (resolvedAutowireMode == AUTOWIRE_BY_TYPE) {
                autowireByType(beanName, mbd, bw, newPvs);
            }
            pvs = newPvs;
        }

        // 后置处理器是否已经初始化
        boolean hasInstAwareBpps = hasInstantiationAwareBeanPostProcessors();
        // 是否需要依赖检测
        boolean needsDepCheck = (mbd.getDependencyCheck() != AbstractBeanDefinition.DEPENDENCY_CHECK_NONE);

        if (hasInstAwareBpps || needsDepCheck) {
            if (pvs == null) {
                pvs = mbd.getPropertyValues();
            }
            PropertyDescriptor[] filteredPds = filterPropertyDescriptorsForDependencyCheck(bw, mbd.allowCaching);
            if (hasInstAwareBpps) {
                for (BeanPostProcessor bp : getBeanPostProcessors()) {
                    if (bp instanceof InstantiationAwareBeanPostProcessor) {
                        InstantiationAwareBeanPostProcessor ibp = (InstantiationAwareBeanPostProcessor) bp;
                        // 对所有需要依赖检查的属性进行后置处理
                        pvs = ibp.postProcessPropertyValues(pvs, filteredPds, bw.getWrappedInstance(), beanName);
                        if (pvs == null) {
                            return;
                        }
                    }
                }
            }
            if (needsDepCheck) {
                // 依赖检查，对应 depends-on 属性，3.0 已经弃用此属性
                checkDependencies(beanName, mbd, filteredPds, pvs);
            }
        }

        if (pvs != null) {
            // 将属性应用到 bean 中
            applyPropertyValues(beanName, mbd, bw, pvs);
        }
    }
```



###### 1.autowireByName

- 根据注入类型（byName/byType）提取依赖的 bean，byName 功能的实现如下：
- 在传入的参数 pvs 中找出已经加载的 bean，并递归实例化，进而加入到 pvs 中。

```java
    protected void autowireByName(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

        // 寻找 bw 中需要依赖注入的属性
        String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
        for (String propertyName : propertyNames) {
            if (containsBean(propertyName)) {
                // 递归初始化相关的 bean
                Object bean = getBean(propertyName);
                // bean 加入到 pvs 中
                pvs.add(propertyName, bean);
                // 注册依赖
                registerDependentBean(propertyName, beanName);
                if (logger.isDebugEnabled()) {
                    logger.debug("Added autowiring by name from bean name '" + beanName +
                                 "' via property '" + propertyName + "' to bean named '" + propertyName + "'");
                }
            }
            else {
                if (logger.isTraceEnabled()) {
                    logger.trace("Not autowiring property '" + propertyName + "' of bean '" + beanName +
                                 "' by name: no matching bean found");
                }
            }
        }
    }
```



###### 2.autoriweByType

- autowireByType 和 autowireByName 对于我们理解与使用来说复杂程度都很相似，但是其实现功能的复杂度却完全不一样。
- ByType 自动匹配的实现的第一步也是寻找 bw 中需要依赖注入的属性，然后遍历这些属性并寻找类型匹配的 bean，其中最复杂的是**寻找类型匹配的 bean**。

```java
    protected void autowireByType(
        String beanName, AbstractBeanDefinition mbd, BeanWrapper bw, MutablePropertyValues pvs) {

        TypeConverter converter = getCustomTypeConverter();
        if (converter == null) {
            converter = bw;
        }

        Set<String> autowiredBeanNames = new LinkedHashSet<>(4);
        // 寻找 bw 中需要依赖注入的属性
        String[] propertyNames = unsatisfiedNonSimpleProperties(mbd, bw);
        for (String propertyName : propertyNames) {
            try {
                PropertyDescriptor pd = bw.getPropertyDescriptor(propertyName);
                // Don't try autowiring by type for type Object: never makes sense,
                // even if it technically is a unsatisfied, non-simple property.
                if (Object.class != pd.getPropertyType()) {
                    // 探测指定属性的 set 方法
                    MethodParameter methodParam = BeanUtils.getWriteMethodParameter(pd);
                    // Do not allow eager init for type matching in case of a prioritized post-processor.
                    boolean eager = !(bw.getWrappedInstance() instanceof PriorityOrdered);
                    DependencyDescriptor desc = new AutowireByTypeDependencyDescriptor(methodParam, eager);
                    // 解析指定 beanName 的属性所匹配的值，并把解析到的属性名称
                    // 存储在 autowiredBeanNames 中，当属性存在多个封装 bean 时如：
                    // @Autowired private List<A> aList; 将会找到所有匹配 A 类型的 bean 并将其注入
                    Object autowiredArgument = resolveDependency(desc, beanName, autowiredBeanNames, converter);
                    if (autowiredArgument != null) {
                        pvs.add(propertyName, autowiredArgument);
                    }
                    for (String autowiredBeanName : autowiredBeanNames) {
                        // 注册依赖
                        registerDependentBean(autowiredBeanName, beanName);
                        if (logger.isDebugEnabled()) {
                            logger.debug("Autowiring by type from bean name '" + beanName + "' via property '" +
                                         propertyName + "' to bean named '" + autowiredBeanName + "'");
                        }
                    }
                    autowiredBeanNames.clear();
                }
            }
            catch (BeansException ex) {
                throw new UnsatisfiedDependencyException(mbd.getResourceDescription(), beanName, propertyName, ex);
            }
        }
    }
```

- 寻找类型匹配的逻辑实现封装在了 resolveDependency 函数中。

```java
    public Object resolveDependency(DependencyDescriptor descriptor, @Nullable String requestingBeanName,
                                    @Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

        descriptor.initParameterNameDiscovery(getParameterNameDiscoverer());
        if (Optional.class == descriptor.getDependencyType()) {
            return createOptionalDependency(descriptor, requestingBeanName);
        }
        else if (ObjectFactory.class == descriptor.getDependencyType() ||
                 ObjectProvider.class == descriptor.getDependencyType()) {
            // ObjectFactory 类注入的特殊处理
            return new DependencyObjectProvider(descriptor, requestingBeanName);
        }
        else if (javaxInjectProviderClass == descriptor.getDependencyType()) {
            // javaxInjectProviderClass 类注入的特殊处理
            return new Jsr330ProviderFactory().createDependencyProvider(descriptor, requestingBeanName);
        }
        else {
            Object result = getAutowireCandidateResolver().getLazyResolutionProxyIfNecessary(
                descriptor, requestingBeanName);
            if (result == null) {
                // 通用处理逻辑
                result = doResolveDependency(descriptor, requestingBeanName, autowiredBeanNames, typeConverter);
            }
            return result;
        }
    }


	public Object doResolveDependency(DependencyDescriptor descriptor, @Nullable String beanName,
			@Nullable Set<String> autowiredBeanNames, @Nullable TypeConverter typeConverter) throws BeansException {

		// 注入点
		InjectionPoint previousInjectionPoint = ConstructorResolver.setCurrentInjectionPoint(descriptor);
		try {
			// 针对给定的工厂给定一个快捷实现的方法，例如考虑一些预先解析的信息
			// 在进入所有bean的常规类型匹配算法之前，解析算法将首先尝试通过此方法解析快捷方式
			// 子类可以覆盖此方法
			Object shortcut = descriptor.resolveShortcut(this);
			if (shortcut != null) {
				// 返回快捷的解析信息
				return shortcut;
			}

			// 依赖的类型
			Class<?> type = descriptor.getDependencyType();
			// 用于支持 Spring 中的注解 @Value
			Object value = getAutowireCandidateResolver().getSuggestedValue(descriptor);
			if (value != null) {
				if (value instanceof String) {
					String strVal = resolveEmbeddedValue((String) value);
					BeanDefinition bd = (beanName != null && containsBean(beanName) ? getMergedBeanDefinition(beanName) : null);
					value = evaluateBeanDefinitionString(strVal, bd);
				}
				TypeConverter converter = (typeConverter != null ? typeConverter : getTypeConverter());
				return (descriptor.getField() != null ?
						converter.convertIfNecessary(value, type, descriptor.getField()) :
						converter.convertIfNecessary(value, type, descriptor.getMethodParameter()));
			}

			// 解析复合 bean，其实就是对 bean 的属性进行解析
			// 包括：数组、Collection 、Map 类型
			Object multipleBeans = resolveMultipleBeans(descriptor, beanName, autowiredBeanNames, typeConverter);
			if (multipleBeans != null) {
				return multipleBeans;
			}

			// 查找与类型相匹配的 bean
			// 返回值构成为：key = 匹配的 beanName，value = beanName 对应的实例化 bean
			Map<String, Object> matchingBeans = findAutowireCandidates(beanName, type, descriptor);
			// 没有找到，检验 @Autowired  的 require 是否为 true
			if (matchingBeans.isEmpty()) {
				if (isRequired(descriptor)) {
					// 如果 @Autowired 的 require 属性为 true ，但是没有找到相应的匹配项，则抛出异常
					raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
				}
				return null;
			}

			String autowiredBeanName;
			Object instanceCandidate;

			if (matchingBeans.size() > 1) {
				// 确认给定 bean autowire 的候选者
				// 按照 @Primary 和 @Priority 的顺序
				autowiredBeanName = determineAutowireCandidate(matchingBeans, descriptor);
				if (autowiredBeanName == null) {
					if (isRequired(descriptor) || !indicatesMultipleBeans(type)) {
						// 唯一性处理
						return descriptor.resolveNotUnique(type, matchingBeans);
					}
					else {
						// In case of an optional Collection/Map, silently ignore a non-unique case:
						// possibly it was meant to be an empty collection of multiple regular beans
						// (before 4.3 in particular when we didn't even look for collection beans).
						// 在可选的Collection / Map的情况下，默默地忽略一个非唯一的情况：
						// 可能它是一个多个常规bean的空集合
						return null;
					}
				}
				instanceCandidate = matchingBeans.get(autowiredBeanName);
			}
			else {
				// We have exactly one match.
				// 只有一个 bean 匹配
				Map.Entry<String, Object> entry = matchingBeans.entrySet().iterator().next();
				autowiredBeanName = entry.getKey();
				instanceCandidate = entry.getValue();
			}

			if (autowiredBeanNames != null) {
				autowiredBeanNames.add(autowiredBeanName);
			}
			if (instanceCandidate instanceof Class) {
				instanceCandidate = descriptor.resolveCandidate(autowiredBeanName, type, this);
			}
				Object result = instanceCandidate;
				if (result instanceof NullBean) {
					if (isRequired(descriptor)) {
						raiseNoMatchingBeanFound(type, descriptor.getResolvableType(), descriptor);
					}
				result = null;
			}
			if (!ClassUtils.isAssignableValue(type, result)) {
				throw new BeanNotOfRequiredTypeException(autowiredBeanName, type, instanceCandidate.getClass());
			}
			return result;
		}
		finally {
			ConstructorResolver.setCurrentInjectionPoint(previousInjectionPoint);
		}
	}
```



###### 3.applyPropertyValues

- 上述已经完成了对所有注入属性的获取，但是获取的属性是以 PropertyValues 形式存在的，应用到已经实例化的 bean 中的工作是在 applyPropertyValues 方法中进行的。

```java
    protected void applyPropertyValues(String beanName, BeanDefinition mbd, BeanWrapper bw, PropertyValues pvs) {
        if (pvs.isEmpty()) {
            return;
        }

        if (System.getSecurityManager() != null && bw instanceof BeanWrapperImpl) {
            ((BeanWrapperImpl) bw).setSecurityContext(getAccessControlContext());
        }

        // MutablePropertyValues 类型属性
        MutablePropertyValues mpvs = null;
        // 原始类型
        List<PropertyValue> original;

        if (pvs instanceof MutablePropertyValues) {
            mpvs = (MutablePropertyValues) pvs;
            // 如果 mpvs 中的值已经被转换为对应的类型那么可以直接设置到 BeanWrapper 中
            if (mpvs.isConverted()) {
                // Shortcut: use the pre-converted values as-is.
                try {
                    bw.setPropertyValues(mpvs);
                    return;
                }
                catch (BeansException ex) {
                    throw new BeanCreationException(
                        mbd.getResourceDescription(), beanName, "Error setting property values", ex);
                }
            }
            original = mpvs.getPropertyValueList();
        }
        // 如果 pvs 不是 MutablePropertyValues 类型，则直接使用原始类型
        else {
            original = Arrays.asList(pvs.getPropertyValues());
        }

        // 获取 TypeConverter
        TypeConverter converter = getCustomTypeConverter();
        if (converter == null) {
            converter = bw;
        }
        // 获取对应的解析器
        BeanDefinitionValueResolver valueResolver = new BeanDefinitionValueResolver(this, beanName, mbd, converter);

        // Create a deep copy, resolving any references for values.
        List<PropertyValue> deepCopy = new ArrayList<>(original.size());
        boolean resolveNecessary = false;
        // 遍历属性，将属性转换为对应类的对应属性的类型
        for (PropertyValue pv : original) {
            if (pv.isConverted()) {
                deepCopy.add(pv);
            }
            else {
                String propertyName = pv.getName();
                Object originalValue = pv.getValue();
                Object resolvedValue = valueResolver.resolveValueIfNecessary(pv, originalValue);
                Object convertedValue = resolvedValue;
                boolean convertible = bw.isWritableProperty(propertyName) &&
                    !PropertyAccessorUtils.isNestedOrIndexedProperty(propertyName);
                if (convertible) {
                    convertedValue = convertForProperty(resolvedValue, propertyName, bw, converter);
                }
                // Possibly store converted value in merged bean definition,
                // in order to avoid re-conversion for every created bean instance.
                if (resolvedValue == originalValue) {
                    if (convertible) {
                        pv.setConvertedValue(convertedValue);
                    }
                    deepCopy.add(pv);
                }
                else if (convertible && originalValue instanceof TypedStringValue &&
                         !((TypedStringValue) originalValue).isDynamic() &&
                         !(convertedValue instanceof Collection || ObjectUtils.isArray(convertedValue))) {
                    pv.setConvertedValue(convertedValue);
                    deepCopy.add(pv);
                }
                else {
                    resolveNecessary = true;
                    deepCopy.add(new PropertyValue(pv, convertedValue));
                }
            }
        }
        if (mpvs != null && !resolveNecessary) {
            mpvs.setConverted();
        }

        // Set our (possibly massaged) deep copy.
        try {
            bw.setPropertyValues(new MutablePropertyValues(deepCopy));
        }
        catch (BeansException ex) {
            throw new BeanCreationException(
                mbd.getResourceDescription(), beanName, "Error setting property values", ex);
        }
    }
```



##### 4.7.4 初始化 bean

- Spring 中程序已经执行过 bean 的实例化，并且进行了属性的填充后，将会调用用户设定的初始化方法。

```java
	protected Object initializeBean(final String beanName, final Object bean, @Nullable RootBeanDefinition mbd) {
		if (System.getSecurityManager() != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				// 激活 Aware 方法
				invokeAwareMethods(beanName, bean);
				return null;
			}, getAccessControlContext());
		}
		else {
			// 对特殊的 bean 处理：Aware、BeanClassLoaderAware、BeanFactoryAware
			invokeAwareMethods(beanName, bean);
		}

		Object wrappedBean = bean;
		if (mbd == null || !mbd.isSynthetic()) {
			// 应用后置处理器
			wrappedBean = applyBeanPostProcessorsBeforeInitialization(wrappedBean, beanName);
		}

		try {
			// 激活用户自定义的 init 方法
			invokeInitMethods(beanName, wrappedBean, mbd);
		}
		catch (Throwable ex) {
			throw new BeanCreationException(
					(mbd != null ? mbd.getResourceDescription() : null),
					beanName, "Invocation of init method failed", ex);
		}
		if (mbd == null || !mbd.isSynthetic()) {
			// 后置处理器应用
			wrappedBean = applyBeanPostProcessorsAfterInitialization(wrappedBean, beanName);
		}

		return wrappedBean;
	}
```

###### 1.激活 Aware 方法

- Spring 中提供一些 Aware 相关接口，比如：BeanFactoryAware、ApplicationContextAware、ResourceLoaderAware、ServletContextAware 等，**实现这些 Aware 接口的 bean 在初始化后，可以取得一些相对应的资源。**如：实现 BeanFactoryAware 的 bean 在初始化后，Spring 容器将会注入 BeanFactory 的实例，而实现 ApplicationContextAware 的 bean，在 bean 被初始化后，将会被注入 ApplicationContext 的实例等。Aware 的使用示例如下：

```java
// (1)定义普通 bean
public class Hello {
    public void say() {
        System.out.println("hello");
    }
}

// (2)定义 BeanFactoryAware 类型的 bean
public class Test implements BeanFactoryAware {
    private BeanFactory beanFactory;
    
    // 声明 bean 的时候 Spring 会自动注入 BeanFactory
    @Override
    public void setBeanFactory(BeanFactory beanFactory) throws BeansException {
        this.beanFactory = beanFactory;
    }
    
    public void testAware() {
        // 通过 hello 这个 bean id 从 beanFactory 获取实例
        Hello hello = (Hello) beanFactory.getBean("hello");
        hello.say();
    }
}

// (3)使用 main 方法测试。
public static void main(String[] args) {
    ApplicationContext ctx = new ClassPathXmlApplicationContext("applicationContext.xml");
    Test test = (Test) ctx.getBean("test");
    test.testAware();	// 控制台输出：hello
}

```



- Spring 的 Aware 方法的实现方式。

```java
    private void invokeAwareMethods(final String beanName, final Object bean) {
        if (bean instanceof Aware) {
            if (bean instanceof BeanNameAware) {
                ((BeanNameAware) bean).setBeanName(beanName);
            }
            if (bean instanceof BeanClassLoaderAware) {
                ClassLoader bcl = getBeanClassLoader();
                if (bcl != null) {
                    ((BeanClassLoaderAware) bean).setBeanClassLoader(bcl);
                }
            }
            if (bean instanceof BeanFactoryAware) {
                ((BeanFactoryAware) bean).setBeanFactory(AbstractAutowireCapableBeanFactory.this);
            }
        }
    }
```



###### 2.后置处理器的应用

- 在调用客户自定义初始化方法前以及调用自定义初始化方法后分别会调用 BeanPostProcessor 的 postProcessBeforeInitialization和 postProcessorsAfterInitialization 方法，使用户可以根据自己的业务需求进行相应的处理。

```java
	public Object applyBeanPostProcessorsBeforeInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
			Object current = processor.postProcessBeforeInitialization(result, beanName);
			if (current == null) {
				return result;
			}
			result = current;
		}
		return result;
	}



	public Object applyBeanPostProcessorsAfterInitialization(Object existingBean, String beanName)
			throws BeansException {

		Object result = existingBean;
		for (BeanPostProcessor processor : getBeanPostProcessors()) {
				Object current = processor.postProcessAfterInitialization(result, beanName);
				if (current == null) {
					return result;
				}
			result = current;
		}
		return result;
	}
```



###### 3.激活自定义的 init 方法

- 初始化方法除了使用配置 init-method 外，还有使自定义的 bean 实现 InitializationBean 接口，并在 afterPropertiesSet 中实现自己的初始化业务逻辑。执行顺序是 afterPropertiesSet ----> init-method。
- 在 invokeInitMethods 方法中实现了这两个步骤的初始化方法调用。

```java
	protected void invokeInitMethods(String beanName, final Object bean, @Nullable RootBeanDefinition mbd)
			throws Throwable {

		// 首先检查是否是 InitializationBean，如果是的话需要调用 afterPropertiesSet 方法
		boolean isInitializingBean = (bean instanceof InitializingBean);
		if (isInitializingBean && (mbd == null || !mbd.isExternallyManagedInitMethod("afterPropertiesSet"))) {
			if (logger.isDebugEnabled()) {
				logger.debug("Invoking afterPropertiesSet() on bean with name '" + beanName + "'");
			}
			if (System.getSecurityManager() != null) {
				try {
					AccessController.doPrivileged((PrivilegedExceptionAction<Object>) () -> {
						((InitializingBean) bean).afterPropertiesSet();
						return null;
					}, getAccessControlContext());
				}
				catch (PrivilegedActionException pae) {
					throw pae.getException();
				}
			}
			else {
				// 属性初始化后的处理
				((InitializingBean) bean).afterPropertiesSet();
			}
		}

		if (mbd != null && bean.getClass() != NullBean.class) {
			String initMethodName = mbd.getInitMethodName();
			if (StringUtils.hasLength(initMethodName) &&
					!(isInitializingBean && "afterPropertiesSet".equals(initMethodName)) &&
					!mbd.isExternallyManagedInitMethod(initMethodName)) {
				// 调用自定义初始化方法
				invokeCustomInitMethod(beanName, bean, mbd);
			}
		}
	}
```



##### 5.7.5 注册 DisposableBean

- Spring 还提供了销毁方法的扩展入口，对于销毁方法的扩展，除了配置属性 destroy-method 方法外，用户还可以注册后置处理器 DestructionAwareBeanPostProcessor 来统一处理 bean 的销毁方法。 registerDisposableBeanIfNecessary 方法代码如下：

```java
    protected void registerDisposableBeanIfNecessary(String beanName, Object bean, RootBeanDefinition mbd) {
        AccessControlContext acc = (System.getSecurityManager() != null ? getAccessControlContext() : null);
        if (!mbd.isPrototype() && requiresDestruction(bean, mbd)) {
            if (mbd.isSingleton()) {
                // Register a DisposableBean implementation that performs all destruction
                // work for the given bean: DestructionAwareBeanPostProcessors,
                // DisposableBean interface, custom destroy method.
                /*
                     *	单例模式下注册需要销毁的 bean，此方法会处理实现 DIsposableBean 的 bean，
                     * 	并且对所有的 bean 使用 DestructionAwareBeanPostProcessors 处理
                     */
                registerDisposableBean(beanName,
                                       new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
            }
            else {
                // A bean with a custom scope...
                // 自定义 scope 的处理
                Scope scope = this.scopes.get(mbd.getScope());
                if (scope == null) {
                    throw new IllegalStateException("No Scope registered for scope name '" + mbd.getScope() + "'");
                }
                scope.registerDestructionCallback(beanName,
                                                  new DisposableBeanAdapter(bean, beanName, mbd, getBeanPostProcessors(), acc));
            }
        }
    }
```



### 5 容器的功能扩展

- ApplicationContext 和 BeanFactory 都是用于加载 Bean 的，但是 ApplicationContext 提供了更多的扩展功能。
- 写法：ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml"); 。ClassPathXmlApplicationContext 中可以将配置文件路径以数组的方式传入，ClassPathXmlApplicationContext  可以对数组进行解析并进行加载，解析及功能实现都在 refresh() 中实现。

```java
	public ClassPathXmlApplicationContext(String configLocation) throws BeansException {
		this(new String[] {configLocation}, true, null);
	}

	public ClassPathXmlApplicationContext(
			String[] configLocations, boolean refresh, @Nullable ApplicationContext parent)
			throws BeansException {

		super(parent);
		setConfigLocations(configLocations);
		if (refresh) {
			refresh();
		}
	}
```



#### 5.1 设置配置路径

- 在 ClassPathXmlApplicationContext 中支持多个配置文件已数组方式同时传入：

```java
	public void setConfigLocations(@Nullable String... locations) {
		if (locations != null) {
			Assert.noNullElements(locations, "Config locations must not be null");
			this.configLocations = new String[locations.length];
			for (int i = 0; i < locations.length; i++) {
				// 解析给定路径，如果数组中包含特殊符号，如 ${var}，那么 resolvePath 会搜寻匹配的系统变量并替换
				this.configLocations[i] = resolvePath(locations[i]).trim();
			}
		}
		else {
			this.configLocations = null;
		}
	}
```



#### 5.2 扩展功能

- 设置了路径之后，便可以根据路径做配置文件的解析以及各种功能的实现。refresh 包含了几乎 ApplicationContext 中提供的全部功能。
- ClassPathXmlApplicationXontext 初始化的步骤：
  - （1）初始化前的准备工作，例如对系统属性或者环境变量进行准备及验证。prepareRefresh 函数可以在 Spring 启动的时候提前对必须的变量进行存在性验证。
  - （2）初始化 BeanFactory，并进行 XML 文件读取。obtainFreshBeanFactory 会复用 BeanFactory 中的配置文件解析及其他功能，可以进行 bean 的提取等基础操作。
  - （3）对 BeanFactory 进行各种功能填充。如增加 @Qualifier 和 @Autowired 注解支持。
  - （4）子类覆盖方法做额外的处理。Spring 提供了一个空的函数实现 postProcessBeanFactory 来方便程序员在业务上做进一步扩展。
  - （5）激活各种 BeanFactory 处理器。invokeBeanFactoryPostProcessors
  - （6）注册拦截 Bean 创建的 bean 处理器，这里只是注册，真正的调用是在 getBean 时候。registerBeanPostProcessors
  - （7）为上下文初始化 Message 源，即对不同语言的消息体进行国际化处理。initMessageSource
  - （8）初始化应用消息广播，并放入"applicationEventMulticaster" bean 中。
  - （9）留给子类来初始化其他的 bean。onRefresh();
  - （10）在所有注册的 bean 中查找 Listener bean，注册到消息广播器中。registerListeners();
  - （11）初始化上下的单例实例（非惰性的）。finishBeanFactoryInitialization(beanFactory);
  - （12）完成刷新过程，通知生命周期处理器 lifecycleProcessor 刷新过程，同时发出 ContextRefreshEvent 通知别人。finishRefresh();

```java
	public void refresh() throws BeansException, IllegalStateException {
		synchronized (this.startupShutdownMonitor) {
			// Prepare this context for refreshing.
			// 准备刷新的上下文环境
			prepareRefresh();

			// Tell the subclass to refresh the internal bean factory.
			// 初始化 BeanFactory，并进行 XML 文件读取
			ConfigurableListableBeanFactory beanFactory = obtainFreshBeanFactory();

			// Prepare the bean factory for use in this context.
			// 对 BeanFactory 进行各种功能填充
			prepareBeanFactory(beanFactory);

			try {
				// Allows post-processing of the bean factory in context subclasses.
				// 子类覆盖方法做额外的处理
				postProcessBeanFactory(beanFactory);

				// Invoke factory processors registered as beans in the context.
				// 激活各种 BeanFactory 处理器
				invokeBeanFactoryPostProcessors(beanFactory);

				// Register bean processors that intercept bean creation.
				// 注册拦截 Bean 创建的 bean 处理器，这里只是注册，真正的调用是在 getBean 时候
				registerBeanPostProcessors(beanFactory);

				// Initialize message source for this context.
				// 为上下文初始化 Message 员，即不同语言的消息体，国际化处理
				initMessageSource();

				// Initialize event multicaster for this context.
				// 初始化应用消息广播，并放入"applicationEventMulticaster" bean 中
				initApplicationEventMulticaster();

				// Initialize other special beans in specific context subclasses.
				// 留给子类来初始化其他的 bean
				onRefresh();

				// Check for listener beans and register them.
				// 在所有注册的 bean 中查找 Listener bean，注册到消息广播器中
				registerListeners();

				// Instantiate all remaining (non-lazy-init) singletons.
				// 初始化上下的单例实例（非惰性的）
				finishBeanFactoryInitialization(beanFactory);

				// Last step: publish corresponding event.
				// 完成水泥过程，通知生命周期处理器 lifecycleProcessor 刷新过程，
				// 同时发出 ContextRefreshEvent 通知别人
				finishRefresh();
			}

			catch (BeansException ex) {
				if (logger.isWarnEnabled()) {
					logger.warn("Exception encountered during context initialization - " +
							"cancelling refresh attempt: " + ex);
				}

				// Destroy already created singletons to avoid dangling resources.
				destroyBeans();

				// Reset 'active' flag.
				cancelRefresh(ex);

				// Propagate exception to caller.
				throw ex;
			}

			finally {
				// Reset common introspection caches in Spring's core, since we
				// might not ever need metadata for singleton beans anymore...
				resetCommonCaches();
			}
		}
	}
```



#### 5.3 环境准备

-  initPropertySources 符合 Spring 的开放式结构设计，给用户最大扩展 Spring 的能力。用户可以根据自身的需要重写 initPropertySources 方法，并在方法中进行个性化的属性处理。
- validateRequiredProperties 则是对属性进行验证。可以自定义继承自 ClassPathXmlApplicationContext 的 MyClassPathXmlApplicationContext，并重写 initPropertySources 方法。

```java
    protected void prepareRefresh() {
        // Switch to active.
        this.startupDate = System.currentTimeMillis();
        this.closed.set(false);
        this.active.set(true);

        if (logger.isInfoEnabled()) {
            logger.info("Refreshing " + this);
        }

        // Initialize any |placeholder property sources in the context environment.
        // 留给子类覆盖
        initPropertySources();

        // Validate that all properties marked as required are resolvable:
        // see ConfigurablePropertyResolver#setRequiredProperties
        // 验证需要的属性文件是否都已经放入环境中
        getEnvironment().validateRequiredProperties();

        // Store pre-refresh ApplicationListeners...
        if (this.earlyApplicationListeners == null) {
            this.earlyApplicationListeners = new LinkedHashSet<>(this.applicationListeners);
        }
        else {
            // Reset local application listeners to pre-refresh state.
            this.applicationListeners.clear();
            this.applicationListeners.addAll(this.earlyApplicationListeners);
        }

        // Allow for the collection of early ApplicationEvents,
        // to be published once the multicaster is available...
        this.earlyApplicationEvents = new LinkedHashSet<>();
    }
```



#### 5.4 加载 BeanFactory

- 经过 obtainFreshBeanFactory 函数后 ApplicationContext 就已经拥有了 BeanFactory 的全部功能。
- 核心实现委托给了 refreshBeanFactory：
  - （1）创建 DefaultListableBeanFactory。DefaultListableBeanFactory 是容器的基础，必须首先要实例化。
  - （2）指定序列化 Id。beanFactory.setSerializationId(getId());
  - （3）定制 BeanFactory。customizeBeanFactory(beanFactory);
  - （4）加载 BeanDefinition。loadBeanDefinitions(beanFactory);
  - （5）使用全局变量记录 BeanFactory 类实例。this.beanFactory = beanFactory;

```java
	protected ConfigurableListableBeanFactory obtainFreshBeanFactory() {
		// 初始化 BeanFactory，并进行 XML文件读取，并将得到的 BeanFactory 记录在当前实体的属性中
		refreshBeanFactory();
		// 返回当前实体的 beanFactory 属性
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (logger.isDebugEnabled()) {
			logger.debug("Bean factory for " + getDisplayName() + ": " + beanFactory);
		}
		return beanFactory;
	}

	// 上述方法的核心实现委托给了 refreshBeanFactory（AbstractRefreshableApplicationContext.java）
	protected final void refreshBeanFactory() throws BeansException {
		if (hasBeanFactory()) {
			destroyBeans();
			closeBeanFactory();
		}
		try {
			// 创建 DefaultListableBeanFactory
			DefaultListableBeanFactory beanFactory = createBeanFactory();
			// 为了序列号指定 id，如果需要的话，让这个 BeanFactory 从 id 反序列化到 BeanFactory 对象
			beanFactory.setSerializationId(getId());
			// 定制 beanFactory，设置相关属性，包括是否允许覆盖同名称的不同定义的对象以及循环依赖
			// 以及设置 @Autowired 和 @Qualifier 注解解析器 QualifierAnnotationAutowireCandidateResolver
			customizeBeanFactory(beanFactory);
			// 初始化 DocumentReader，并进行 XML 文件读取及解析
			loadBeanDefinitions(beanFactory);
			synchronized (this.beanFactoryMonitor) {
				this.beanFactory = beanFactory;
			}
		}
		catch (IOException ex) {
			throw new ApplicationContextException("I/O error parsing bean definition source for " + getDisplayName(), ex);
		}
	}
	
```



##### 5.4.1 定制 BeanFactory

- 在基础容器的基础上，增加了是否允许覆盖、是否允许扩展的设置并提供了注解 @Qualifier 和 @Autowired 的支持。

```java
	protected void customizeBeanFactory(DefaultListableBeanFactory beanFactory) {
		// 如果 allowBeanDefinitionOverriding 属性不为空，设置给 beanFactory 对象相应属性，
		// 此属性的含义：是否允许覆盖同名称的不同定义的对象
		if (this.allowBeanDefinitionOverriding != null) {
			beanFactory.setAllowBeanDefinitionOverriding(this.allowBeanDefinitionOverriding);
		}
		// 如果属性 allowCircularReferences 不为空，设置给 bean 对象相应属性，
		// 此属性的含义：是否允许 bean 之间存在循环依赖
		if (this.allowCircularReferences != null) {
			beanFactory.setAllowCircularReferences(this.allowCircularReferences);
		}
	}
```

- 对于允许覆盖和允许依赖的设置，是通过子类覆盖方法进行设置的，方便扩展。如：

```java
public class MyClassPathXmlApplicationContext extends ClassPathXmlApplicationContext {
    ......
    protected void customizeBeanFactory(DefaultListableBeaFactory beanFactory) {
        super.seetAllowBeanDefinitionOverriding(false);
        super.setAllowCircularReferences(false);
        super.customizeBeanFactory(beanFactory);
    }
}
```



##### 5.4.2 加载 BeanDefinition

- 已经初始化的 DefaultListableBeaFactory 还需要 XMLBeanDefinitionReader 来读取 XML，在这个步骤中首先要做的是初始化 XMLBeanDefinitionReader。

AbstractXmlApplicationContext.java

```JAVA
	protected void loadBeanDefinitions(DefaultListableBeanFactory beanFactory) throws BeansException, IOException {
		// Create a new XmlBeanDefinitionReader for the given BeanFactory.
		// 为指定的 beanFactory 创建 XmlBeanDefinitionReader
		XmlBeanDefinitionReader beanDefinitionReader = new XmlBeanDefinitionReader(beanFactory);

		// Configure the bean definition reader with this context's
		// resource loading environment.
		// 对 beanDefinitionReader 进行环境变量的设置
		beanDefinitionReader.setEnvironment(this.getEnvironment());
		beanDefinitionReader.setResourceLoader(this);
		beanDefinitionReader.setEntityResolver(new ResourceEntityResolver(this));

		// Allow a subclass to provide custom initialization of the reader,
		// then proceed with actually loading the bean definitions.
		// 对 beanDefinitionReader 进行设置，可以覆盖
		initBeanDefinitionReader(beanDefinitionReader);
		loadBeanDefinitions(beanDefinitionReader);
	}

	// 在初始化了 DefaultListableBeanFactory 和 XmlBeanDefinitionReader 后就可以进行配置文件的读取了
	protected void loadBeanDefinitions(XmlBeanDefinitionReader reader) throws BeansException, IOException {
		Resource[] configResources = getConfigResources();
		if (configResources != null) {
			reader.loadBeanDefinitions(configResources);
		}
		String[] configLocations = getConfigLocations();
		if (configLocations != null) {
			reader.loadBeanDefinitions(configLocations);
		}
	}
```





#### 5.5 功能扩展

* Spring  完成了对配置的解析， prepareBeanFactory 函数对 BeanFactory 进行各种功能填充。 
* prepareBeanFactory 函数主要进行了几个方面的扩展。
  * 增加了对 SpEL 语言的支持。
  * 增加对属性编辑器的支持。
  * 增加对一些内置类，如 EnvironmentAware、MessageSourceAware 的信息注入。
  * 设置了依赖功能可忽略的接口。
  * 注册一些固定依赖的属性。
  * 增加 AspectJ 的支持。
  * 将相关环境变量即属性注册以单例模式注册。

```java
	protected void prepareBeanFactory(ConfigurableListableBeanFactory beanFactory) {
		// Tell the internal bean factory to use the context's class loader etc.
		// 设置 beanFactory 的 classLoader 为当前 context 的 classLoader
		beanFactory.setBeanClassLoader(getClassLoader());
		// 设置 beanFactory 的表达式语言处理器，Spring3 增加了表达式语言的支持
		// 默认可以使用 #{bean.xxx} 的形式来调用相关属性（SpEL）
		beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader()));
		// 为 beanFactory 增加了一个默认的 propertyEditor，这个主要是对 bean 的属性等设置管理的一个工具
		beanFactory.addPropertyEditorRegistrar(new ResourceEditorRegistrar(this, getEnvironment()));

		// Configure the bean factory with context callbacks.
		// 添加 BeanPostProcessor
		beanFactory.addBeanPostProcessor(new ApplicationContextAwareProcessor(this));
		// 设置了几个忽略自动装配的接口
		beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
		beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
		beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
		beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
		beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);

		// BeanFactory interface not registered as resolvable type in a plain factory.
		// MessageSource registered (and found for autowiring) as a bean.
		// 设置了几个自动装配的特殊规则
		beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
		beanFactory.registerResolvableDependency(ResourceLoader.class, this);
		beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
		beanFactory.registerResolvableDependency(ApplicationContext.class, this);

		// Register early post-processor for detecting inner beans as ApplicationListeners.
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(this));

		// Detect a LoadTimeWeaver and prepare for weaving, if found.
		// 增加对 AspectJ 的支持
		if (beanFactory.containsBean(LOAD_TIME_WEAVER_BEAN_NAME)) {
			beanFactory.addBeanPostProcessor(new LoadTimeWeaverAwareProcessor(beanFactory));
			// Set a temporary ClassLoader for type matching.
			beanFactory.setTempClassLoader(new ContextTypeMatchClassLoader(beanFactory.getBeanClassLoader()));
		}

		// Register default environment beans.
		// 添加默认的系统环境 bean
		if (!beanFactory.containsLocalBean(ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(ENVIRONMENT_BEAN_NAME, getEnvironment());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_PROPERTIES_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_PROPERTIES_BEAN_NAME, getEnvironment().getSystemProperties());
		}
		if (!beanFactory.containsLocalBean(SYSTEM_ENVIRONMENT_BEAN_NAME)) {
			beanFactory.registerSingleton(SYSTEM_ENVIRONMENT_BEAN_NAME, getEnvironment().getSystemEnvironment());
		}
	}
```



##### 5.5.1 增加 SpEL 语言的支持

- SpEL 能在运行时构建复杂表达式、存取对象图属性、对象方法调用等，并且能与 Spring 功能完美融合，比如能用来配置 bean 定义。**SpEL 是单独模块，只能依赖于 core 模块，不能依赖于其他模块，可以单独使用。**
- 在源码中通过代码 beanFactory.setBeanExpressionResolver(new StandardBeanExpressionResolver(beanFactory.getBeanClassLoader())); 注册语言解析器，就可以对 SpEL 进行解析了。
- 注册解析器后 Spring 在什么时候调用这个解析器进行解析呢？Spring 在 bean 进行初始化的时候会有**属性填充**的一步，而在这一步中 Spring 会调用 AbstractAutowireCapableBeanFactory 类的 applyPropertyValues 函数来完成功能。在这个函数中，会通过构造 BeanDefinitionValueResolver 类型实例 valueResolver 来进行属性值的解析。同时，也是在这个步骤中一般通过 AbstractBeanFactory 中的 **evaluateBeanDefinitionString** 方法去完成 SpEL 的解析。
- 当调用这个方法时会判断是否存在语言解析器，如果存在则调用语言解析器的方法进行解析，解析的过程是在 Spring 的 expression 包内。通过查看 evaluateBeanDefinitionString 方法的调用层次可以看出，**应用语言解析器的调用主要是在解析依赖注入 bean 的时候，以及在完成 bean 的初始化和属性获取后进行属性填充的时候。**

```java
	protected Object evaluateBeanDefinitionString(@Nullable String value, @Nullable BeanDefinition beanDefinition) {
		if (this.beanExpressionResolver == null) {
			return value;
		}

		Scope scope = null;
		if (beanDefinition != null) {
			String scopeName = beanDefinition.getScope();
			if (scopeName != null) {
				scope = getRegisteredScope(scopeName);
			}
		}
		return this.beanExpressionResolver.evaluate(value, new BeanExpressionContext(this, scope));
	}
```



##### 5.5.2 增加属性注册编辑器

- Spring DI 注入的时候可以把普通属性注入进来，但是像 Date 类型就无法被识别。直接注入的话，在 Java 的 date 属性是 Date 类型的，而在 XML 中配置的却是 String 类型的，所以会报异常。
- Spring 针对此问题提供了两种解决方法。

###### 1.使用自定义属性编辑器。

- 使用自定义属性编辑器，通过继承 PropertyEditorSupport，重写 setAsText 方法，具体如下。

(1)编写自定义的属性编辑器

```java
	public class DatePropertyEditor extends PropertyEditorSupport {
        private String format = "yyyy-MM-dd";
        public void setFormat(String format) {
            this.format = format;
        }
        
        @Override
        public void setAsTest(String args) throws IllegalArgumentException {
            System.out.println("arg0: " + arg0);
            SimpleDateFormat sdf = new SimpleDateFormat(format);
            try {
                Date date = sdf.parse(arg0);
                this.setValue(date);
            } catch(ParseException e) {
                e.printStackTrace();
            }
        }
    }
	
	
```



（2）将自定义属性编辑器注册到 Spring 中。

```xml
<!-- 自定义属性编辑器 -->
<bean class = "org.Springframework.beans.factory.config.CustomEditorConfigurer">
	<property name = "customEditors">
    	<map>
        	<entry key = "java.util.Date">
            	<bean class = "com.test.DatePropertyEditor">
                	<property name = "format" value = "yyyy-MM-dd"/>
                </bean>
            </entry>
        </map>
    </property>
</bean>
```



###### 2.注册 Spring 自带的属性编辑器 CustomDateEditor

- 通过注册 Spring 自带的属性编辑器 CustomDateEditor，具体步骤如下：

（1）定义属性编辑器

```java
public class DatePropertyEditorRegistrar implements PropertyEditorRegistrar {
    public void registerCustomEditors(PropertyEditorRegistry registry) {
        registry.registerCustomEditor(Date.class, new CustomDateEditor(new SimpleDateFormat("yyyy-MM-dd"), true));
    }
}
```

（2）注册到 Spring 中。

```xml
<!-- 注册 Spring 自带编辑器 -->
<bean class = "org.Springframework.beans.factory.config.CustomEditorConfigurer">
	<property name = "propertyEditorRegistrars">
    	<list>
        	<bean class = "com.test.DatePropertyEditorRegistrar"></bean>
        </list>
    </property>
</bean>
```



- 在 bean 的初始化后会调用 ResourceEditorRegistrar 的 registerCustomEditors 方法进行批量的通用属性编辑器注册。注册后，在属性填充的环节便可以直接让 Spring 使用这些编辑器进行属性的解析了。

- Spring 中定义了一系列常用的属性编辑器使我们可以方便地进行配置。如果我们定义的 bean 中的某个属性的类型不在上面的常用配置中的话，才需要我们进行个性化属性编辑器的注册。



##### 5.5.3 添加 ApplicationContextAwareProcessor 处理器

- ApplicationContextAwareProcessor  中的 postProcessBeforeInitialization 和 postProcessAfterInitialization 方法。
- 关于 postProcessAfterInitialization 方法，在 ApplicationContextAwareProcessor 并没有做处理。

```java
	@Override
	public Object postProcessAfterInitialization(Object bean, String beanName) {
		return bean;
	}
```

- 重点在 postProcessBeforeInitialization  方法。

```java
	public Object postProcessBeforeInitialization(final Object bean, String beanName) throws BeansException {
		AccessControlContext acc = null;

		if (System.getSecurityManager() != null &&
				(bean instanceof EnvironmentAware || bean instanceof EmbeddedValueResolverAware ||
						bean instanceof ResourceLoaderAware || bean instanceof ApplicationEventPublisherAware ||
						bean instanceof MessageSourceAware || bean instanceof ApplicationContextAware)) {
			acc = this.applicationContext.getBeanFactory().getAccessControlContext();
		}
		
        // 调用了 invokeAwareInterfaces(bean); 方法
		if (acc != null) {
			AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
				invokeAwareInterfaces(bean);
				return null;
			}, acc);
		}
		else {
			invokeAwareInterfaces(bean);
		}

		return bean;
	}
	
	// 实现下列 Aware 接口的 bean 在被初始化之后，可以取得一些对应的资源。
	private void invokeAwareInterfaces(Object bean) {
		if (bean instanceof Aware) {
			if (bean instanceof EnvironmentAware) {
				((EnvironmentAware) bean).setEnvironment(this.applicationContext.getEnvironment());
			}
			if (bean instanceof EmbeddedValueResolverAware) {
				((EmbeddedValueResolverAware) bean).setEmbeddedValueResolver(this.embeddedValueResolver);
			}
			if (bean instanceof ResourceLoaderAware) {
				((ResourceLoaderAware) bean).setResourceLoader(this.applicationContext);
			}
			if (bean instanceof ApplicationEventPublisherAware) {
				((ApplicationEventPublisherAware) bean).setApplicationEventPublisher(this.applicationContext);
			}
			if (bean instanceof MessageSourceAware) {
				((MessageSourceAware) bean).setMessageSource(this.applicationContext);
			}
			if (bean instanceof ApplicationContextAware) {
				((ApplicationContextAware) bean).setApplicationContext(this.applicationContext);
			}
		}
	}
```



##### 5.5.4 设置忽略依赖

- 当 Spring 将 ApplicationContextAwareProcessor 注册后，那么在 invokeAwareInterfaces 方法中间接调用的 Aware 类已经不是普通的 bean 了，如 ResourceLoaderAware、ApplicationContextAware、MessageSourceAware 等，需要在 Spring 做 bean 的依赖注入的时候忽略它们。ignoreDependencyInterface 的作用正在于此。

```java
    // 设置了几个忽略自动装配的接口
    beanFactory.ignoreDependencyInterface(EnvironmentAware.class);
    beanFactory.ignoreDependencyInterface(EmbeddedValueResolverAware.class);
    beanFactory.ignoreDependencyInterface(ResourceLoaderAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationEventPublisherAware.class);
    beanFactory.ignoreDependencyInterface(MessageSourceAware.class);
    beanFactory.ignoreDependencyInterface(ApplicationContextAware.class);
```

##### 5.5.5 注册依赖

- 当注册了依赖解析后，例如当注册了对 BeanFactory.class 的解析依赖后，当 bean 的属性注入的时候，一旦检测到属性为 BeanFactory 类型便会将 beanFactory 的实例注入进行。

```java
    // BeanFactory interface not registered as resolvable type in a plain factory.
    // MessageSource registered (and found for autowiring) as a bean.
    // 设置了几个自动装配的特殊规则
    beanFactory.registerResolvableDependency(BeanFactory.class, beanFactory);
    beanFactory.registerResolvableDependency(ResourceLoader.class, this);
    beanFactory.registerResolvableDependency(ApplicationEventPublisher.class, this);
    beanFactory.registerResolvableDependency(ApplicationContext.class, this);
```



#### 5.6 *BeanFactory 的后置处理器



##### 5.6.1 激活和注册的 BeanFactoryPostProcessor

- BeanFactoryPostProcessor 的用法。
  - BeanFactoryPostProcessor 接口跟 BeanPostProcessor 类似，可以对 bean 的定义（配置元数据）进行处理。即 Spring IoC 容器允许 BeanFactoryPostProcessor 在容器实际实例化任何其他的 bean 之前读取配置元数据，并有可能修改它。
  - 可以配置多个 BeanFactoryPostProcessor，还能通过设置 “order” 属性来控制 BeanFactoryPostProcessor 的执行次序（**仅当 BeanFactoryPostProcessor 实现了 Ordered 接口时才可以设置此属性，**因此在实现 BeanFactoryPostProcessor 时，就应当考虑实现 Ordered 接口）。
  - 如果想改变实际的 bean 实例（例如从配置元数据创建的对象），最好使用 BeanPostProcessor。
  - BeanFactoryPostProcessor 的作用域范围是容器级的，它只和所使用的容器有关。如果在容器中定义一个 BeanFactoryPostProcessor，它仅仅对此容器中的 bean 进行后置处理。BeanFactoryPostProcessor 不会对定义在另一个容器中的 bean 进行后置处理，即使这两个容器都是在同一层次上。



###### 1.BeanFactoryPostProcessor 的典型应用：PropertyPlaceholderConfigurer

（1）示例：

```xml
<!-- 配置 -->
<bean id = "message" class = "distConfig.HelloMessage">
	<property name = "mes">
    	<value>${bean.message}</value>
    </property>
</bean>
```

bean.properties 配置如下：

```properties
bean.message = Hi, can you find me?
```

当访问名为 message 的 bean 时，mes 属性就会被置为 bean.message 的值。Spring 通过 PropertyPlaceholderConfigurer 这个类的 bean 来确定存在 bean.properties 配置文件：

```xml
<bean id = "mesHandler" class ="org.Springframework.beans.factory.config.PropertyPlaceholderConfigurer">
	<property name = "locations">
    	<list>
        	<value>config/bean.properties</value>
        </list>
    </property>
</bean>
```

- Spring 的 beanFactory 是怎么知道要从这个 bean 中获取配置信息的呢？
  - 查看层级结构可以看出 PropertyPlaceholderConfigurer 这个类间接继承了 BeanFactoryPostProcessor 接口。当 Spring 加载任何实现了这个接口的 bean 的配置时，都会在 bean 工厂载入所有 bean 的配置之后执行 postProcessBeanFactory 方法。在 PropertyResourceConfigurer（PropertyPlaceholderConfigurer 的父类） 类中实现了 postProcessBeanFactory 方法，在方法中先后调用了 mergeProperties、convertProperties、processProperties 这三个方法，分别得到配置，将得到的配置转换为合适的类型，最后将配置内容告知 BeanFactory。



###### 2.使用自定义 BeanFactoryPostProcessor

- 通过 ObscenityRemovingBeanFactoryPostProcessor 实现屏蔽掉  obscenties 定义的不应该展示的属性。

配置文件 applicationContext.xml

```xml
<bean id = "bfpp" class = "processor.ObscenityRemovingBeanFactoryPostProcessor">
    <property name="obscenties">
        <set>
            <value>bollocks</value>
            <value>winky</value>
            <value>bum</value>
            <value>Microsoft</value>
        </set>
    </property>
</bean>
<bean id = "simpleBean" class = "processor.SimplePostProcessor">
    <property name="connectionString" value="bollocks" />
    <property name="password" value="imagineCup" />
    <property name="username" value="Microsoft" />
</bean>
```

ObscenityRemovingBeanFactoryPostProcessor.java

```java
public class ObscenityRemovingBeanFactoryPostProcessor implements BeanFactoryPostProcessor {

	private Set<String> obscenties;
	public ObscenityRemovingBeanFactoryPostProcessor() {
		this.obscenties = new HashSet<String>();
	}

	@Override
	public void postProcessBeanFactory(ConfigurableListableBeanFactory beanFactory) throws BeansException {
		String[] beanNames = beanFactory.getBeanDefinitionNames();
		for (String beanName : beanNames) {
			BeanDefinition bd = beanFactory.getBeanDefinition(beanName);
			StringValueResolver valueResolver = new StringValueResolver() {
				@Override
				public String resolveStringValue(String strVal) {
					if (isObscene(strVal)) return "******";
					return strVal;
				}
			};
			BeanDefinitionVisitor visitor = new BeanDefinitionVisitor(valueResolver);
			visitor.visitBeanDefinition(bd);
		}
	}

	public  boolean isObscene(Object value) {
		String potentialObscenity = value.toString().toUpperCase();
		return this.obscenties.contains(potentialObscenity);
	}

	public void setObscenties(Set<String> obscenties) {
		this.obscenties.clear();
		for (String  obscenity : obscenties) {
			this.obscenties.add(obscenity.toUpperCase());
		}
	}
}
```

SimplePostProcessor.java

```java
public class SimplePostProcessor {

	private String connectionString;

	private String password;

	private String username;

	public String getConnectionString() {
		return connectionString;
	}

	public void setConnectionString(String connectionString) {
		this.connectionString = connectionString;
	}

	public String getPassword() {
		return password;
	}

	public void setPassword(String password) {
		this.password = password;
	}

	public String getUsername() {
		return username;
	}

	public void setUsername(String username) {
		this.username = username;
	}

	@Override
	public String toString() {
		return "SimplePostProcessor{" +
				"connectionString='" + connectionString + '\'' +
				", password='" + password + '\'' +
				", username='" + username + '\'' +
				'}';
	}
}
```

执行类：

```java
public class PropertyConfigurerDemo {
	public static void main(String[] args) {
		// 不能用 ApplicationContext context = new ClassPathXmlApplicationContext("applicationContext.xml");
		ConfigurableListableBeanFactory bf = new XmlBeanFactory(new ClassPathResource("applicationContext.xml"));
		BeanFactoryPostProcessor bfpp = (BeanFactoryPostProcessor)bf.getBean("bfpp");
		bfpp.postProcessBeanFactory(bf);
		System.out.println(bf.getBean("simpleBean"));
	}
}
```

输出结果：

```java
SimplePostProcessor{connectionString='******', password='imagineCup', username='******'}
```



###### 3.激活 BeanFactoryPostProcessor

- 对于 BeanFactoryPostProcessor 的处理主要分两种情况进行，一个是对于 BeanDefinitionRegistry 类的特殊处理，另一种是对普通的 BeanFactoryPostProcessor 进行处理。而对于每种情况都要考虑硬编码注入注册的后置处理器以及通过配置注入的后置处理器。

 PostProcessorRegistrationDelegate.java

```java
	public static void invokeBeanFactoryPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanFactoryPostProcessor> beanFactoryPostProcessors) {

		// Invoke BeanDefinitionRegistryPostProcessors first, if any.
		Set<String> processedBeans = new HashSet<>();

		// 对 BeanDefinitionRegistry 类型的处理
		if (beanFactory instanceof BeanDefinitionRegistry) {
			BeanDefinitionRegistry registry = (BeanDefinitionRegistry) beanFactory;
			List<BeanFactoryPostProcessor> regularPostProcessors = new ArrayList<>();
			List<BeanDefinitionRegistryPostProcessor> registryProcessors = new ArrayList<>();

			// 硬编码注册的后置处理器
			for (BeanFactoryPostProcessor postProcessor : beanFactoryPostProcessors) {
				if (postProcessor instanceof BeanDefinitionRegistryPostProcessor) {
					BeanDefinitionRegistryPostProcessor registryProcessor =
							(BeanDefinitionRegistryPostProcessor) postProcessor;
					// 对于 BeanDefinitionRegistryPostProcessor 类型，在 BeanFactoryPostProcessor
					// 的基础上还有自己定义的方法需要先调用
					registryProcessor.postProcessBeanDefinitionRegistry(registry);
					registryProcessors.add(registryProcessor);
				}
				else {
					// 记录常规 BeanFactoryPostProcessor
					regularPostProcessors.add(postProcessor);
				}
			}

			// Do not initialize FactoryBeans here: We need to leave all regular beans
			// uninitialized to let the bean factory post-processors apply to them!
			// Separate between BeanDefinitionRegistryPostProcessors that implement
			// PriorityOrdered, Ordered, and the rest.
			List<BeanDefinitionRegistryPostProcessor> currentRegistryProcessors = new ArrayList<>();

			// First, invoke the BeanDefinitionRegistryPostProcessors that implement PriorityOrdered.
			String[] postProcessorNames =
					beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			// 对后置处理器进行分类
			for (String ppName : postProcessorNames) {
				if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					// 保存已经加载过的 Processor
					processedBeans.add(ppName);
				}
			}
			// 对 currentRegistryProcessors 进行排序
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			// 注册优先级 Processor
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Next, invoke the BeanDefinitionRegistryPostProcessors that implement Ordered.
			postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
			for (String ppName : postProcessorNames) {
				if (!processedBeans.contains(ppName) && beanFactory.isTypeMatch(ppName, Ordered.class)) {
					currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
					processedBeans.add(ppName);
				}
			}
			sortPostProcessors(currentRegistryProcessors, beanFactory);
			registryProcessors.addAll(currentRegistryProcessors);
			invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
			currentRegistryProcessors.clear();

			// Finally, invoke all other BeanDefinitionRegistryPostProcessors until no further ones appear.
			boolean reiterate = true;
			while (reiterate) {
				reiterate = false;
				postProcessorNames = beanFactory.getBeanNamesForType(BeanDefinitionRegistryPostProcessor.class, true, false);
				for (String ppName : postProcessorNames) {
					if (!processedBeans.contains(ppName)) {
						currentRegistryProcessors.add(beanFactory.getBean(ppName, BeanDefinitionRegistryPostProcessor.class));
						processedBeans.add(ppName);
						reiterate = true;
					}
				}
				sortPostProcessors(currentRegistryProcessors, beanFactory);
				registryProcessors.addAll(currentRegistryProcessors);
				invokeBeanDefinitionRegistryPostProcessors(currentRegistryProcessors, registry);
				currentRegistryProcessors.clear();
			}

			// Now, invoke the postProcessBeanFactory callback of all processors handled so far.
			// 调用到目前为止处理的所有处理器的 postProcessBeanFactory 回调
			invokeBeanFactoryPostProcessors(registryProcessors, beanFactory);
			invokeBeanFactoryPostProcessors(regularPostProcessors, beanFactory);
		}

		else {
			// Invoke factory processors registered with the context instance.
			invokeBeanFactoryPostProcessors(beanFactoryPostProcessors, beanFactory);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let the bean factory post-processors apply to them!
		String[] postProcessorNames =
				beanFactory.getBeanNamesForType(BeanFactoryPostProcessor.class, true, false);

		// Separate between BeanFactoryPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		List<BeanFactoryPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		List<String> orderedPostProcessorNames = new ArrayList<>();
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (processedBeans.contains(ppName)) {
				// skip - already processed in first phase above
			}
			else if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				priorityOrderedPostProcessors.add(beanFactory.getBean(ppName, BeanFactoryPostProcessor.class));
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, invoke the BeanFactoryPostProcessors that implement PriorityOrdered.
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(priorityOrderedPostProcessors, beanFactory);

		// Next, invoke the BeanFactoryPostProcessors that implement Ordered.
		List<BeanFactoryPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String postProcessorName : orderedPostProcessorNames) {
			orderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		invokeBeanFactoryPostProcessors(orderedPostProcessors, beanFactory);

		// Finally, invoke all other BeanFactoryPostProcessors.
		List<BeanFactoryPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String postProcessorName : nonOrderedPostProcessorNames) {
			nonOrderedPostProcessors.add(beanFactory.getBean(postProcessorName, BeanFactoryPostProcessor.class));
		}
		invokeBeanFactoryPostProcessors(nonOrderedPostProcessors, beanFactory);

		// Clear cached merged bean definitions since the post-processors might have
		// modified the original metadata, e.g. replacing placeholders in values...
		beanFactory.clearMetadataCache();
	}
```





##### 5.6.2 注册 BeanPostProcessor

- 探索 BeanPostProcessor，不是调用，而是注册。**真正的调用是在 bean 的实例化阶段进行的，这是一个很重要的阶段，也是很多功能 BeanFactory 不支持的重要原因。**Spring 中大部分功能都是通过后置处理器的方式进行扩展的，这是 Spring 框架的一个特性，但是**在 BeanFactory 中并没有实现后置处理器的自动注册**，所以在调用的时候如果没有进行手动注册是不能使用的。
- 但是在 ApplicationContext 中却添加了自动注册功能，如下：

（1）定义一个后置处理器：

```java
public class MyInstantiationAwareBeanPostProcessor implements InstantiationAwareBeanPostProcessor {
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        System.out.println("=======");
        return null;
    }
    ......
}
```

（2）在配置文件中添加配置：

```xml
<bean class="processors.MyInstantiationAwareBeanPostProcessor" />
```

- 则使用 BeanFactory 方式进行 Spring 的 bean 的加载时是不会有任何变化的，但是使用 ApplicationContext 方式获取 bean 的时候会在获取每个 bean 时打印出 "==="，而这个特性就是在 registerBeanPostProcessor 方法中完成的。

 PostProcessorRegistrationDelegate.java

```java
	public static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, AbstractApplicationContext applicationContext) {

		String[] postProcessorNames = beanFactory.getBeanNamesForType(BeanPostProcessor.class, true, false);

		// Register BeanPostProcessorChecker that logs an info message when
		// a bean is created during BeanPostProcessor instantiation, i.e. when
		// a bean is not eligible for getting processed by all BeanPostProcessors.
		/*
		 *	BeanPostProcessorChecker 是一个普通的信息打印，可能会有些情况，
		 * 	当 Spring 的配置中的后置处理器还没有被注册就已经开始了 bean 的初始化时，
		 * 	便会打印出 BeanPostProcessorChecker 中设定的信息
		 */
		int beanProcessorTargetCount = beanFactory.getBeanPostProcessorCount() + 1 + postProcessorNames.length;
		beanFactory.addBeanPostProcessor(new BeanPostProcessorChecker(beanFactory, beanProcessorTargetCount));

		// Separate between BeanPostProcessors that implement PriorityOrdered,
		// Ordered, and the rest.
		// 使用 PriorityOrdered 保证顺序
		List<BeanPostProcessor> priorityOrderedPostProcessors = new ArrayList<>();
		// MergedBeanDefinitionPostProcessor
		List<BeanPostProcessor> internalPostProcessors = new ArrayList<>();
		// 使用 Ordered 保证顺序
		List<String> orderedPostProcessorNames = new ArrayList<>();
		// 无序 BeanPostProcessor 
		List<String> nonOrderedPostProcessorNames = new ArrayList<>();
		for (String ppName : postProcessorNames) {
			if (beanFactory.isTypeMatch(ppName, PriorityOrdered.class)) {
				BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
				priorityOrderedPostProcessors.add(pp);
				if (pp instanceof MergedBeanDefinitionPostProcessor) {
					internalPostProcessors.add(pp);
				}
			}
			else if (beanFactory.isTypeMatch(ppName, Ordered.class)) {
				orderedPostProcessorNames.add(ppName);
			}
			else {
				nonOrderedPostProcessorNames.add(ppName);
			}
		}

		// First, register the BeanPostProcessors that implement PriorityOrdered.
		// 第一步，注册所有实现 PriorityOrdered 的 BeanPostProcessor
		sortPostProcessors(priorityOrderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, priorityOrderedPostProcessors);

		// Next, register the BeanPostProcessors that implement Ordered.
		// 第二步，注册所有实现 Ordered 的 BeanPostProcessor
		List<BeanPostProcessor> orderedPostProcessors = new ArrayList<>();
		for (String ppName : orderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			orderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		sortPostProcessors(orderedPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, orderedPostProcessors);

		// Now, register all regular BeanPostProcessors.
		// 第三步，注册所有无序的 BeanPostProcessor
		List<BeanPostProcessor> nonOrderedPostProcessors = new ArrayList<>();
		for (String ppName : nonOrderedPostProcessorNames) {
			BeanPostProcessor pp = beanFactory.getBean(ppName, BeanPostProcessor.class);
			nonOrderedPostProcessors.add(pp);
			if (pp instanceof MergedBeanDefinitionPostProcessor) {
				internalPostProcessors.add(pp);
			}
		}
		registerBeanPostProcessors(beanFactory, nonOrderedPostProcessors);

		// Finally, re-register all internal BeanPostProcessors.
		// 第四步，注册所有实现 MergedBeanDefinitionPostProcessor 类型 的 BeanPostProcessor，
		// 并非重复注册，在 beanFactory.addBeanPostProcessor 中会先移除已经存在的 BeanPostProcessor
		sortPostProcessors(internalPostProcessors, beanFactory);
		registerBeanPostProcessors(beanFactory, internalPostProcessors);

		// Re-register post-processor for detecting inner beans as ApplicationListeners,
		// moving it to the end of the processor chain (for picking up proxies etc).
		// 添加 ApplicationListenerDetector 探测器
		beanFactory.addBeanPostProcessor(new ApplicationListenerDetector(applicationContext));
	}


	// registerBeanPostProcessors 方法的实现方式：
	private static void registerBeanPostProcessors(
			ConfigurableListableBeanFactory beanFactory, List<BeanPostProcessor> postProcessors) {

		for (BeanPostProcessor postProcessor : postProcessors) {
			beanFactory.addBeanPostProcessor(postProcessor);
		}
	}
	
	public void addBeanPostProcessor(BeanPostProcessor beanPostProcessor) {
		Assert.notNull(beanPostProcessor, "BeanPostProcessor must not be null");
		// Remove from old position, if any
		this.beanPostProcessors.remove(beanPostProcessor);
		// Track whether it is instantiation/destruction aware
		if (beanPostProcessor instanceof InstantiationAwareBeanPostProcessor) {
			this.hasInstantiationAwareBeanPostProcessors = true;
		}
		if (beanPostProcessor instanceof DestructionAwareBeanPostProcessor) {
			this.hasDestructionAwareBeanPostProcessors = true;
		}
		// Add to end of list
		this.beanPostProcessors.add(beanPostProcessor);
	}
```

- 对于 BeanPostProcessor 的处理与 BeanFactoryPostProcessor 的处理极为相似，但也有不同。对于 BeanFactoryPostProcessor 的处理要区分两种情况，一种方式是通过硬编码方式的处理，另一种是通过配置文件方法的处理。而在 BeanPostProcessor 的处理中只考虑了配置文件的方式。**对于 BeanFactoryPostProcessor 的处理，不但要实现注册功能，而且还要实现对后置处理器的激活操作，所以需要载入配置中的定义，并进行激活；而对于 BeanPostProcessor 并不需要马上调用，再者，硬编码的方式实现的功能是将后置处理器提取并调用，这里并不需要调用，就不需要考虑硬编码的方法了，这里的功能只是将配置文件的 BeanPostProcessor 提取处理啊并注册进入 beanFactory 就可以了。**
- 在 registerBeanPostProcessors 方法的实现中已经确保了 BeanPostProcessor 的唯一性。



##### 5.6.3 初始化消息资源

- Spring 国际化的使用。
- “国际化信息”也成为“本地化信息”，一般需要两个条件才可以确定一个特定类型的本地化信息，分别是“语言类型”和“国家/地区的类型”。如中文本地信息既有中国大陆地区的中文，又有中国台湾地区、中国香港地区的中文，还有新加坡地区的中文。Java 通过 java.util.Locale 类表示一个本地化对象，它允许通过语言参数和国家/地区参数创建一个确定的本地化对象。
- JDK 的 java.util 提供了几个支持本地化的格式化操作工具类：NumberFormat、DateFormat、MessageFormat，在 Spring 中的国际化资源操作也是对于这些类的封装操作。
- MessageFormat 的用法：

```java
// 1.信息格式化串
String pattern1 = "{0}，你好！你于{1}在工商银行存入{2}元。";
String pattern2 = "At {1,time,short} On {1,date,long}, {0} paid {2,number,currency}.";

// 2.用于动态替换占位符的参数
Object[] params = {"John", new GregorianCalendar().getTime(), 1.0E3};

// 3.使用默认本地化对象格式化信息
String msg1 = MessageFormat.format(pattern1, params);

// 4.使用指定的本地化对象格式化信息
MessageFormat mf = new MessageFormat(pattern2, Locale.US);
String msg2 = mf.format(params);
System.out.println(msg1);
System.out.println(msg2);
```

-  **HierarchicalMessageSource 接口最重要的两个实现类是  ResourceBundleMessageSource 和 ReloadableResourceBundleMessageSource。**它们基于 Java 的 ResourceBundle 基础类实现，允许仅通过资源名加载国际化资源。ReloadableResourceBundleMessageSource 提供了定时刷新功能，允许在不重启系统的情况下，更新资源。StaticMessageSource 主要用于程序测试，它允许通过编程的方式提供国际化信息。而 DelegatingMessageSource 是为方便操作父 MessageSource 而提供的代理类。

![MessageSource](E:\学习笔记\Spring\images\MessageSource.png)

- ResourceBundleMessageSource 的实现方式。

（1）定义资源文件

messages.properties(默认：英文)，内容仅一句，如下：

test = test

messages_zh_CN.properties（简体中文）：

test = 测试

（2）定义配置文件。

```xml 
<!-- 这个 bean 的 id 必须命名为 messageSource，否则会抛出 NoSuchMessageException 异常。 -->
<bean id="messageSource" class="org.springframework.context.support.ResourceBundleMessageSource">
	<property name="basenames">
    	<list>
        	<value>test/messages</value>
        </list>
    </property>
</bean>
```

（3）使用。通过 ApplicationContext 访问国际化信息。

```java
String[] configs = {"applicationContext.xml"};
ApplicationContext ctx = new ClassPathXmlApplicationContext(configs);
// 1.直接通过容器访问国际化信息
Object[] params = {"John", new GregorianCalendar().getTime()};

String str1 = ctx.getMessage("test", params, Locale.US);
String str2 = ctx.getMessage("test", params, Locale.CHINA);
System.out.println(str1);
System.out.println(str2);
```



- **在 initMessageSource 中的方法主要功能是提取配置中定义的 messageSource，并将其记录在 Spring 容器中，也就是 AbstractApplicationContext 中。**如果用户未设置资源文件的话，Spring 中也提供了默认的配置 DelegatingMessageSource。
- 在 initMessageSource 中获取自定义资源文件的方式为 beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class)，在这里 Spring 使用了硬编码的方式硬性规定了自定义资源文件必须为 messageSource，否则会获取不到自定义资源配置，这也是为什么 bean 的 id 如果不为 messageSource会抛出异常。

```java
	protected void initMessageSource() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(MESSAGE_SOURCE_BEAN_NAME)) {
			// 如果在配置中已经配置了 messageSource，那么将 messageSource 提取并记录在 this.messageSource 中
			this.messageSource = beanFactory.getBean(MESSAGE_SOURCE_BEAN_NAME, MessageSource.class);
			// Make MessageSource aware of parent MessageSource.
			if (this.parent != null && this.messageSource instanceof HierarchicalMessageSource) {
				HierarchicalMessageSource hms = (HierarchicalMessageSource) this.messageSource;
				if (hms.getParentMessageSource() == null) {
					// Only set parent context as parent MessageSource if no parent MessageSource
					// registered already.
					hms.setParentMessageSource(getInternalParentMessageSource());
				}
			}
			if (logger.isDebugEnabled()) {
				logger.debug("Using MessageSource [" + this.messageSource + "]");
			}
		}
		else {
			// Use empty MessageSource to be able to accept getMessage calls.
			// 如果用户并没有定义配置文件，那么使用临时的 DelegatingMessageSource 以便于作为调用 getMessage 方法的返回。
			DelegatingMessageSource dms = new DelegatingMessageSource();
			dms.setParentMessageSource(getInternalParentMessageSource());
			this.messageSource = dms;
			beanFactory.registerSingleton(MESSAGE_SOURCE_BEAN_NAME, this.messageSource);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate MessageSource with name '" + MESSAGE_SOURCE_BEAN_NAME +
						"': using default [" + this.messageSource + "]");
			}
		}
	}
```

- 通过读取并将自定义资源文件配置记录在容器中，那么就可以在获取资源文件的时候直接使用，如：在 AbstractApplicationContext 中的获取资源文件属性的方法：

```java
	// 其中 getMessageSource() 方法正是获取了之前定义的自定义资源配置。
	public String getMessage(String code, @Nullable Object[] args, @Nullable String defaultMessage, Locale locale) {
		return getMessageSource().getMessage(code, args, defaultMessage, locale);
	}
```



##### 5.6.4 初始化 ApplicationEventMulticaster

- Spring 的事件监听用法：

（1）定义监听事件。

```java
public class TestEvent extends ApplicationEvent {
    public String msg;
    
    public TestEvent(Object source) {
        super(source);
    }
    publci TestEvent(Object source, String msg) {
        super(source);
        this.msg = msg;
    }
    public void print() {
        msg.sout
    }
}
```

（2）定义监听器。

```java
public class TestListener implements ApplicationListener {
    public void onApplicationEvent(ApplicationEvent event) {
        if (event instanceof TestEvent) {
            TestEvent testEvent = (TestEvent) event;
            testEvent.print();
        }
    }
}
```

（3）添加配置文件。

```xml
<bean id="tesstListener" class="com.test.event.TestListener" />
```

（4）测试

```java
public class Test {
    public static void main(String[] args) {
        ApplicationContext context = new ClassPathXmlApplicationContext("classpath:applicationContext.xml");
        TestEvent event = new TestEvent("hello", "msg");
        context.publishEvent(event);
    }
}
```

（5）结果：当程序运行时，Spring 会将发出的 TestEvent 事件转给自定义的 TestListener 进行进一步的处理。

- 上述即是观察者模式。ApplicationEventMulticaster 的初始化方法是 initApplicationEventMulticaster 方法。initApplicationEventMulticaster  的方式只考虑两种情况。
  - 如果用户自定义了事件广播器，那么使用用户自定义的事件广播器。
  - 如果用户没有自定义事件广播器，那么使用默认的 ApplicationEventMulticaster。

```java
	protected void initApplicationEventMulticaster() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME)) {
			this.applicationEventMulticaster =
					beanFactory.getBean(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, ApplicationEventMulticaster.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using ApplicationEventMulticaster [" + this.applicationEventMulticaster + "]");
			}
		}
		else {
			this.applicationEventMulticaster = new SimpleApplicationEventMulticaster(beanFactory);
			beanFactory.registerSingleton(APPLICATION_EVENT_MULTICASTER_BEAN_NAME, this.applicationEventMulticaster);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate ApplicationEventMulticaster with name '" +
						APPLICATION_EVENT_MULTICASTER_BEAN_NAME +
						"': using default [" + this.applicationEventMulticaster + "]");
			}
		}
	}
```

 SimpleApplicationEventMulticaster.java

```java
	public void multicastEvent(final ApplicationEvent event, @Nullable ResolvableType eventType) {
		ResolvableType type = (eventType != null ? eventType : resolveDefaultEventType(event));
		for (final ApplicationListener<?> listener : getApplicationListeners(event, type)) {
			Executor executor = getTaskExecutor();
			if (executor != null) {
				executor.execute(() -> invokeListener(listener, event));
			}
			else {
				invokeListener(listener, event);
			}
		}
	}
```

- 当产生 Spring 事件的时候会默认使用 SimpleApplicationEventMulticaster 的 multicastEvent 来广播事件，遍历所有监听器，并使用监听器中的 invokeListener(listener, event) 方法来进行监听器的处理。对于每个监听器来说都可以获取到产生的事件，但是是否进行处理则由事件监听器来决定。



##### 5.6.5 注册监听器

- Spring 注册监听器时做了以下的逻辑操作：

```java
	protected void registerListeners() {
		// Register statically specified listeners first.
		// 硬编码方式注册的监听器处理
		for (ApplicationListener<?> listener : getApplicationListeners()) {
			getApplicationEventMulticaster().addApplicationListener(listener);
		}

		// Do not initialize FactoryBeans here: We need to leave all regular beans
		// uninitialized to let post-processors apply to them!
		// 配置文件注册的监听器处理
		String[] listenerBeanNames = getBeanNamesForType(ApplicationListener.class, true, false);
		for (String listenerBeanName : listenerBeanNames) {
			getApplicationEventMulticaster().addApplicationListenerBean(listenerBeanName);
		}

		// Publish early application events now that we finally have a multicaster...
		Set<ApplicationEvent> earlyEventsToProcess = this.earlyApplicationEvents;
		this.earlyApplicationEvents = null;
		if (earlyEventsToProcess != null) {
			for (ApplicationEvent earlyEvent : earlyEventsToProcess) {
				getApplicationEventMulticaster().multicastEvent(earlyEvent);
			}
		}
	}
```





#### 5.7 初始化非延迟加载单例

- 完成 BeanFactory 的初始化工作，其中包括 ConversionService 的设置、配置冻结以及非延迟加载的 bean 的初始化工作。

```java
	protected void finishBeanFactoryInitialization(ConfigurableListableBeanFactory beanFactory) {
		// Initialize conversion service for this context.
		if (beanFactory.containsBean(CONVERSION_SERVICE_BEAN_NAME) &&
				beanFactory.isTypeMatch(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class)) {
			beanFactory.setConversionService(
					beanFactory.getBean(CONVERSION_SERVICE_BEAN_NAME, ConversionService.class));
		}

		// Register a default embedded value resolver if no bean post-processor
		// (such as a PropertyPlaceholderConfigurer bean) registered any before:
		// at this point, primarily for resolution in annotation attribute values.
		if (!beanFactory.hasEmbeddedValueResolver()) {
			beanFactory.addEmbeddedValueResolver(strVal -> getEnvironment().resolvePlaceholders(strVal));
		}

		// Initialize LoadTimeWeaverAware beans early to allow for registering their transformers early.
		String[] weaverAwareNames = beanFactory.getBeanNamesForType(LoadTimeWeaverAware.class, false, false);
		for (String weaverAwareName : weaverAwareNames) {
			getBean(weaverAwareName);
		}

		// Stop using the temporary ClassLoader for type matching.
		beanFactory.setTempClassLoader(null);

		// Allow for caching all bean definition metadata, not expecting further changes.
		// 冻结所有的 bean 定义，说明注册的 bean 定义将不被修改或任何进一步的处理。
		beanFactory.freezeConfiguration();

		// Instantiate all remaining (non-lazy-init) singletons.
		// 实例化剩下的单例实例（非惰性的）
		beanFactory.preInstantiateSingletons();
	}
```



##### 1.ConversionService 的设置

- Spring 提供了类似从 String 转换为 Date 的方式：使用 Converter。

（1）定义转换器。

```java
public class String2DateConverter implements Converter<String, Date> {
    @Override
    public Date convert(String arg0) {
        try {
            return DateUtils.parseDate(arg0, new String[] {"yyyy-MM-dd HH:mm:ss"});
        } catch(ParseException e) {
            return null;
        }
    }
}
```

（2）注册。

```xml
<bean id="conversionService" calss="org.Springframework.context.support.ConversionServiceFactoryBean">
	<property name="converters">
    	<list>
        	<bean class="String2DateConverter" />
        </list>
    </property>
</bean>
```

（3）测试。

```java
public void testStringToPhoneNumberConvert() {
    DefaultConversionService conversionService = new DefaultConversionService();
    conversionService.addConverter(new StrignToPhoneNumberConverter());
    
    String phoneNumberStr = "010-12345678";
    PhoneNumberModel phoneNumber = conversionService.convert(phoneNumberStr, PhoneNumberModel.class);
    
    Assert.assertEquals("010", phoneNumber.getAreaCode());
}
```

- ConversionService 的配置就是在 finishBeanFactoryInitialization 函数中被初始化的。



##### 2.冻结配置

- 冻结所有的 bean 定义，说明注册的 bean 定义将不被修改或进行任何进一步的处理。

```java
	public void freezeConfiguration() {
		this.configurationFrozen = true;
		this.frozenBeanDefinitionNames = StringUtils.toStringArray(this.beanDefinitionNames);
	}
```



##### 3.初始化非延迟加载

- **ApplicationContext 实现的默认行为就是在启动时将所有单例 bean 提前进行实例化。**提前实例化意味着作为初始化过程的一部分，ApplicationContext 实例会创建并配置所有的单例 bean。这样在配置中的任何错误就会即可被发现。这个实例化过程在 finishBeanFactoryInitialization 方法下的 beanFactory.preInstantiateSingletons(); 函数完成的。

```java
	public void preInstantiateSingletons() throws BeansException {
		if (logger.isDebugEnabled()) {
			logger.debug("Pre-instantiating singletons in " + this);
		}

		// Iterate over a copy to allow for init methods which in turn register new bean definitions.
		// While this may not be part of the regular factory bootstrap, it does otherwise work fine.
		List<String> beanNames = new ArrayList<>(this.beanDefinitionNames);

		// Trigger initialization of all non-lazy singleton beans...
		for (String beanName : beanNames) {
			RootBeanDefinition bd = getMergedLocalBeanDefinition(beanName);
			if (!bd.isAbstract() && bd.isSingleton() && !bd.isLazyInit()) {
				if (isFactoryBean(beanName)) {
					Object bean = getBean(FACTORY_BEAN_PREFIX + beanName);
					if (bean instanceof FactoryBean) {
						final FactoryBean<?> factory = (FactoryBean<?>) bean;
						boolean isEagerInit;
						if (System.getSecurityManager() != null && factory instanceof SmartFactoryBean) {
							isEagerInit = AccessController.doPrivileged((PrivilegedAction<Boolean>)
											((SmartFactoryBean<?>) factory)::isEagerInit,
									getAccessControlContext());
						}
						else {
							isEagerInit = (factory instanceof SmartFactoryBean &&
									((SmartFactoryBean<?>) factory).isEagerInit());
						}
						if (isEagerInit) {
							getBean(beanName);
						}
					}
				}
				else {
					getBean(beanName);
				}
			}
		}

		// Trigger post-initialization callback for all applicable beans...
		for (String beanName : beanNames) {
			Object singletonInstance = getSingleton(beanName);
			if (singletonInstance instanceof SmartInitializingSingleton) {
				final SmartInitializingSingleton smartSingleton = (SmartInitializingSingleton) singletonInstance;
				if (System.getSecurityManager() != null) {
					AccessController.doPrivileged((PrivilegedAction<Object>) () -> {
						smartSingleton.afterSingletonsInstantiated();
						return null;
					}, getAccessControlContext());
				}
				else {
					smartSingleton.afterSingletonsInstantiated();
				}
			}
		}
	}
```



#### 5.8 finishRefresh

- 在 Spring 中还提供了 Lifecycle 接口，Lifecycle 中包含 start/stop 方法，实现此接口后 Spring 会保证在启动的时候调用其 start 方法开始生命周期，并在 Spring 关闭的时候调用 stop 方法来结束生命周期，通常用来配置后台程序，在启动后一直运行（如对 MQ 进行轮询等）。ApplicationContext 的初始化最后的 finishRefresh(); 就是保证这一功能的实现。

```java
	protected void finishRefresh() {
		// Clear context-level resource caches (such as ASM metadata from scanning).
		clearResourceCaches();

		// Initialize lifecycle processor for this context.
		initLifecycleProcessor();

		// Propagate refresh to lifecycle processor first.
		getLifecycleProcessor().onRefresh();

		// Publish the final event.
		publishEvent(new ContextRefreshedEvent(this));

		// Participate in LiveBeansView MBean, if active.
		LiveBeansView.registerApplicationContext(this);
	}
```

##### 1.initLifecycleProcessor

- 当 ApplicationContext 启动或停止时，它会通过 LifecycleProcessor 来与所有声明的 bean 的周期做状态更新，而在 LifecycleProcessor 的使用前需要初始化。

```java
	protected void initLifecycleProcessor() {
		ConfigurableListableBeanFactory beanFactory = getBeanFactory();
		if (beanFactory.containsLocalBean(LIFECYCLE_PROCESSOR_BEAN_NAME)) {
			this.lifecycleProcessor =
					beanFactory.getBean(LIFECYCLE_PROCESSOR_BEAN_NAME, LifecycleProcessor.class);
			if (logger.isDebugEnabled()) {
				logger.debug("Using LifecycleProcessor [" + this.lifecycleProcessor + "]");
			}
		}
		else {
			DefaultLifecycleProcessor defaultProcessor = new DefaultLifecycleProcessor();
			defaultProcessor.setBeanFactory(beanFactory);
			this.lifecycleProcessor = defaultProcessor;
			beanFactory.registerSingleton(LIFECYCLE_PROCESSOR_BEAN_NAME, this.lifecycleProcessor);
			if (logger.isDebugEnabled()) {
				logger.debug("Unable to locate LifecycleProcessor with name '" +
						LIFECYCLE_PROCESSOR_BEAN_NAME +
						"': using default [" + this.lifecycleProcessor + "]");
			}
		}
	}
```



##### 2.onRefresh

- 启动所有实现了 Lifecycle 接口的 bean。

```java
	public void onRefresh() {
		startBeans(true);
		this.running = true;
	}

	private void startBeans(boolean autoStartupOnly) {
		Map<String, Lifecycle> lifecycleBeans = getLifecycleBeans();
		Map<Integer, LifecycleGroup> phases = new HashMap<>();
		lifecycleBeans.forEach((beanName, bean) -> {
			if (!autoStartupOnly || (bean instanceof SmartLifecycle && ((SmartLifecycle) bean).isAutoStartup())) {
				int phase = getPhase(bean);
				LifecycleGroup group = phases.get(phase);
				if (group == null) {
					group = new LifecycleGroup(phase, this.timeoutPerShutdownPhase, lifecycleBeans, autoStartupOnly);
					phases.put(phase, group);
				}
				group.add(beanName, bean);
			}
		});
		if (!phases.isEmpty()) {
			List<Integer> keys = new ArrayList<>(phases.keySet());
			Collections.sort(keys);
			for (Integer key : keys) {
				phases.get(key).start();
			}
		}
	}
```



##### 3.publishEvent

- 当完成 ApplicationContext 初始化的时候，要通过 Spring 中的事件发布机制来发出 ContextRefreshedEvent 事件，以保证对应的监听器可以做进一步的逻辑处理。

```java
	public void publishEvent(ApplicationEvent event) {
		publishEvent(event, null);
	}

	protected void publishEvent(Object event, @Nullable ResolvableType eventType) {
		Assert.notNull(event, "Event must not be null");
		if (logger.isTraceEnabled()) {
			logger.trace("Publishing event in " + getDisplayName() + ": " + event);
		}

		// Decorate event as an ApplicationEvent if necessary
		ApplicationEvent applicationEvent;
		if (event instanceof ApplicationEvent) {
			applicationEvent = (ApplicationEvent) event;
		}
		else {
			applicationEvent = new PayloadApplicationEvent<>(this, event);
			if (eventType == null) {
				eventType = ((PayloadApplicationEvent<?>) applicationEvent).getResolvableType();
			}
		}

		// Multicast right now if possible - or lazily once the multicaster is initialized
		if (this.earlyApplicationEvents != null) {
			this.earlyApplicationEvents.add(applicationEvent);
		}
		else {
			getApplicationEventMulticaster().multicastEvent(applicationEvent, eventType);
		}

		// Publish event via parent context as well...
		if (this.parent != null) {
			if (this.parent instanceof AbstractApplicationContext) {
				((AbstractApplicationContext) this.parent).publishEvent(event, eventType);
			}
			else {
				this.parent.publishEvent(event);
			}
		}
	}
```

