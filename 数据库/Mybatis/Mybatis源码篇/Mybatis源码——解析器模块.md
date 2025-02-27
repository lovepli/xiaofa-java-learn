## 解析器模块的功能
- 一个功能，是对 XPath 进行封装，为 MyBatis 初始化时解析 mybatis-config.xml 配置文件以及映射配置文件提供支持。  
- 另一个功能，是为处理动态 SQL 语句中的占位符提供支持。

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/mybatis/%E8%A7%A3%E6%9E%90%E5%99%A8%E8%B7%AF%E5%BE%84%E6%88%AA%E5%9B%BE.png)


**先仔细研读下所有类的源码实现。结合源码提供的测试类一步步debug测试。**

## XPathParser ##

具体源码如下：
[https://github.com/zhaoxiaofa/xiaofa-mybatis/blob/master/src/main/java/org/apache/ibatis/parsing/XPathParser.java](https://github.com/zhaoxiaofa/xiaofa-mybatis/blob/master/src/main/java/org/apache/ibatis/parsing/XPathParser.java "XPathParser源码详解")

结合XPathParserTest测试类一步步debug。  

    @Test
    void shouldTestXPathParserMethods() throws Exception {
	    String resource = "resources/nodelet_test.xml";
	    try (InputStream inputStream = Resources.getResourceAsStream(resource)) {
	      XPathParser parser = new XPathParser(inputStream, false, null, null);
	      assertEquals((Long) 1970L, parser.evalLong("/employee/birth_date/year"));
	      assertEquals((short) 6, (short) parser.evalShort("/employee/birth_date/month"));
	      assertEquals((Integer) 15, parser.evalInteger("/employee/birth_date/day"));
	      assertEquals((Float) 5.8f, parser.evalFloat("/employee/height"));
	      assertEquals((Double) 5.8d, parser.evalDouble("/employee/height"));
	      assertEquals("${id_var}", parser.evalString("/employee/@id"));
	      assertEquals(Boolean.TRUE, parser.evalBoolean("/employee/active"));
	      assertEquals("<id>${id_var}</id>", parser.evalNode("/employee/@id").toString().trim());
	      assertEquals(7, parser.evalNodes("/employee/*").size());
	      XNode node = parser.evalNode("/employee/height");
	      assertEquals("employee/height", node.getPath());
	      assertEquals("employee[${id_var}]_height", node.getValueBasedIdentifier());
	    }
  	}



定义资源路径，通过io模块的Resources类读入资源，得到一个InputStream，具体的实现，在io模块讲。其实最底层还是调用的ClassLoader类的getResourceAsStream方法。这里只需要知道，通过xml文件路径，读入一个输入流就行了。  

读入流之后，才进入XPathParser的构造方法。

    public XPathParser(InputStream inputStream, boolean validation, Properties variables, EntityResolver entityResolver) {
	    commonConstructor(validation, variables, entityResolver);
	    this.document = createDocument(new InputSource(inputStream));
    }

进入该方法时，inputStream为读入的xml的流，其余各参数均为默认值。接着进入一个公用的commonConstructor方法。

    private void commonConstructor(boolean validation, Properties variables, EntityResolver entityResolver) {
	    this.validation = validation;
	    this.entityResolver = entityResolver;
	    this.variables = variables;
	    // 构建xPath工厂,用工厂创建xPath
	    XPathFactory factory = XPathFactory.newInstance();
	    // 创建的xPath对象很多属性值都为null
	    this.xpath = factory.newXPath();
    }

commonConstructor方法其实才是最基础的构造方法，他给XPathParser类各属性初始化值。唯一值得注意的是xpath属性，使用工厂模式创建了一个XPath类。具体细节不赘述，不是重点。最终得到的xpath其中的属性基本也都是默认值（大多为null）。  

接着往下看，new InputSource(inputStream)将inputStream转化为InputSource。其实本质是把入参inputStream赋值给了InputSource类的byteStream属性（该属性的类型是InputStream）。  

createDocument方法是将InputSource转化为Document。大致流程是创建工厂，用工厂模式创建builder,用builder最终解析InputSource返回Document。最重要的一个方法是builder.parse(inputSource)。这个方法最底层是调用的DOMParser，这个具体的实现不是当前重点。总之，最终成功得到一个Document。  

createDocument源码和得到的Document截图如下：

    private Document createDocument(InputSource inputSource) {
    // important: this must only be called AFTER common constructor
    try {
      // 创建DocumentBuilder工厂
      DocumentBuilderFactory factory = DocumentBuilderFactory.newInstance();
      factory.setValidating(validation);

      factory.setNamespaceAware(false);
      factory.setIgnoringComments(true);
      factory.setIgnoringElementContentWhitespace(false);
      factory.setCoalescing(false);
      factory.setExpandEntityReferences(true);
      // 创建DocumentBuilder
      DocumentBuilder builder = factory.newDocumentBuilder();
      builder.setEntityResolver(entityResolver);
      builder.setErrorHandler(new ErrorHandler() {
        @Override
        public void error(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void fatalError(SAXParseException exception) throws SAXException {
          throw exception;
        }

        @Override
        public void warning(SAXParseException exception) throws SAXException {
        }
      });
      // 通过DocumentBuilder解析xml文件,返回Document对象
      return builder.parse(inputSource);
    } catch (Exception e) {
      throw new BuilderException("Error creating document instance.  Cause: " + e, e);
    }

上面是解析的xml文件部分截图，下面是解析之后的document对象。

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/mybatis/xpathParser.png)

此时，已经得到了一个 XPathParser 对象，对象中的 document 属性就是解析 xml 之后的结果。下面进入单元测试的结果校验方法。取第一个校验

    assertEquals((Long) 1970L, parser.evalLong("/employee/birth_date/year"));

进入evalLong方法，一步步debug，会发现，最终都是调用的 evaluate 方法。所以这里着重讲一下 evaluate 方法。源码如下：  

	private Object evaluate(String expression, Object root, QName returnType) {
	    try {
	      // 调用xPath的解析方法
	      return xpath.evaluate(expression, root, returnType);
	    } catch (Exception e) {
	      throw new BuilderException("Error evaluating XPath.  Cause: " + e, e);
	    }
    }

expression 就是想要核对的节点，在本例中是"/employee/birth_date/year";  
root 就是第一步获取的 document;  
returnType 定义了返回类型;  

底层调用的是 XPath 类的 evaluate 方法。具体实现暂时不看，总之就是能够根据节点路径在document中找到对应的值。  

其他 eval 前缀的方法大概类似，不再赘述。  


**总结一下XPathParser：**   
就是用来解析xml的。其实看着方法很多，其实只有各种构造方法XPathParser，每个构造方法都调用的commonConstructor方法，基础的evaluate方法，基于该方法封装的各种形式的eval前缀开头的转化方法，还有createDocument方法（把文件解析为Document）。这样整合起来，结构就清晰很多了。


## PropertyParser ##
是动态属性解析器。




## GenericTokenParser ##
这个类其实很简单，三个属性，一个构造方法，一个解析方法。  
三个属性：  

      private final String openToken; // 解析开始标识
	  private final String closeToken; // 解析结束标识
	  private final TokenHandler handler; 

TokenHandler 是一个接口，在实际解析过程中使用的是 VariableTokenHandler 类（这个类是PropertyParser 类的内部类），类中的属性 variables，用来接收配置文件。 最重要的是 parse 方法，我个人是调过了内部的实现细节。 总之，能够替换掉配置文件中使用$占位符的值。

在使用源码中的测试类 PropertyParserTest 的 replaceToVariableValue 方法进行测试时的截图如下：

![](https://raw.githubusercontent.com/zhaoxiaofa/xiaofa-java-learn/master/pictures/mybatis/GenericTokenParser.parse.png)

也可以用 GenericTokenParserTest 类中的测试方法进行测试，效果是一样的。  


