---
kind: logging_system
name: 基于自定义 Log4AltoroJ 单例的日志系统
category: logging_system
scope:
    - '**'
source_files:
    - build.gradle
    - WebContent/WEB-INF/init.log
    - src/com/ibm/security/appscan/listener/StartupListener.java
    - src/com/ibm/security/appscan/util/ServletUtil.java
    - src/com/ibm/security/appscan/util/DBUtil.java
    - src/com/ibm/security/appscan/servlet/LoginServlet.java
    - src/com/ibm/security/appscan/api/LoginAPI.java
---

## 1. 使用的系统与框架

AltoroJ 采用**自研的单例日志门面 `com.ibm.security.appscan.Log4AltoroJ`**，底层依赖 **Apache Log4j 1.2.16**（见 `build.gradle` 中 `log4j:log4j:1.2.16`）。该门面类以 JAR 形式随应用分发（`WebContent/WEB-INF/lib/Log4AltoroJ.jar`），在构建时通过 `fileTree(dir: "${webAppDirName}/WEB-INF/lib", include: '*.jar')` 被打包进 WAR。

此外，代码中还残留少量裸 `System.out.println` 调用（如 `LoginAPI.java`、`StartupListener.java`、`DBUtil.java`、`ServletUtil.java`），属于遗留调试输出，并非正式日志通道。

## 2. 关键文件与包

- `src/com/ibm/security/appscan/Log4AltoroJ.java`（JAR 内）：日志门面，提供 `getInstance()` 单例及 `logInfo` / `logError` / `logTransaction` 等方法，并暴露 `getLogFileLocation()` 供上层读取日志路径。
- `src/com/ibm/security/appscan/util/ServletUtil.java`：负责初始化日志文件（`initLogFile`）、加载 demo 属性、读取日志位置；是日志子系统的主要协调者。
- `src/com/ibm/security/appscan/listener/StartupListener.java`：Web 容器启动时触发初始化，捕获异常并通过 `Log4AltoroJ.getInstance().logError(...)` 记录。
- `src/com/ibm/security/appscan/util/DBUtil.java`：数据库操作的核心日志点——初始化成功 (`logInfo("Database initialized")`)、SQL 异常 (`logError(e.getMessage())`)、转账事务 (`logTransaction(debitAccount..., creditAccount..., amount)`)。
- `src/com/ibm/security/appscan/servlet/LoginServlet.java`：登录失败时记录 `"Login failed >>> User: ... >>> Password: ..."`。
- `WebContent/WEB-INF/init.log`：运行时生成的示例日志文件，展示最终输出格式。
- `build.gradle`：声明 `log4j:log4j:1.2.16` 依赖。

## 3. 架构与约定

### 门面 + 单例模式
所有业务代码不直接持有 `Logger` 实例，而是统一通过 `Log4AltoroJ.getInstance()` 获取全局单例后调用方法。这使日志实现可被替换，同时避免在每个类中维护 logger 引用。

### 三个固定 API
门面对外仅暴露三类方法：
- `logInfo(String)` — 一般信息（如数据库初始化、交易摘要）
- `logError(String)` — 错误信息（如 SQL 异常、属性初始化失败、Swagger 初始化失败）
- `logTransaction(String from, String to, BigDecimal amount)` — 专门记录资金转账的审计型日志

### 结构化字段约定
从 `init.log` 可见，日志行遵循固定的文本结构：
- 时间戳 + 级别 + 消息，例如：
  - `INFO Database initialized`
  - `ERROR Column '0' not found.`
  - `ERROR Login failed >>> User: jsmith >>> Password: demo123`
  - `INFO Transaction >>> From: 800003 - Checking >>> To: 4539082039396288 - Credit Card >>> Amount: $489.15`
- 转账日志使用 `>>>` 分隔符串联 `From` / `To` / `Amount` 三个字段，便于后续解析。

### 日志文件管理
`ServletUtil.initLogFile()` 通过 `Log4AltoroJ.getInstance().getLogFileLocation()` 取得日志路径，再创建 `File` 写入。若初始化失败，则回退到 `System.out` 打印错误（见 `ServletUtil.java` L287-L288 的双写逻辑）。

### 启动阶段日志
`StartupListener.contextInitialized()` 在 Web 应用启动时执行初始化，任何异常均通过 `Log4AltoroJ.getInstance().logError(...)` 上报，确保即使应用初始化失败也能留下诊断痕迹。

## 4. 约定与约束

- **日志门面唯一入口**：业务层应通过 `com.ibm.security.appscan.Log4AltoroJ.getInstance()` 访问日志，而非直接使用 Log4j 的 `org.apache.log4j.Logger`。当前代码库中所有正式日志点均遵循此约定。
- **日志级别语义**：`logInfo` 用于正常业务流程事件（数据库就绪、交易流水），`logError` 用于异常与配置失败；没有更细粒度的 DEBUG/TRACE 级别使用。
- **转账审计字段顺序**：`logTransaction(from, to, amount)` 的参数顺序固定为 debit → credit → fee，对应 `init.log` 中的 `From >>> To >>> Amount` 三段式格式。
- **遗留 System.out 输出**：部分类仍保留 `System.out.println`（`LoginAPI.java`、`StartupListener.java`、`DBUtil.java`、`ServletUtil.java`），这些是历史遗留的调试语句，不属于正式日志通道，且可能泄露敏感信息（如密码明文出现在 `LoginServlet` 的错误日志中）。
- **日志文件位置由门面决定**：上层组件（`ServletUtil`）只负责消费 `getLogFileLocation()` 返回的路径，不自行构造日志文件名或目录。
- **无 SLF4J / JUL 桥接**：项目未引入 SLF4J 或 java.util.logging 适配层，日志实现完全绑定到 Log4j 1.2.x。

## 5. 风险与注意事项

- `LoginServlet` 将用户输入的密码原样写入错误日志（`"Login failed >>> User: " + username + " >>> Password: " + password`），构成敏感信息泄露风险。
- 转账金额等敏感数据也通过纯文本写入日志，需评估合规性。
- 使用已停更的 Log4j 1.2.16，存在已知安全漏洞（CVE-2019-17571 等），生产环境建议升级至 Log4j 2.x 并迁移到 SLF4J 门面。