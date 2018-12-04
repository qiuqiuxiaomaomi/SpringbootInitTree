# SpringbootInitTree
Springboot启动分析


<pre>
    public SpringApplication(Object... sources) {
      initialize(sources); // sources目前是一个MyApplication的class对象
    }

    private void initialize(Object[] sources) {
      if (sources != null && sources.length > 0) {
        this.sources.addAll(Arrays.asList(sources)); // 把sources设置到SpringApplication的sources属性中，目前只是一个MyApplication类对象
      }
      this.webEnvironment = deduceWebEnvironment(); // 判断是否是web程序(javax.servlet.Servlet和org.springframework.web.context.ConfigurableWebApplicationContext都必须在类加载器中存在)，并设置到webEnvironment属性中
      // 从spring.factories文件中找出key为ApplicationContextInitializer的类并实例化后设置到SpringApplication的initializers属性中。这个过程也就是找出所有的应用程序初始化器
      setInitializers((Collection) getSpringFactoriesInstances(
          ApplicationContextInitializer.class));
      // 从spring.factories文件中找出key为ApplicationListener的类并实例化后设置到SpringApplication的listeners属性中。这个过程就是找出所有的应用程序事件监听器
      setListeners((Collection) getSpringFactoriesInstances(ApplicationListener.class));
      // 找出main类，这里是MyApplication类
      this.mainApplicationClass = deduceMainApplicationClass();
    }

</pre>

<pre>
SpringApplication的run方法

     public ConfigurableApplicationContext run(String... args) {
      StopWatch stopWatch = new StopWatch(); // 构造一个任务执行观察器
      stopWatch.start(); // 开始执行，记录开始时间
      ConfigurableApplicationContext context = null;
      configureHeadlessProperty();
      // 获取SpringApplicationRunListeners，内部只有一个EventPublishingRunListener
      SpringApplicationRunListeners listeners = getRunListeners(args);
      // 上面分析过，会封装成SpringApplicationEvent事件然后广播出去给SpringApplication中的listeners所监听
      // 这里接受ApplicationStartedEvent事件的listener会执行相应的操作
      listeners.started();
      try {
        // 构造一个应用程序参数持有类
        ApplicationArguments applicationArguments = new DefaultApplicationArguments(
            args);
        // 创建Spring容器
        context = createAndRefreshContext(listeners, applicationArguments);
        // 容器创建完成之后执行额外一些操作
        afterRefresh(context, applicationArguments);
        // 广播出ApplicationReadyEvent事件给相应的监听器执行
        listeners.finished(context, null);
        stopWatch.stop(); // 执行结束，记录执行时间
        if (this.logStartupInfo) {
          new StartupInfoLogger(this.mainApplicationClass)
              .logStarted(getApplicationLog(), stopWatch);
        }
        return context; // 返回Spring容器
      }
      catch (Throwable ex) {
        handleRunFailure(context, listeners, ex); // 这个过程报错的话会执行一些异常操作、然后广播出ApplicationFailedEvent事件给相应的监听器执行
        throw new IllegalStateException(ex);
      }
    }

</pre>

<pre>
创建容器的 createAndRefreshContext

      private ConfigurableApplicationContext createAndRefreshContext(
        SpringApplicationRunListeners listeners,
        ApplicationArguments applicationArguments) {
      ConfigurableApplicationContext context; // 定义Spring容器
      // 创建应用程序的环境信息。如果是web程序，创建StandardServletEnvironment；否则，创建StandardEnvironment
      ConfigurableEnvironment environment = getOrCreateEnvironment();
      // 配置一些环境信息。比如profile，命令行参数
      configureEnvironment(environment, applicationArguments.getSourceArgs());
      // 广播出ApplicationEnvironmentPreparedEvent事件给相应的监听器执行
      listeners.environmentPrepared(environment);
      // 环境信息的校对
      if (isWebEnvironment(environment) && !this.webEnvironment) {
        environment = convertToStandardEnvironment(environment);
      }

      if (this.bannerMode != Banner.Mode.OFF) { // 是否在控制台上打印自定义的banner
        printBanner(environment);
      }

      // Create, load, refresh and run the ApplicationContext
      context = createApplicationContext(); // 创建Spring容器
      context.setEnvironment(environment); // 设置Spring容器的环境信息
      postProcessApplicationContext(context); // 回调方法，Spring容器创建之后做一些额外的事
      applyInitializers(context); // SpringApplication的的初始化器开始工作
      // 遍历调用SpringApplicationRunListener的contextPrepared方法。目前只是将这个事件广播器注册到Spring容器中
      listeners.contextPrepared(context);
      if (this.logStartupInfo) {
        logStartupInfo(context.getParent() == null);
        logStartupProfileInfo(context);
      }

      // 把应用程序参数持有类注册到Spring容器中，并且是一个单例
      context.getBeanFactory().registerSingleton("springApplicationArguments",
          applicationArguments);

      Set<Object> sources = getSources();
      Assert.notEmpty(sources, "Sources must not be empty");
      load(context, sources.toArray(new Object[sources.size()]));
      // 广播出ApplicationPreparedEvent事件给相应的监听器执行
      listeners.contextLoaded(context);

      // Spring容器的刷新
      refresh(context);
      if (this.registerShutdownHook) {
        try {
          context.registerShutdownHook();
        }
        catch (AccessControlException ex) {
          // Not allowed in some environments.
        }
      }
      return context;
    }
</pre>

<pre>
Spring容器的创建createApplicationContext

      protected ConfigurableApplicationContext createApplicationContext() {
      Class<?> contextClass = this.applicationContextClass;
      if (contextClass == null) {
        try {
          // 如果是web程序，那么构造org.springframework.boot.context.embedded.AnnotationConfigEmbeddedWebApplicationContext容器
          // 否则构造org.springframework.context.annotation.AnnotationConfigApplicationContext容器
          contextClass = Class.forName(this.webEnvironment
              ? DEFAULT_WEB_CONTEXT_CLASS : DEFAULT_CONTEXT_CLASS);
        }
        catch (ClassNotFoundException ex) {
          throw new IllegalStateException(
              "Unable create a default ApplicationContext, "
                  + "please specify an ApplicationContextClass",
              ex);
        }
      }
      return (ConfigurableApplicationContext) BeanUtils.instantiate(contextClass);
    }
</pre>

<pre>
Spring容器创建之后有个回调方法postProcessApplicationContext

      protected void postProcessApplicationContext(ConfigurableApplicationContext context) {
        if (this.webEnvironment) { // 如果是web程序
            if (context instanceof ConfigurableWebApplicationContext) { // 并且也是Spring Web容器
                ConfigurableWebApplicationContext configurableContext = (ConfigurableWebApplicationContext) context;
                if (this.beanNameGenerator != null) { // 如果SpringApplication设置了是实例命名生成器，注册到Spring容器中
                    configurableContext.getBeanFactory().registerSingleton(
                            AnnotationConfigUtils.CONFIGURATION_BEAN_NAME_GENERATOR,
                            this.beanNameGenerator);
                }
            }
        }
        if (this.resourceLoader != null) { // 如果SpringApplication设置了资源加载器，设置到Spring容器中
            if (context instanceof GenericApplicationContext) {
                ((GenericApplicationContext) context)
                        .setResourceLoader(this.resourceLoader);
            }
            if (context instanceof DefaultResourceLoader) {
                ((DefaultResourceLoader) context)
                        .setClassLoader(this.resourceLoader.getClassLoader());
            }
        }
    }
</pre>

<pre>
初始化器做的工作，比如ContextIdApplicationContextInitializer会设置应用程序
id；AutoConfigurationReportLoggingInitializer会给应用程序添加一个条件注解解析器报告等

      protected void applyInitializers(ConfigurableApplicationContext context) {
      // 遍历每个初始化器，对调用对应的initialize方法
      for (ApplicationContextInitializer initializer : getInitializers()) {
        Class<?> requiredType = GenericTypeResolver.resolveTypeArgument(
            initializer.getClass(), ApplicationContextInitializer.class);
        Assert.isInstanceOf(requiredType, context, "Unable to call initializer.");
        initializer.initialize(context);
      }
    }
</pre>

<pre>
Spring容器的刷新refresh方法内部会做很多很多的事情：比如BeanFactory的设置，
BeanFactoryPostProcessor接口的执行、BeanPostProcessor接口的执行、自动化配置类的解析、
条件注解的解析、国际化的初始化等等.

run方法中的Spring容器创建完成之后会调用afterRefresh方法

      protected void afterRefresh(ConfigurableApplicationContext context,
        ApplicationArguments args) {
      afterRefresh(context, args.getSourceArgs()); // 目前是个空实现
      callRunners(context, args); // 调用Spring容器中的ApplicationRunner和CommandLineRunner接口的实现类
    }

    private void callRunners(ApplicationContext context, ApplicationArguments args) {
        List<Object> runners = new ArrayList<Object>();
      // 找出Spring容器中ApplicationRunner接口的实现类
        runners.addAll(context.getBeansOfType(ApplicationRunner.class).values());
      // 找出Spring容器中CommandLineRunner接口的实现类
        runners.addAll(context.getBeansOfType(CommandLineRunner.class).values());
      // 对runners进行排序
        AnnotationAwareOrderComparator.sort(runners);
      // 遍历runners依次执行
        for (Object runner : new LinkedHashSet<Object>(runners)) {
            if (runner instanceof ApplicationRunner) { // 如果是ApplicationRunner，进行ApplicationRunner的run方法调用
                callRunner((ApplicationRunner) runner, args);
            }
            if (runner instanceof CommandLineRunner) { // 如果是CommandLineRunner，进行CommandLineRunner的run方法调用
                callRunner((CommandLineRunner) runner, args);
            }
        }
    }
</pre>

<pre>
如果想在Spring容器启动的时候做一些特殊的事情：
      1）CommandLineRunner接口
      2）ApplicationRunner接口

      如果有顺序控制，可以添加  @Order 注解
</pre>

![](https://i.imgur.com/8Aj1DJp.png)

<pre>
把整个项目的流程比作一条河，那么监听器的作用就是能够听到河流里的所有声音，过滤器就是能够过
滤出其中的鱼，而拦截器则是拦截其中的部分鱼，并且作标记。所以当需要监听到项目中的一些信息，
并且不需要对流程做更改时，用监听器；当需要过滤掉其中的部分信息，只留一部分时，就用过滤器；
当需要对其流程进行更改，做相关的记录时用拦截器。


拦截器：
      HandlerInterceptorAdapter

过滤器：
      Filter
          doFilter方法
      1）在HttpServletRequest 到达 Servlet 之前，拦截客户的 HttpServletRequest 。 
         根据需要检查 HttpServletRequest ，也可以修改HttpServletRequest 头和数据。
      2）在HttpServletResponse 到达客户端之前，拦截HttpServletResponse 。 根据需要检
         查 HttpServletResponse ，也可以修改HttpServletResponse头和数据。  

监听器 Listener：
      1）监听器Listener就是在application,session,request三个对象创建、销毁或者往其中
         添加修改删除属性时自动执行代码的功能组件。

　　  2）Listener是Servlet的监听器，可以监听客户端的请求，服务端的操作等
</pre>