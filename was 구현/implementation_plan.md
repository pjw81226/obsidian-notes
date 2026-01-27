# Mini-Spring Framework Implementation Plan

## 1. Project Overview
* **Goal**: Build a custom Web Application Server (Tomcat-like) and an MVC Framework (Spring-like) from scratch using Pure Java.
* **Objective**: Understand the internal mechanisms of Servlet Containers, DispatcherServlet, DI (Dependency Injection), and MVC execution flow.
* **Constraints**:
    * Language: Java 17+
    * Dependencies: `Jackson` (only for JSON parsing), No other frameworks allowed.
    * Architecture: Multi-threaded Blocking I/O (BIO) Server + Front Controller Pattern.

---

## 2. Package Structure (Proposed)
The project is divided into three main layers: `server` (Tomcat), `framework` (Spring), and `application` (Test App).

```text
src/main/java
├── com.minispring
│   ├── server                 <-- [Phase 1 & 2] Tomcat Implementation
│   │   ├── MiniTomcat.java    (Main Entry Point)
│   │   ├── Connector.java     (ServerSocket & ThreadPool)
│   │   ├── HttpRequest.java   (Parsing Logic)
│   │   ├── HttpResponse.java
│   │   └── Servlet.java       (Interface)
│   │
│   ├── framework              <-- [Phase 3, 4, 5] Spring Core Implementation
│   │   ├── context            (DI Container)
│   │   │   ├── ApplicationContext.java
│   │   │   └── annotation     (@Controller, @Service, @Autowired)
│   │   ├── web                (MVC Core)
│   │   │   ├── DispatcherServlet.java
│   │   │   ├── HandlerMapping.java
│   │   │   ├── HandlerAdapter.java
│   │   │   ├── filter         (Filter Interface & Chain)
│   │   │   └── interceptor    (HandlerInterceptor)
│   │   ├── resolver           (Argument Resolvers)
│   │   └── view               (ReturnValue Handlers & MessageConverter)
│   │
│   └── application            <-- User Application Code
│       ├── controller         (Test Controllers)
│       ├── service            (Test Services)
│       └── dto                (Test DTOs)

```
---

## 3. Detailed Implementation Phases

### Phase 1: Infrastructure & Protocol Layer (The Server)

**Goal**: Establish TCP connection and parse HTTP/1.1 messages.

- [ ] **Step 1.1: Server Socket Setup**
    - Implement `MiniTomcat.java`.
        
    - Use `java.net.ServerSocket` to listen on port 8080.
        
    - Implement a `while(true)` loop to accept connections.
        
    - Integrate `java.util.concurrent.ExecutorService` for thread pooling (handle multiple requests).
        
- [ ] **Step 1.2: HTTP Parsing**
    
    - Implement `HttpRequest.java`.
        
    - Read `InputStream` from the socket.
        
    - Parse the Request Line (`METHOD URI PROTOCOL`).
        
    - Parse Headers (store in `Map<String, String>`).
        
    - Parse Body (if exists).
        
- [ ] **Step 1.3: Response Object**
    
    - Implement `HttpResponse.java`.
        
    - Provide methods to write status codes, headers, and body to `OutputStream`.
        

### Phase 2: Servlet Container & Filter Chain

**Goal**: Abstract the request handling and implement the Filter Chain pattern.

- [ ] **Step 2.1: Servlet Interface**
    
    - Define `Servlet` interface with `init()`, `service(req, res)`, and `destroy()`.
        
    - Refactor `MiniTomcat` to delegate request processing to a Servlet implementation.
        
- [ ] **Step 2.2: Filter Implementation**
    
    - Define `Filter` interface (`doFilter(req, res, chain)`).
        
    - Implement `FilterChain` to manage a list of filters.
        
    - Create a test filter (e.g., `EncodingFilter`, `LoggingFilter`) to verify the chain execution order.
        

### Phase 3: Spring Core & DispatcherServlet

**Goal**: Implement the Front Controller pattern and Dependency Injection (DI).

- [ ] **Step 3.1: Annotation & Scanning**
    
    - Create annotations: `@Controller`, `@Service`, `@RequestMapping`, `@Autowired`.
        
    - Implement `ApplicationContext` (IoC Container).
        
    - Use `Reflections` (or file system scanning) to find classes with annotations in the `application` package.
        
- [ ] **Step 3.2: Bean Lifecycle & DI**
    
    - Instantiate found classes using `clazz.getDeclaredConstructor().newInstance()`.
        
    - Store instances in a `Map<String, Object> beans`.
        
    - Iterate through fields of beans; if `@Autowired` is present, inject the dependency using `field.set()`.
        
- [ ] **Step 3.3: DispatcherServlet & HandlerMapping**
    
    - Implement `DispatcherServlet` (extends `Servlet`).
        
    - On `init()`: Scan controllers and build `HandlerMapping` (`Map<String, Method>`).
        
    - On `service()`: Match the request URI to a controller method.
        

### Phase 4: MVC Strategy (Input Processing)

**Goal**: Execute methods dynamically and bind arguments automatically.

- [ ] **Step 4.1: HandlerAdapter**
    
    - Implement `HandlerAdapter` to decouple execution logic from `DispatcherServlet`.
        
    - Use `method.invoke(controllerInstance, args)` to execute the target method.
        
- [ ] **Step 4.2: HandlerMethodArgumentResolver**
    
    - Interface: `boolean supportsParameter(Parameter p)` and `Object resolveArgument(...)`.
        
    - Implement `RequestParamResolver`: Extract query parameters.
        
    - Implement `RequestBodyResolver`: Use Jackson to deserialize JSON body to Java Object (DTO).
        

### Phase 5: Response & Exception Handling (Output Processing)

**Goal**: Handle return values and global exceptions.

- [ ] **Step 5.1: HandlerMethodReturnValueHandler**
    
    - Check if the method has `@ResponseBody`.
        
    - If yes, serialize the return object to JSON.
        
- [ ] **Step 5.2: HttpMessageConverter**
    
    - Wrap `Jackson` ObjectMapper to convert Java Objects to JSON strings and write to `HttpResponse`.
        
- [ ] **Step 5.3: HandlerExceptionResolver**
    
    - Wrap the `DispatcherServlet.doDispatch()` logic in a `try-catch` block.
        
    - Define `@ExceptionHandler`.
        
    - If an exception occurs, catch it, look up the handler, and write a standard JSON error response.
        

---

## 4. Execution Flow Summary

1. **Client** sends HTTP Request.
    
2. **MiniTomcat** accepts socket -> creates Thread.
    
3. **Filters** process request (Logging/Encoding).
    
4. **DispatcherServlet** receives request.
    
5. **HandlerMapping** finds the Controller Method.
    
6. **HandlerAdapter** prepares execution.
    
    - **ArgumentResolvers** convert JSON/Params to Java Objects.
        
7. **Controller** executes business logic.
    
8. **ReturnValueHandlers** convert return value (DTO) to JSON.
    
9. **DispatcherServlet** sends response.
    
10. **Socket** closes.