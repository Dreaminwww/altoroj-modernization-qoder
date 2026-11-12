---
kind: configuration_system
name: AltoroJ 基于 ServletContextListener + app.properties 的轻量配置系统
category: configuration_system
scope:
    - '**'
source_files:
    - WebContent/WEB-INF/web.xml
    - WebContent/WEB-INF/app.properties
    - src/com/ibm/security/appscan/altoromutual/listener/StartupListener.java
    - src/com/ibm/security/appscan/altoromutual/util/ServletUtil.java
---

## 1. 使用的系统与方式

AltoroJ 没有引入任何外部配置框架（如 Spring Boot、Lightbend Config、Hikari 等），而是采用最基础的 Java EE 原生机制：
- **Servlet 3.0 `ServletContextListener`** 作为应用启动钩子，负责加载所有运行时配置。
- **Java Properties 文件** (`WebContent/WEB-INF/app.properties`) 承载业务开关与数据库连接参数。
- **`web.xml`** 声明监听器、Filter、Servlet 以及 Jersey JAX-RS 应用的初始化参数。
- **Swagger UI 的 `swagger/properties.json`** 通过 `ServletUtil.initializeRestAPI` 在启动时动态改写 `basePath`。

整个配置体系是手写解析、内存缓存的单例式实现，没有任何类型安全或 schema 校验。

## 2. 关键文件

- `WebContent/WEB-INF/web.xml` — 部署描述符，声明 `StartupListener`、Filter、Servlet、Jersey `javax.ws.rs.Application` 初始参数。
- `WebContent/WEB-INF/app.properties` — 应用行为开关与数据库相关属性（`enableAdminFunctions`、`specialLink`、`advancedStaticPageProcessing`、`enableFeedbackRetention`、`database.alternateDataSource`、`database.reinitializeOnStart`）。
- `src/com/ibm/security/appscan/altoromutual/listener/StartupListener.java` — 应用启动入口，调用 `ServletUtil.initializeAppProperties`、`initializeLogFile`、`DBUtil.isValidUser`、`initializeRestAPI`。
- `src/com/ibm/security/appscan/altoromutual/util/ServletUtil.java` — 配置加载与访问的核心工具类，持有静态 `HashMap<String, String> demoProperties`。
- `WebContent/swagger/properties.json` — Swagger API base path 配置文件，由代码在启动时覆盖写入。

## 3. 架构与约定

### 3.1 启动时序
`web.xml` 中注册了 `com.ibm.security.appscan.altoromutual.listener.StartupListener`。容器启动时依次执行：
1. `ServletUtil.initializeAppProperties(servletContext)` — 从 `/WEB-INF/app.properties` 读取键值对到 `demoProperties`。
2. `ServletUtil.initializeLogFile(servletContext)` — 将 `WEB-INF/init.log` 复制到磁盘日志位置（仅当目标不存在或非空时）。
3. `DBUtil.isValidUser("bogus", "user")` — 触发 JDBC 驱动加载与 HSQLDB 初始化。
4. `ServletUtil.initializeRestAPI(servletContext)` — 读取并改写 `swagger/properties.json` 中的 `basePath` 为 `${contextPath}/api`。

### 3.2 属性加载逻辑（`ServletUtil.initializeAppProperties`）
- 使用 `ServletContext.getResourceAsStream("/WEB-INF/app.properties")` 获取流。
- 逐行解析，最多处理前 99 行（硬编码 `counter < 99`）。
- 支持 `#` 和 `//` 注释；以 `=` 分割 key/value；非注释且包含 `=` 的行才入库。
- 解析失败的非注释行会记录错误日志并通过 `System.out.println` 输出。
- 结果保存在类级静态字段 `demoProperties`，后续通过 `getAppProperty(key)` 和 `isAppPropertyTrue(key)` 访问。

### 3.3 属性访问约定
- `ServletUtil.getAppProperty(String)` 返回字符串值，未找到时返回空字符串 `""`。
- `ServletUtil.isAppPropertyTrue(String)` 将值转为小写后判断是否等于 `"true"`（使用 `StringTokenizer` 按空格/引号分词，要求恰好一个 token）。
- 业务代码直接调用这两个方法，而不是通过 `ServletContext.getInitParameter` 或环境变量。

### 3.4 运行期可写配置
`initializeRestAPI` 会打开 `servletContext.getRealPath("swagger/properties.json")` 并覆写 `basePath` 字段，把 WAR 内的 JSON 改写到磁盘。这是仓库中唯一被代码主动修改的配置文件。

### 3.5 无环境变量 / 无多环境分层
仓库中没有 `.env`、`application.yml`、`application-dev.properties` 之类的多环境配置。所有可配置项集中在单个 `app.properties` 中，通过取消注释来启用功能。

## 4. 约定与约束

- **配置文件位置固定**：应用行为配置必须放在 `WebContent/WEB-INF/app.properties`，由 `ServletContext.getResourceAsStream("/WEB-INF/app.properties")` 加载（见 `ServletUtil.initializeAppProperties` 第 264 行）。其他路径不会被扫描。
- **文件格式约定**：每行 `key=value`，注释以 `#` 或 `//` 开头；非注释且不含 `=` 的行会被记录为解析错误（见 `ServletUtil` 第 285–289 行）。
- **最大行数限制**：只读取前 99 行（`counter < 99`），超出部分被忽略（见 `ServletUtil` 第 275 行）。
- **布尔属性判定规则**：只有当值的第一个 token（去除空格和引号后）严格等于 `"true"`（不区分大小写）时才视为真，其余情况均为假（见 `ServletUtil.isAppPropertyTrue` 第 211–218 行）。
- **缺失属性默认值**：`getAppProperty` 在未找到 key 时返回空字符串，而非抛出异常（见第 201–208 行）。
- **Swagger basePath 自动覆盖**：启动时若 `swagger/properties.json` 中的 `basePath` 不等于 `${contextPath}/api`，则直接覆写该文件（见 `ServletUtil.initializeRestAPI` 第 384–398 行）。
- **无敏感信息注入通道**：仓库未发现对 `System.getenv()`、`System.getProperty()` 或 JNDI 的调用用于加载应用配置（`app.properties` 中的 `database.alternateDataSource` 只是 Tomcat JNDI 数据源名称，实际数据源由 Tomcat 配置）。
- **配置不可热重载**：`demoProperties` 是静态 HashMap，仅在 `contextInitialized` 时加载一次，运行期修改 `app.properties` 不会生效。
- **Swagger 配置需手动匹配 context path**：如果自动改写失败，日志提示需要手动设置 `basePath` 以匹配上下文路径加 `/api` 后缀（见 `ServletUtil` 第 400–403 行的错误消息）。