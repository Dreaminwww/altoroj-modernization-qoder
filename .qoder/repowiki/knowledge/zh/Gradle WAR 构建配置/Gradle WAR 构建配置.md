---
kind: build_system
name: Gradle WAR 构建配置
category: build_system
scope:
    - '**'
source_files:
    - build.gradle
    - .classpath
    - .project
    - Importing AltoroJ into Eclipse from GitHub.md
---

## 1. 使用的系统/工具

- **构建工具**: Gradle（`build.gradle`），使用内置的 `war` 插件打包为 WAR 归档。
- **JDK 版本**: source/target compatibility 锁定为 Java 1.7（`sourceCompatibility = "1.7"; targetCompatibility = "1.7"`）。
- **依赖来源**: Maven Central（`mavenCentral()`）以及本地 `WebContent/WEB-INF/lib/*.jar` 文件树。
- **IDE 支持**: 同时保留 Eclipse 工程文件 `.classpath`、`.project` 与 `.settings/`，用于在 Eclipse 中直接导入（参见 `Importing AltoroJ into Eclipse from GitHub.md`）。

## 2. 关键文件

- `build.gradle` — 唯一的构建脚本，定义源集、依赖与 WAR 打包行为。
- `WebContent/WEB-INF/web.xml` — Servlet/JAX-RS 部署描述符（由 WAR 产物携带）。
- `src/com/ibm/security/appscan/altoromutual/` — Java 源码根目录。
- `WebContent/` — JSP 与静态资源目录，作为 Web 应用内容被打包进 WAR。

## 3. 架构与约定

- **源集布局**：通过 `sourceSets.main.java.srcDirs = ['src']` 和 `resources.srcDirs = ['src']` 将 `src/` 同时作为 Java 源码和资源目录；Web 内容位于 `webAppDirName = 'WebContent'`。
- **依赖分类**：运行期依赖使用 `compile`（如 Jersey 2.27、Derby 10.8.2.2、Log4j 1.2.16、Wink JSON-Java 1.4、commons-codec/lang 等）；Servlet API 使用 `providedCompile`（`javax:javaee-api:7.0`），表明 WAR 期望在外部容器（Tomcat 等）中运行，容器提供 EE API。
- **本地库**：`fileTree(dir: "${webAppDirName}/WEB-INF/lib", include: '*.jar')` 将所有放在 `WebContent/WEB-INF/lib/` 下的 jar 加入 classpath，这是该仓库内第三方二进制的主要承载位置。
- **WAR 输出**：`war.archiveName = 'altoromutual.war'`，并显式 `exclude 'WEB-INF/classes/**/*.class'` 与 `exclude 'WEB-INF/lib/**/*.jar'`，意味着默认 Gradle war 任务会排除已编译类与 lib 下 jar——实际部署时这些内容需由外部步骤或 IDE 以其他方式注入到容器中。

## 4. 约定与约束

- **Java 版本约束**：构建脚本强制 source/target 为 1.7，任何高于 1.7 的语法都会导致编译失败。
- **依赖版本固定**：所有第三方依赖均指定精确版本号（Jersey 2.27、Derby 10.8.2.2、Log4j 1.2.16 等），不存在版本范围或 BOM 管理。
- **EE API 由容器提供**：`providedCompile 'javax:javaee-api:7.0'` 表明 WAR 不嵌入 Servlet/JAX-RS 运行时，而是依赖外部 EE 容器。
- **无 CI/Dockerfile/Makefile**：仓库根目录未发现 `.github/workflows`、`Dockerfile`、`Makefile` 或 `*.sh` 构建脚本；构建入口仅为 `gradle` 命令。
- **Eclipse 优先导入**：README 旁的 `Importing AltoroJ into Eclipse from GitHub.md` 文档说明该项目主要面向 Eclipse 用户，Gradle 更多是辅助打包用途。