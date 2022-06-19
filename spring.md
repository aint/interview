## Spring Core


### Init of Container

1. **ApplicationContext** creation
	- parsing configurations and creating bean definitons
	- **ClassPathXmlApplicationContex, AnnotationConfigApplicationContext, AnnotationConfigWebApplicationContextt**
2. Modification bean definitions with **BeanFactoryPostProcessor**
	- **PropertyPlaceholderConfigurer** resolves placeholders against local properties and/or system properties and environment variables
	- **PropertySourcesPlaceholderConfigurer** resolves placeholders within bean definition property values and `@Value`
3. Beans creation
	- using **BeanFactory** by default
	- using a custom **FactoryBean** (only for xml)
4. Beans configuration with **BeanPostProcessor** that triggered before and after a bean initialization
	- `postProcessBeforeInitialization()` applies before any bean initialization
	- `postProcessAfterInitialization()` applies after any bean initialization
5. Beans are ready now


#### BeanDefinition
- class
- name
- scope
- constructor arguments
- properties
- autowiring mode
- lazy-initialization mode
- initialization method
- destruction method


#### Bean Initialization/Destruction Callbacks
- Init
	- `@PostConstruct`
	- interface `InitializingBean.afterPropertiesSet()`
	- `<bean id="..." class="..." init-method="init"/>`
	- `<beans default-init-method="init">`
- Destroy
	- `@PreDestroy`
	- interface `DisposableBean.destroy()`
	- `<bean id="..." class="..." destroy-method="cleanup"/>`
	- `<beans default-destroy-method="cleanup">`


#### FactoryBean
Interface used as a factory for an bean creation. For example we have `java.awt.Color` with RGB parameters and want have random params on every bean creation. To achieve this we should introduce **FactoryBean<Color>** and implement creation logic in a `getObject()` method. There are also `isSingleton()` and `getObjectType()` methods. This all have a point only for **XML**, because with Java Config we can easely implement any bean creation logic.


### Dependency Injection types
- **constructor dependency injection**
	- used for mandatory dependencies
	- enables implement components as immutable objects and to ensure that required dependencies are not **null**
	- make unpossible circular dependencies
- **setter dependency injection**
	- used for optional dependencies
	- dependencies can be reinjected
- **field dependency injection**
	- can lead to too many dependencies
	- unclear purpose of dependencies (optional or not)
	- testing requires reflection or something like `@InjectMocks` to inject dependencies
	- dependencies cann't be final




#### Autowiring modes
- **no** (Default) No autowiring.
- **byType**
- **byName** Autowiring by property name.
- **constructor** Analogous to byType, but applies to constructor arguments. If there is not exactly one bean of the constructor argument type in the container, a fatal error is raised.

`@Autowired, @Inject` using by type mode, `@Resource` by name.


#### Singelton

Singelton scope means that when you define a bean definition with this scope, then the Spring IoC container will create exactly one instance of the object defined by that **bean definition**.

**Injectiong  prototype bean into a singleton bean.**
As singleton beans initializes only once on the start of application context, thus their prototype dependencies initialized only once too.
To solve this issue, we have 2 approaches:
- Lookup Method injection
  ```xml
  <bean id="single" class="SingletonBean">
    <lookup-method name="getDependency" bean="proto"/>
  </bean>
  <bean id="proto" scope="prototype" class="PrototypeBean"/>
  ```
	Whenever we define a bean with lookup methods, Spring creates a proxy that delegates all the non-lookup methods to the original class. For the lookup methods, it overrides the implementation to something like this: `context.getBean(\"proto\")`. We have to define that method in the class and either provide a dummy implementation for it or make it abstract.

- Scoped Proxies
  ```xml
  <bean id="single" class="SingletonBean">
    <property name="dependency" ref="proto"/>
  </bean>
  <bean id="proto" scope="prototype" class="PrototypeBean">
     <aop:scoped-proxy/>
  </bean>
  ```
  Inthis case proxy is created for prototype bean. The proxy understands the scope and returns instances based on the requirements of the scope.

#### Misc

- Two proxy types:
	- default using CGLIB library which directly subclasses the object
	- Java Proxies, but the scoped bean must implement at least one interface
- Beans declared in `@Configuration`
	- configuration class proxied by **CGLIB**, so each call of bean definition methods will intercepted (to guarantee singelton for example)
	- according to **CGLIB** usage, configuration classes and bean-defining methods must not be *private* or *final*
	- due to CGLIB, whatever params you use while calling such methods, you will get the proper bean:
	```java
    @Bean
    public MyService myService() {
        return new MyService(myRepository(null));
    }
    @Bean
    public MyRepository myRepository(@Value("${val}") String val) {
        return new MyRepository(val);
    }
	```
- Beans declared in `@Component` (lite mode)
	- no proxies created, so method invocations are not intercepted and thus interpreted as a typical Java method invocation.
- `@Async` used in some cases you can return to the client immediately while a background job completes processing. For example sending an email, kicking off a database job, and others represent fire-and-forget scenarios
- `@Repository` beans used to catch persistence specific exceptions and rethrow them as one of Spring’s unified unchecked exception using **PersistenceExceptionTranslationPostProcessor**.


## Spring MVC

**Request -> DispatcherServlet -> HandlerMapping -> Pre-Interceptor -> Controller -> Post-Interceptor -> ViewResolver**

#### HttpMessageConverter

- `@ResponseBody` on a Controller method indicates to Spring that the return value of the method is **serialized directly to the body** of the HTTP Response. The **Accept** header specified by the client will be used to determine the respond's media type and the appropriate Converter to marshall the entity.
- `@RequestBody` is used on the argument of a Controller method – it indicates to Spring that **the body of the request is deserialized** to that particular Java entity. The **Content-Type** header will be used to determine the media type of the request body and converter for this.

![MessageConverter1.png](#file:792e1a7c-1bec-f6df-c19c-3d59b3428acf)

Types:
- ByteArrayHttpMessageConverter
- StringHttpMessageConverter
- **FormHttpMessageConverter** converts form data to/from a `MultiValueMap<String, String>`
- Jaxb2RootElementHttpMessageConverter – converts Java objects to/from XML (added only if JAXB2 is present on the classpath)
- MappingJacksonHttpMessageConverter – converts JSON (added only if Jackson is present on the classpath)


#### ContentNegotiatingViewResolver

Uses the requested media type to select a suitable *View* for a request.

- resolving using a path extension in the URL
- a URL parameter like this: http://test.com/accounts/list?format=xls. Using a parameter is disabled by default, but when enabled, it is checked second.
- finally the **Accept** header is checked

Once the requested media type has been determined, this resolver queries each delegate **view resolver** for a View and determines if the requested media type is compatible with the view's content type. The most compatible view is returned.

- ViewResolver
	- InternalResourceViewResolver
	- XmlViewResolver
	- JasperReportsViewResolver


#### Misc

- Custom Error Pages
    - Add /errors in web.xml and map contoller
    - Check error code from request attrubute `javax.servlet.error.status_code`
- implement `Converter<S, T>` and than
  ```java
  @GetMapping("/findbydate/{date}")
  findByDate(@PathVariable("date") LocalDateTime date) {
      ...;
  }
  ```
- **MethodArgumentNotValidException** exception to be thrown when validation on an argument annotated with `@Valid` fails. `@ExceptionHandler(MethodArgumentNotValidException.class)`


## Spring Boot

- `@ConditionalOnClass/@ConditionalOnMissingClass` activates a configuration only if a class are/aren't present on the classpath, `@ConditionalOnBean/@ConditionalOnMissingBean` enables a bean definition only if the bean was/wasn’t previously defined.
- `@SpringBootApplication` contains
	- `@SpringBootConfiguration`
    - `@EnableAutoConfiguration`
    	- `@AutoConfigurationImportSelector` uses `SpringFactoriesLoader#loadFactoryNames` who will look for ***AutoConfiguration** classes in a file with the path **META-INF/spring.factories**.
    - `@ComponentScan`


## Spring Security

- Spring Security uses an **AuthenticationEntryPoint** object to decide what to do when a user (anonymous) requires authentication. By default **LoginUrlAuthenticationEntryPoint** redirects all of unauthenticated users to the login page.
- **AccessDeniedHandler** used to handle an **AccessDeniedException** for all *authenticated* users.


The most fundamental object is **SecurityContextHolder**. This is where we store details of the present security context of the application, which includes details of the principal currently using the application. By default the **SecurityContextHolder** uses a **ThreadLocal** to store these details.

- **SecurityContextHolder**, to provide access to the **SecurityContext**.
- **SecurityContext**, to hold the **Authentication** and possibly request-specific security information.
- **Authentication**, to represent the principal in a Spring Security-specific manner.
- **GrantedAuthority**, to reflect the application-wide permissions granted to a principal.
- **UserDetails**, to provide the necessary information to build an **Authentication** object from your application’s DAOs or other source of security data.
- **UserDetailsService**, to create a **UserDetails** when passed in a String-based username (or certificate ID or the like).

Filters
- DelegatingFilterProxy
- springSecurityFilterChain -> UsernamePasswordAuthenticationFilter, FilterSecurityInterceptor

@Secured("ROLE_ADMIN")
@PreAuthorize ("hasRole('ROLE_WRITE')")
@PostFilter ("filterObject.owner == authentication.name")   filter collection or arrays


## Spring Data

`@Transactional` - for the JPA module we have this annotation on the implementation class backing the proxy (SimpleJpaRepository). This is due to persisting and deleting objects requires a transaction in JPA.

Reading methods like findAll() and findOne(…) are using `@Transactional(readOnly = true)` which is not strictly necessary but triggers a few optimizations in the transaction infrastructure (setting the **FlushMode** to **MANUAL** to let persistence providers potentially *skip dirty checks* when closing the EntityManager). Beyond that the flag is set on the JDBC Connection as well which causes further optimizations on that level.

Typically, you want the **readOnly** flag to be set to **true**, as most of the query methods only read data. For modifying methodswe need the `@Modifying` annotation and overrides the transaction configuration. Thus, the method runs with the **readOnly** flag set to **false**.


## AOP

- **Advice**: @Before, @After, @Around, @AfterThrowning/Returning
- **Pointcut** is a predicate or expression that matches join points `@After("execution(* EmployeeManager.getEmployeeById(..))")`
- **Join point** always represents a method execution. **Actually a method.**
- **Aspect**  is associated with a specific feature of a program. It is a feature or characteristic that crosses over the object. **Actually is a class.**


## Gotchas

#### @Async and @PostConstruct don't work in combination

Invoking an `@PostConstruct` callback asynchronously will probably lead to the situation, that a bean is still initializing although it is considered completely initialized by the container after the bean leaves the post-processing phase. So this might lead to situations where one can call `ApplicationContext.getBean(…)` getting a reference to a bean that is not completely initialized.