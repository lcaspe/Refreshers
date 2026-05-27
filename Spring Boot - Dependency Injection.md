## What is Dependency Injection?

At its core, Dependency Injection is a design pattern used to achieve **Inversion of Control (IoC)** between classes and their dependencies.

In plain English: **Instead of a class creating the objects it needs to do its job, those objects are "injected" into the class from the outside.**

### The "Old Way" (Without DI)

Imagine you are building a car, and the car is responsible for manufacturing its own engine.

```java
public class Car {
    private Engine engine;

    public Car() {
        // The Car is tightly coupled to this specific V8Engine
        this.engine = new V8Engine();
    }
}
```

- **The Problem:** If you want to change the car to use an `ElectricEngine`, you have to rewrite the `Car` class. It's tightly coupled, hard to test, and hard to maintain.

### The "DI Way" (With DI)

Instead of the car making the engine, you pass the engine into the car's constructor.

```java
public class Car {
    private Engine engine;

    // The engine is supplied (injected) from the outside
    public Car(Engine engine) {
        this.engine = engine;
    }
}

```

- **The Benefit:** The `Car` doesn't care what kind of `Engine` it gets, as long as it implements the `Engine` interface. It is now loosely coupled and incredibly easy to test (you can pass a "mock" engine).

## How Spring Boot Handles DI

In Spring Boot, you don't have to manually pass these objects around. Spring has an **IoC Container** (the application context) that acts as a factory. It creates the objects, manages their lifecycle, and injects them where they are needed.

We call these Spring-managed objects **Beans**.

### The 3 Ways to Inject Dependencies in Spring

Spring Boot gives you three main ways to inject dependencies. Here they are, ranked from best practice to discouraged:

#### 1. Constructor Injection (Recommended)

The dependencies are provided through the class constructor.

```java
@Service
public class CarService {

    private final Engine engine;

    // Spring automatically injects the Engine bean here
    @Autowired
    public CarService(Engine engine) {
        this.engine = engine;
    }
}

```

> **Why it’s the best:** It allows you to make dependencies `final` (immutable), ensures the class can't be initialized without its required dependencies, and makes unit testing simple since you can just pass mocks into the constructor. _Note: As of Spring 4.3, if a class has only one constructor, you can actually omit the `@Autowired` annotation entirely!_

#### 2. Setter Injection

The dependencies are provided through public setter methods.

```java
@Service
public class CarService {

    private Engine engine;

    @Autowired
    public void setEngine(Engine engine) {
        this.engine = engine;
    }
}

```

> **When to use:** Good for optional dependencies that can be changed or initialized later.

#### 3. Field Injection (Discouraged)

The dependencies are injected directly into the fields using reflection.

```java
@Service
public class CarService {

    @Autowired
    private Engine engine; // Clean looking, but dangerous
}

```

> **Why it’s discouraged:** It hides dependencies, makes unit testing harder (you are forced to use Spring rules or reflection just to instantiate the class in a test), and allows the class to violate the Single Responsibility Principle too easily.

---

## Configuring Beans Using Annotations

In modern Spring Boot, we do this almost entirely using **Annotations**. There are two main ways to configure beans: **Stereotype Annotations** (implicit/automatic) and **Java-Based Configuration** (explicit).

## 1. Stereotype Annotations (Component Scanning)

Stereotype annotations are applied directly to your Java classes. When Spring Boot starts up, it performs a **Component Scan** (starting from your main `@SpringBootApplication` class) and automatically creates beans for any class marked with these annotations.

Here is the breakdown of the primary stereotype annotations:

| Annotation                        | Purpose                                                                                                  | Best Used For                                                                              |
| --------------------------------- | -------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------ |
| `@Component`                      | The generic stereotype for any Spring-managed component.                                                 | General utility classes, helpers, or anything that doesn't fit a specific layer.           |
| `@Service`                        | A specialization of `@Component`.                                                                        | The **Business Logic Layer**. Contains calculations, orchestrates data, and applies rules. |
| `@Repository`                     | A specialization of `@Component` that also enables automatic database exception translation.             | The **Data Access Layer (DAO)**. Interfaces or classes interacting with the database.      |
| `@RestController` / `@Controller` | Specializations that handle HTTP requests. `@RestController` combines `@Controller` and `@ResponseBody`. | The **Presentation Layer**. Exposing REST APIs or serving MVC views.                       |

### Example of Stereotype Usage:

```java
package com.example.demo.service;

import org.springframework.stereotype.Service;

@Service // Tells Spring: "Hey, manage this class as a bean, it's a service!"
public class PaymentService {

    public void processPayment(double amount) {
        System.out.println("Processing payment of $" + amount);
    }
}

```

## 2. Java-Based Configuration (`@Configuration` and `@Bean`)

Sometimes, you cannot use stereotype annotations. For example, if you are using a third-party library (like Apache Commons or a JWT library), you cannot open their compiled `.class` files to add `@Component` on top of them.

This is where **Java-based configuration** comes in. You create a configuration class and manually define the beans inside it.

- **`@Configuration`**: Indicates that a class declares one or more `@Bean` methods.
- **`@Bean`**: Placed on a _method_ inside a configuration class. Spring runs this method, takes the object returned, and registers it as a bean in the IoC container.

### Example of Explicit Bean Configuration:

```java
package com.example.demo.config;

import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import com.thirdparty.ExternalPaymentProcessor; // Imagine this is from a dependency jar

@Configuration // Tells Spring this class contains bean definitions
public class AppConfig {

    @Bean // The method name "externalProcessor" becomes the bean name by default
    public ExternalPaymentProcessor externalProcessor() {
        // You handle the manual instantiation and setup here
        ExternalPaymentProcessor processor = new ExternalPaymentProcessor();
        processor.setApiKey("12345-ABCDE");
        return processor;
    }
}

```

## Stereotype vs. `@Bean`: When to use which?

- **Use Stereotypes (`@Component`, `@Service`, etc.)** for your own application code. It's clean, requires less boilerplate, and utilizes Spring's automatic discovery.
- **Use `@Bean` inside a `@Configuration` class** when configuring third-party classes, or when you need complex, custom initialization logic that depends on environment properties.

---

## Controlling Bean Selection

Things get interesting when you have **multiple beans of the exact same type**. If Spring tries to inject a dependency and finds more than one matching bean, it gets confused and throws a `NoUniqueBeanDefinitionException`.

To prevent this, Spring Boot provides annotations to explicitly control which bean should be selected. The three heavy hitters here are `@Primary`, `@Qualifier`, and `@Conditional`.

## 1. `@Primary` (The Default Choice)

When multiple beans of the same type exist, `@Primary` tells Spring: _"If no one specifies otherwise, use this one by default."_

### Example Scenario:

Imagine you have a `NotificationService` interface with two implementations: Email and SMS.

```java
public interface NotificationService {
    void send(String message);
}

```

```java
@Component
@Primary // This makes Email the default implementation
public class EmailNotification implements NotificationService {
    public void send(String message) { System.out.println("Email sent: " + message); }
}

@Component
public class SmsNotification implements NotificationService {
    public void send(String message) { System.out.println("SMS sent: " + message); }
}

```

Now, if you inject `NotificationService` normally, Spring will pick `EmailNotification` without throwing an exception:

```java
@RestController
public class NotificationController {

    private final NotificationService notificationService;

    // Spring sees two beans, but picks Email because it's marked @Primary
    public NotificationController(NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}

```

## 2. `@Qualifier` (The Specific Choice)

What if you _specifically_ want the SMS implementation in a certain class, but want to keep Email as the default elsewhere? You use `@Qualifier`.

`@Qualifier` lets you name specific beans and request them by that name at the injection point. By default, a bean's qualifier name is the class name in **camelCase** (e.g., `smsNotification`).

### Example Usage:

```java
@RestController
public class UrgentAlertController {

    private final NotificationService notificationService;

    // @Qualifier overrides @Primary by explicitly asking for "smsNotification"
    public UrgentAlertController(@Qualifier("smsNotification") NotificationService notificationService) {
        this.notificationService = notificationService;
    }
}

```

> **Pro Tip:** You can also define custom names for your qualifiers directly on the component, like `@Component("sms")` or `@Qualifier("sms")` at the class level, to avoid relying on class names.

## 3. `@Conditional` (The Dynamic Choice)

Instead of hardcoding preferences, what if you want to select beans dynamically based on the environment, application properties, or whether another bean exists?

Spring Boot provides `@Conditional` annotations for this exact runtime/startup logic. Here are the most common variants:

### A. `@ConditionalOnProperty`

Enables a bean only if a specific property in `application.properties` or `application.yml` matches a certain value.

```java
@Component
@ConditionalOnProperty(name = "payment.gateway", havingValue = "paypal")
public class PaypalProcessor implements PaymentProcessor { ... }

@Component
@ConditionalOnProperty(name = "payment.gateway", havingValue = "stripe")
public class StripeProcessor implements PaymentProcessor { ... }

```

### B. `@ConditionalOnProfile` (Simplified as `@Profile`)

Enables a bean only when a specific Spring Profile is active (e.g., `dev`, `test`, `prod`).

```java
@Component
@Profile("dev") // Only instantiated if spring.profiles.active=dev
public class DevDataSource implements DataSource { ... }

```

### C. `@ConditionalOnMissingBean`

Commonly used in library/framework development. It tells Spring: _"Only create this bean if the user hasn't already defined their own version of it."_

```java
@Configuration
public class AutoConfig {

    @Bean
    @ConditionalOnMissingBean
    public CustomLogger defaultLogger() {
        return new StandardOutputLogger(); // Fallback logger
    }
}

```

## Quick Summary Cheat Sheet

- **Use `@Primary`** when you want a sensible default bean for general use.
- **Use `@Qualifier`** when you need to bypass the default and pick a specific bean by name.
- **Use `@Conditional` / `@Profile`** when bean selection depends on external configuration or environment states.

---

## Externalizing Configuration

When you build a Spring Boot application, you want the same compiled artifact (the `.jar` file) to run seamlessly in your local environment, your testing environment, and your production environment without changing the code.

To achieve this, Spring Boot supports **Externalizing Configuration**, allowing you to maintain properties outside your application code using properties files, YAML files, environment variables, or command-line arguments.

Here is how to access and manage these externalized properties inside your Java classes.

## 1. Reading Properties via `@Value`

The `@Value` annotation is the quickest way to inject single, specific property values directly into your beans. It uses the Spring Expression Language (SpEL) syntax `${property.name}`.

### Example:

Let's say you have this in your `src/main/resources/application.properties`:

```properties
app.bonus.rate=0.15
app.welcome.message=Welcome to the platform!

```

You can inject them into a component like this:

```java
@Component
public class RewardCalculator {

    // Injecting a String
    @Value("${app.welcome.message}")
    private String welcomeMessage;

    // Injecting a primitive/double with a default value of 0.10 if not found
    @Value("${app.bonus.rate:0.10}")
    private double bonusRate;

    // ...
}

```

> **When to use:** Great for quick, isolated properties (like an API flag or a feature toggle). **The Downside:** It becomes messy and error-prone if you have dozens of related configurations, as it lacks type safety and validation.

## 2. Structured Properties via `@ConfigurationProperties` (Recommended)

For complex, hierarchical, or related configurations, Spring Boot provides `@ConfigurationProperties`. This maps a group of properties onto a type-safe Java object using standard bean setters.

### Example Scenario:

Imagine you are configuring a third-party SMS gateway. Your `application.properties` looks like this:

```properties
sms.gateway.url=https://api.sms-provider.com
sms.gateway.api-key=secret-key-123
sms.gateway.timeout-seconds=5

```

Instead of using three separate `@Value` annotations, you create a dedicated configuration class:

```java
@Component
@ConfigurationProperties(prefix = "sms.gateway")
public class SmsGatewayProperties {

    private String url;
    private String apiKey;
    private int timeoutSeconds;

    // Standard getters and setters are REQUIRED for binding
    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }

    public String getApiKey() { return apiKey; }
    public void setApiKey(String apiKey) { this.apiKey = apiKey; }

    public int getTimeoutSeconds() { return timeoutSeconds; }
    public void setTimeoutSeconds(int timeoutSeconds) { this.timeoutSeconds = timeoutSeconds; }
}

```

Now, you can just inject `SmsGatewayProperties` anywhere in your application like a regular bean:

```java
@Service
public class NotificationService {
    private final SmsGatewayProperties smsProperties;

    public NotificationService(SmsGatewayProperties smsProperties) {
        this.smsProperties = smsProperties; // Type-safe and organized!
    }
}

```

> **Why it's awesome:** It supports hierarchical structures (lists, maps, nested objects), provides autocomplete support in modern IDEs, and works cleanly with Java Bean Validation annotations (like `@NotNull`, `@Min`, etc.).

## 3. Order of Precedence (The Hierarchy)

Spring Boot is incredibly smart about _where_ it looks for properties. If the same property is defined in multiple places, Spring Boot resolves the conflict using a strict order of priority.

Here are the most common sources, ranked from **highest priority to lowest priority** (items at the top override items at the bottom):

1. **Command-Line Arguments** (e.g., `--server.port=8081`)
2. **JVM System Properties** (e.g., `-Dserver.port=8081`)
3. **OS Environment Variables** (e.g., `SERVER_PORT=8081`)
4. **Application Properties _outside_ your packaged jar** (e.g., an `application.properties` file in the same directory as your jar execution).
5. **Application Properties _inside_ your packaged jar** (e.g., `src/main/resources/application.properties`).

### Environment Variable Relaxed Binding

When overriding properties via Environment Variables (like in Docker or Kubernetes), Spring Boot uses **relaxed binding**.

- Property in file: `sms.gateway.api-key`
- Equivalent Environment Variable: `SMS_GATEWAY_APIKEY` or `SMS_GATEWAY_API_KEY` (Spring automatically figures it out).

## 4. Relaxed Binding Cheat Sheet

Spring Boot doesn't care if you mix casing styles across your property sources. It maps them all to your Java fields seamlessly:

| Style           | Example               | Where it's best used                            |
| --------------- | --------------------- | ----------------------------------------------- |
| **Kebab-case**  | `sms.gateway.api-key` | Recommended for `.properties` and `.yml` files. |
| **CamelCase**   | `sms.gateway.apiKey`  | Standard Java notation.                         |
| **Snake_case**  | `sms_gateway_api_key` | Alternative formatting for files.               |
| **UPPER_SNAKE** | `SMS_GATEWAY_API_KEY` | Recommended for System Environment Variables.   |

---

## Lazy Initialization

By default, when a Spring Boot application starts up, the IoC container creates and initializes all configured beans immediately. This is known as **Eager Initialization**.

While eager initialization is great for catching configuration errors or missing dependencies early at startup, it can slow down your application's boot time, especially if you have a lot of heavy beans (like database connections, cloud clients, or complex processing engines).

This is where **Lazy Initialization** comes into play.

## What is Lazy Initialization?

Lazy initialization tells the Spring container: _"Do not create this bean when the application starts. Wait until someone actually requests it or tries to inject it for the very first time."_

### The Pros and Cons

| Advantages                                                                                                                  | Disadvantages                                                                                                                                                                            |
| --------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Faster Startup Time:** The application boots up much faster because it does less work at launch.                          | **Delayed Errors:** If a bean has a configuration bug (e.g., a wrong URL or missing credential), you won't find out at startup. It will fail at runtime when a user hits that code path. |
| **Lower Memory Footprint:** Beans that are never used during a specific execution run are never created, saving JVM memory. | **Latent First Request:** The very first HTTP request or action that triggers the bean's creation will experience a slight delay (latency spike).                                        |

## How to Enable Lazy Initialization

Spring Boot gives you two ways to configure this: per individual bean, or globally across the entire application.

### 1. Per-Bean Configuration using `@Lazy`

You can drop the `@Lazy` annotation on specific components or `@Bean` methods that you know are heavy and rarely used.

#### On a Component (Stereotype):

```java
@Component
@Lazy // This bean will not be created until it's requested
public class HeavyReportGenerator {
    public HeavyReportGenerator() {
        System.out.println("HeavyReportGenerator initialized!"); // Only prints when called
    }
}

```

#### On a Java-Based Configuration Bean:

```java
@Configuration
public class ThirdPartyConfig {

    @Bean
    @Lazy
    public ComplexCloudClient complexCloudClient() {
        return new ComplexCloudClient();
    }
}

```

#### At the Injection Point:

If you have an eager bean that depends on a lazy bean, you must also put `@Lazy` at the injection point (constructor or field). Otherwise, Spring will be forced to initialize the lazy bean immediately just to satisfy the eager bean's dependency!

```java
@Service
public class ReportService {

    private final HeavyReportGenerator reportGenerator;

    // @Lazy here tells Spring to inject a 'proxy' instead of initializing the real bean right away
    public ReportService(@Lazy HeavyReportGenerator reportGenerator) {
        this.reportGenerator = reportGenerator;
    }
}

```

### 2. Global Configuration (Recommended for Local Dev)

If you are working on a massive enterprise project, waiting 45 seconds for the app to restart locally after every minor code change is painful. You can force **all** beans in your application to be lazy via your configuration file.

Add this line to your `application.properties`:

```properties
spring.main.lazy-initialization=true

```

Or in `application.yml`:

```yaml
spring:
main:
lazy-initialization: true
```

> **Best Practice Pattern:** It is highly recommended to turn global lazy initialization **ON** for your local development profile to keep your loop tight, but keep it **OFF** (default eager) in production so your production container crashes immediately at deployment if there is a configuration error, rather than failing on a live user.

---

## Bean Scopes

In Spring Boot, a **Bean Scope** defines the lifecycle and visibility of a bean instance within the IoC container. It determines _how many_ instances of a bean Spring will create and _when_ it will share an existing instance versus creating a new one.

Spring provides several scopes out of the box, which can be categorized into standard scopes and web-aware scopes.

## 1. Standard Bean Scopes

These two scopes are available in any Spring application (console, desktop, or web).

### A. Singleton Scope (Default)

When a bean is a Singleton, the Spring IoC container creates **exactly one instance** of that bean object. Every time that bean is requested or injected into another class, Spring returns that same cached instance.

```java
@Component
@Scope("singleton") // This is optional, as singleton is the default
public class OrderService {
    // Stateful fields here are NOT thread-safe!
}

```

- **Thread Safety Warning:** Because a single instance is shared across the entire application, Singleton beans must be **stateless** or thread-safe. Avoid storing user-specific data in class member variables.

### B. Prototype Scope

When a bean is a Prototype, the Spring IoC container creates a **brand-new instance** of the bean object every single time it is requested or injected.

```java
@Component
@Scope("prototype") // Or use the constant ConfigurableBeanFactory.SCOPE_PROTOTYPE
public class ReportGenerator {
    // Great for holding state unique to a specific task execution
}

```

- **Lifecycle Note:** Spring creates the prototype bean and hands it over to the client, but Spring does _not_ manage the complete lifecycle afterward. Destruction lifecycle callbacks (like `@PreDestroy`) are not called for prototype beans; the client code must clean them up or let the garbage collector handle them.

## 2. Web-Aware Bean Scopes

These scopes are only available if you are running a web application (using `spring-boot-starter-web`). They bind the lifecycle of a bean to HTTP mechanisms.

| Scope           | Annotation          | Lifecycle Duration                                                                                                                                                     |
| --------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Request**     | `@RequestScope`     | A new bean instance is created for **every single HTTP request**. It is discarded when the request completes.                                                          |
| **Session**     | `@SessionScope`     | A new bean instance is created once per **HTTP Session**. It lives across multiple requests from the same user.                                                        |
| **Application** | `@ApplicationScope` | A single bean instance lives for the lifecycle of the `ServletContext`. (Similar to Singleton, but scoped to the servlet level if multiple apps run in one container). |

### Example Usage:

```java
@Component
@RequestScope // Shorthand for @Scope(value = WebApplicationContext.SCOPE_REQUEST)
public class UserPreferences {
    private String theme;
    private String language;
    // Safe to store state here because this object belongs to exactly one HTTP request thread
}

```

## The Prototype-in-Singleton Problem (Injection Injection)

A classic Spring interview question and architectural trap happens when you inject a **Prototype bean into a Singleton bean**.

Because the Singleton bean is initialized only once during startup, its dependencies are also injected **only once**. Therefore, the Prototype bean is instantiated at startup, injected into the Singleton, and _never changes again_. It behaves like a Singleton!

### The Solution: `@Lookup` or `ObjectProvider`

If your Singleton bean truly needs a fresh instance of a Prototype bean every time a method is called, use `ObjectProvider`.

```java
@Service
public class OrderProcessingService { // Singleton

    private final ObjectProvider<ReportGenerator> reportGeneratorProvider;

    public OrderProcessingService(ObjectProvider<ReportGenerator> reportGeneratorProvider) {
        this.reportGeneratorProvider = reportGeneratorProvider;
    }

    public void processOrder() {
        // .getObject() forces Spring to generate a brand-new Prototype instance here
        ReportGenerator freshProcessor = reportGeneratorProvider.getObject();
        freshProcessor.generate();
    }
}

```

## Summary Cheat Sheet

- **Use Singleton (Default)** for stateless services, controllers, and repositories.
- **Use Prototype** for stateful components or objects that shouldn't be shared across multiple executions.
- **Use Web Scopes** (`@RequestScope`, `@SessionScope`) when the bean needs to track user-specific web data securely across a request or session.

We have covered the core IoC container fundamentals (DI, Annotations, Configuration, Selection, Properties, Laziness, and Scopes)!

---

## Bean Lifecycle Hooks

In Spring Boot, a bean is not just instantiated and thrown into the wild. It goes through a highly choreographed lifecycle managed by the Spring IoC Container.

Understanding this lifecycle allows you to execute custom code at specific phases—most notably, right after a bean is fully set up (**Initialization**) and right before a bean is destroyed when the application shuts down (**Destruction**).

## The Bean Lifecycle at a Glance

When the Spring container starts up, every eager singleton bean goes through the following phases:

```
[1. Instantiation] -> [2. Populate Properties (DI)] -> [3. Initialization Hooks] -> [Bean is READY for use] -> [4. Destruction Hooks]

```

1. **Instantiation:** Spring creates the instance of the bean using its constructor (allocates memory).
2. **Populate Properties:** Spring injects all required dependencies via field, setter, or constructor injection.
3. **Initialization:** Spring runs setup routines. This is your first chance to interact with the bean.
4. **Destruction:** When the application context closes, Spring cleans up the bean.

## How to Hook into Initialization and Destruction

Spring Boot provides three ways to inject custom logic into the initialization and destruction phases.

### 1. Annotations: `@PostConstruct` and `@PreDestroy` (Recommended)

This is the modern standard approach. It keeps your code clean and decouples your bean from Spring-specific interfaces.

- **`@PostConstruct`**: Runs immediately _after_ the bean has been fully initialized and dependencies have been injected. Perfect for starting background tasks, caching initial data, or validating configuration.
- **`@PreDestroy`**: Runs right _before_ the bean is removed from the context (e.g., when the application shuts down). Ideal for closing file streams, releasing database connections, or disconnecting from message brokers.

```java
@Component
public class CacheManager {

    private Map<String, String> localCache;

    @PostConstruct
    public void init() {
        // Dependencies are guaranteed to be ready here
        this.localCache = new ConcurrentHashMap<>();
        System.out.println("Cache initialized successfully.");
    }

    @PreDestroy
    public void cleanup() {
        // Executed on application shutdown
        this.localCache.clear();
        System.out.println("Cache cleared. Releasing resources.");
    }
}

```

### 2. Java Configuration: `@Bean(initMethod, destroyMethod)`

If you are configuring beans using code (for third-party libraries where you cannot modify the source code to add annotations), you can specify your lifecycle hooks directly in the `@Bean` definition.

```java
public class CustomDatabaseClient {
    public void connect() { System.out.println("Connected to custom DB."); }
    public void disconnect() { System.out.println("Disconnected from custom DB."); }
}

@Configuration
public class ThirdPartyConfig {

    // Spring will automatically invoke connect() after creation and disconnect() at shutdown
    @Bean(initMethod = "connect", destroyMethod = "disconnect")
    public CustomDatabaseClient customClient() {
        return new CustomDatabaseClient();
    }
}

```

### 3. Interfaces: `InitializingBean` and `DisposableBean` (Legacy)

This is an older approach where your Java class explicitly implements Spring’s lifecycle interfaces. It is generally discouraged today because it tightly couples your code directly to the Spring framework.

```java
@Component
public class OldSchoolBean implements InitializingBean, DisposableBean {

    @Override
    public void afterPropertiesSet() throws Exception {
        // Equivalent to @PostConstruct
    }

    @Override
    public void destroy() throws Exception {
        // Equivalent to @PreDestroy
    }
}

```

## Summary: A Crucial Gotcha to Remember

> **Warning:** Lifecycle hooks behave differently based on the **Bean Scope**.

- **Singleton Beans:** Both initialization (`@PostConstruct`) and destruction (`@PreDestroy`) hooks work completely as expected.
- **Prototype Beans:** Spring will call the initialization hook (`@PostConstruct`), but it **will not call the destruction hook (`@PreDestroy`)**. Because Spring does not manage the full lifecycle of prototype beans after handing them off to the client, your calling code is entirely responsible for manual cleanup of prototype objects.

That wraps up the core architectural mechanics of Spring Beans!
