---
kind: dependency_management
name: AltoroJ Gradle + Maven Central 依赖管理（WAR 工程）
category: dependency_management
scope:
    - '**'
source_files:
    - build.gradle
    - .classpath
    - WebContent/WEB-INF/lib/
---

## 1. 使用的系统与工具

- **构建/依赖工具**：Gradle（`build.gradle`），使用 `war` 插件打包 WAR。
- **仓库源**：仅配置 `mavenCentral()`，无私有仓库、无代理镜像、无 `settings.gradle` 中的仓库覆盖。
- **IDE 集成**：Eclipse Buildship（`.classpath` 中可见 `org.eclipse.buildship.core.gradleclasspathcontainer`），将 Gradle 依赖与 Eclipse JRE/J2EE 容器类路径关联。
- **Java 版本**：`sourceCompatibility/targetCompatibility = "1.7"`，依赖均为 Java EE 6/7 时代的库。

## 2. 关键文件

- `build.gradle` — 唯一声明式依赖清单，定义仓库、依赖项与 WAR 打包行为。
- `.classpath` — Eclipse 类路径描述，指向 Gradle 生成的类路径容器及 `WEB-INF/lib`。
- `WebContent/WEB-INF/lib/` — 本地 jar 目录（当前仅含许可证文本，无业务 jar）。

## 3. 架构与约定

### 3.1 依赖来源混合策略
`build.gradle` 的 `dependencies` 块采用两种来源并存的方式：

| 来源 | 声明方式 | 示例 |
|---|---|---|
| 远程 Maven 仓库 | `group:artifact:version` 字符串 | `'commons-codec:commons-codec:1.6'`、`'org.glassfish.jersey.core:jersey-server:2.27'`、`'log4j:log4j:1.2.16'` |
| 本地文件系统 | `fileTree(dir: "${webAppDirName}/WEB-INF/lib", include: '*.jar')` | 扫描 `WebContent/WEB-INF/lib/*.jar` 作为编译期依赖 |

这意味着：**新增第三方 jar 有两种途径**——在 `build.gradle` 中以坐标声明（推荐），或把 jar 放入 `WebContent/WEB-INF/lib/`（遗留兼容方式）。两者同时生效。

### 3.2 依赖分类
- `compile`：随应用一起打包进 WAR 的运行时依赖（Jersey、Derby、Log4j、Wink、Commons Codec/Lang 等）。
- `providedCompile`：由目标 Servlet 容器（Tomcat 等）提供的 API，不打入 WAR（`javax:javaee-api:7.0`）。

### 3.3 WAR 打包约定
```groovy
war {
    archiveName 'altoromutual.war'
    exclude 'WEB-INF/classes/**/*.class'
    exclude 'WEB-INF/lib/**/*.jar'
}
```
WAR 输出名为 `altoromutual.war`，且显式排除已编译 class 和 lib 下的 jar——说明部署时 class 与 lib 可能由外部容器或 CI 单独注入，而非完全自包含。

### 3.4 版本管理
所有依赖均**硬编码具体版本号**，没有使用 BOM、平台属性或变量集中管理。例如 Jersey 三件套统一为 `2.27`，但 Commons Lang 存在 `commons-lang:2.5` 与 `com.hynnet:commons-lang3:3.3.2` 两个不同组名/版本的共存，体现历史演进痕迹。

## 4. 约定与约束

- **仓库白名单**：仅允许 `mavenCentral()`，未配置任何私有仓库、认证或代理（`build.gradle` 中 `repositories` 块仅一行 `mavenCentral()`）。
- **IDE 同步**：Eclipse 通过 Buildship 读取 Gradle 依赖；`.classpath` 中将 `org.eclipse.jst.j2ee.internal.web.container` 作为 Web 容器类路径，并标注 `org.eclipse.jst.component.dependency=/WEB-INF/lib`，使 IDE 识别该目录为组件依赖根。
- **JAR 存放位置**：若选择本地 jar 方式，必须放在 `WebContent/WEB-INF/lib/`，因为 `fileTree` 只扫描该目录。
- **Java 版本锁定**：所有依赖需兼容 Java 1.7（`sourceCompatibility/targetCompatibility = "1.7"`），这是硬性约束，升级 JDK 时需同步评估依赖兼容性。
- **API 与实现分离**：Servlet/JAX-RS 相关 API 以 `providedCompile` 形式声明，避免重复打包到 WAR，遵循 Java EE 容器的提供-消费模型。

## 5. 观察到的局限

- 无 `gradle.properties`、`settings.gradle`、`gradle/wrapper/` 等 Gradle Wrapper 或全局配置，无法保证构建环境一致性。
- 无依赖更新自动化脚本（如 Dependabot、Renovate、Depend-O-Matic）。
- 无锁文件（无 `*.lock`、`*.json.lock` 等），Maven Central 解析结果取决于网络可得的最新快照/发布版本（尽管此处全部指定了固定版本）。
- `WebContent/WEB-INF/lib/` 目前为空（仅有许可证文本），本地 jar 机制处于「可用但未使用」状态。