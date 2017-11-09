####Spring框架主要包含的模块
#####Core Container: Core、Beans、Context、Expression Language(Core、Beans是框架的基础 提供了IoC和依赖注入 基础概念就是BeanFactory)
#####Data Access:包含JDBC、ORM、OXM、JMS、Transaction
#####Web模块
#####AOP模块
#####Test：支持JUnit、TestNG
####源代码载入Eclipse：
#####通过到GitHub下载源码，到指定文件夹下 执行命令 导入Eclipse即可
    gradle cleanidea eclipse
#####存在的问题:在spring-core包中缺少spring-cglib-repack-3.2.0.jar 及 spring-objenesis-repack-2.2.jar 所以需要从spring-core中重新解压该包并放入到repack中
####源码分析：
#####常见的加载方式包括BeanFactory AppalicationContext两种方式（前者是基础、后者是扩展）看一个基本的例子如下:
    BeanFactory bf = new XmlBeanFactory(new ClassPathResource(path));
    TestBean bean = (TestBean)bf.getBean(beanName);
####上述代码实现的功能的过程如下：
#####1.根据路径读取配置文件
#####2.根据配置文件的内容找到累的配置、进行实例化操作
#####3.调用实例化之后的实例
#####分析之前，先看一下Spring框架中最重要的两个类DefaultListableBeanFactory XmlBeanDefinitionReader
#####前者是Spring注册和加载bean的默认实现 子类XmlBeanFactory使用了自定义的XML读取器XmlBeanDefinitionReader
####整个XML配置文件读取的流程：
#####1.使用ResourceLoader将资源文件的路径转换为对应的Resource文件
#####2.通过DocumentLoader对Resource文件进行转换 将Resource文件转换为Document
#####3.解析Document，并利用BeanDefinitionParserDelegate对Element进行解析
####对于Spring而言，对所有的资源进行了抽象定义了借口Resource，抽象提供了内部使用到的底层资源 File、URL、Classpath，提供了判断资源状态的方法，提供了不同资源的转换，获取相关属性，当然想获得InputStream 可以通过getInputStream()获得
#####ClassPathResource底层实现就是通过this.class.getResourceAsStream(this.path)获取
#####FileSystemResource则直接用new FileInputStream(this.file)实现
####资源的加载：
#####通过XmlBeanFactory的XmlBeanDefinitionReader读取 调用了
    this.reader.loadBeanDefinitions(resource)
#####加载之前调用了父类的ignoreDEpendencyInterface()，用户忽略给定接口的自动装备功能
####Bean的加载:
#####1.通过EncodedResource对resource封装
#####2.获得输入流
#####3.调用加载函数
    doLoadBeanDefinitions(inputSource, resource);
####接下来做的是三件事：
#####1.获取XML文件的验证模式
#####2.加载XML文件，得到对应的Document
#####3.根据返回的Document注册Bean信息
####XML的验证模式包括两种 DTD(文档类型定义) XSD(XML Schemas Definition)
#####验证的过程比较容易，就是检验 如果设定了验证模式就使用指定的验证模式，否则使用默认的模式
#####Spring检验验证模式的办法就是判断是否包含DOCTYPE，如果包含是DTD否则是XSD
####获取Document
#####首先通过创建DocumentBuilderFactory，创建DocumentBuilder，通过解析inputSource返回Document
#####在这个过程中涉及到了EntityResolver参数，这个参数的作用在于，很多时候读取XML文档寻找DTD定义，但是通过网络来下载，容易出现网络中断或者不可用的情况，
#####EntityResolver的作用是项目本身就提供一个寻找DTD生命的方法，我们可以讲DTD文件放到项目的某处，在实现的时候直接读取该文档并返回给SAX就可以，避免通过了网络寻找DTD文件
####解析及注册BeanDefinitions
#####首先设置环境变量，获得Document的root之后加载及注册Bean
#####加载注册的过程中，首先解析Profile属性(该属性的作用通常是在配置文件中进行配置配合在beans里，对测试和正式环境分开部署，方便进行切换开发部署环境)
#####解析之前和之后Spring留了两个空白的方法用于子类继承改写，如果需要在解析前后进行处理的话，只需要处理这两个方法
####对于bean标签的解析和注册
#####首先从元素解析及信息提取开始，获取BeanDefinitionHolder对象
#####上面对象的获取利用了delegate的parseBeanEdinitionElement(ele)方法，解析出id, name属性
#####进一步解析其他属性封装到GenericBeanDefinition，指定BeanName，将获取到的信息封装到BeanDefinitionHolder中
#####解析属性包括class属性 各种属性 元数据 lookup-method replaced-method 构造函数参数 property子元素 qualifier子元素
#####lookup-method replaced-method都被加载到methodOverrides
#####解析子元素constructor-arg（page61 待写）
####注册BeanDefinition把beanName作为key 进行注册
#####1.首先检验methodOverrides
#####2.对已注册的情况的处理
#####3.加入map缓存
#####4.清理之前留下的对应beanName的缓存
####对于bean的加载(解析、注册、加载):
#####1.把传入的参数转化为beanName(由于传入的可能是别名或者FactoryBean)
#####2.尝试从缓存中加载单例(需要解决循环依赖的问题)
#####3.bean的实例化
#####4.依赖检查
#####5.检查parentBeanFactory
#####6.把GernericBeanDefinition转化成RootBeanDefinition
#####7.寻找依赖
#####8.针对不同的scope进行bean的创建
#####9.类型转换(参数的类型转换)
#####FactoryBean用于实例化的过程中，范型方法包括getObject() isSingleton() getObjectType()
#####也就是说 当<bean>的class配置的实现类是FactoryBean时，通过getBean()获取的是FactoryBean的getObject()方法返回的对象
####尝试从缓存中获取单例bean
#####过程是首先尝试从缓存中加载，然后再次尝试从singletonFactories中加载，而为了防止循环依赖的问题，不等bean创建完成就将创建
#####bean的ObjectFactory曝光到缓存中
#####在getBean的过程中 getObjectForBeanInstance()方法检查正确性，如果该bean 是FactoryBean，则调用对应的getObject()方法
####从缓存中获取单例的过程：
#####1.首先检查缓存是否已经加载过
#####2.如果没有加载过，纪录beanName的正在加载状态
#####3.加载单例前记录加载状态(通过beforeSingletonCreation()方法记录加载状态，把正要创建的bean纪录在缓存中，对循环依赖进行检测)
#####4.通过ObjectFactory的Object方法实例化bean
#####5.加载单例后的处理方法调用，移除缓存中对该bean的正在加载状态的纪录
#####6.将结果纪录至缓存并删除加载bean过程中所纪录的各种辅助状态
#####7.返回处理结果
####创建bean:
#####1.根据class属性/className解析Class
#####2.对override属性进行标记及验证
#####3.应用后处理器
#####4.创建bean
#####对override属性处理时，就是检测如果存在methodOverrides属性，会动态的为当前bean生成代理并使用对应的拦截器为bean做增强处理
#####实例化前的后处理器应用：将AbsractBeanDefinition转换为BeanWrapper前的处理，给子类机会修改BeanDefinition,经过该方法，bean可能是经过处理的代理Bean，可能是通过cglib生成的，也可能是通过其他技术生成的，进行AOP操作
####循环依赖
#####1.构造器循环依赖，无法解决，只能抛出异常
#####2.setter循环依赖(对单例作用域)，通过暴露单例工厂方法ObjectFactory的方式解决
#####3.对于prototype范围的依赖处理 无法完成依赖注入 无法提前暴露创建中的bean
####创建bean:
#####如果创建了代理或者重写了后处理器 改变了bean 则直接返回就可以了，否则进行常规bean的创建
#####1.如果是单例则需要清理缓存
#####2.实例化bean 将BeanDefinition转换为BeanWrapper
#####2.1如果存在工厂方法则使用工厂方法进行初始化
#####2.2如果存在多个构造函数，根据参数锁定构造函数并进行初始化
#####2.3使用默认的构造函数进行bean的实例化
#####3.依赖处理
#####4.属性填充
#####5.循环依赖检查
#####6.注册disposableBean
#####7.完成创建并返回
####创建bean的实例：
#####1.如果RootBeanDefinition存在factoryMethodName属性，或者配置了factory-method,利用工厂方法模式创建bean的实例
#####2.解析构造函数并进行构造函数的实例化，为了提高效率，使用了缓存机制
####autowireConstructor:
#####1.对构造函数参数的确定，如果传入的参数不为空，直接确定参数
#####2.如果之前分析过，参数则已经记录在缓存中，便可以直接拿来使用，此处需要用类型转换器保证参数类型
#####3.配置文件获取，如果前两者都不能获取，则从配置文件里获取，通过从BeanDefinition实例中获取，通过解析方法，返回可以解析到的参数的个数，按照public构造函数有限参数数量降序、非public构造函数参数数量降序，之后转换参数类型，进行验证，实例化Bean即可
####实例化的策略
#####首先判断methodOver中是否存在值，如果不存在，直接使用反射的方式实例化就可以了，如果存在，则使用动态代理的方式将对应逻辑的拦截增强器设置进去，保证调用方法的时候会被拦截器进行增强
####属性填充：
#####1.根据注入类型(byName/byType)提取依赖的bean 统一存入PropertyValues
#####应用后处理PropertyValues方法，对属性再次处理，将所有的属性填充到BeanWrapper中
    
