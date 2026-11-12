---
kind: error_handling
name: AltoroJ 错误处理：Servlet/JSP 内联 try-catch、web.xml 全局 404 与 JAX-RS 局部响应式错误
category: error_handling
scope:
    - '**'
source_files:
    - WebContent/WEB-INF/web.xml
    - WebContent/notfound.jsp
    - src/com/ibm/security/appscan/altoromutual/servlet/LoginServlet.java
    - src/com/ibm/security/appscan/altoromutual/filter/AuthFilter.java
    - src/com/ibm/security/appscan/altoromutual/util/DBUtil.java
    - src/com/ibm/security/appscan/altoromutual/api/LoginAPI.java
    - src/com/ibm/security/appscan/altoromutual/api/FeedbackAPI.java
    - src/com/ibm/security/appscan/altoromutual/api/LogoutAPI.java
    - WebContent/login.jsp
---

## 1. 总体方式

AltoroJ 是一个基于 Servlet 3.0 / JSP + Jersey JAX-RS 的演示银行应用，**没有统一的异常类型体系、没有自定义 `ExceptionMapper`、也没有 panic/recover 机制**。错误处理以“就近 try-catch + 返回用户可见字符串/HTTP 状态”为主，辅以 `web.xml` 中唯一的 `<error-page>` 全局 404 映射。

- **Servlet 层**：在 `doGet/doPost` 中直接 try-catch `Exception`，把错误信息放入 session 属性（如 `loginError`）后重定向到登录页，或直接 `response.sendError(500)`。
- **JAX-RS API 层**：每个 `@Path` 方法内部 try-catch `JSONException` / `Exception`，通过 `Response.status(...).entity("{...}")` 返回 JSON 错误体；未捕获异常由 Jersey 默认处理。
- **Filter 层**：`AuthFilter` / `AdminFilter` 不捕获业务异常，而是通过 `sendRedirect` 将未认证请求转到 `/login.jsp`，属于“鉴权失败即错误分支”。
- **JSP 层**：使用 `page="notfound.jsp" errorPage="notfound.jsp"` 声明式错误页，并通过 `<%@page isErrorPage="true" %>` 访问内置 `exception` 对象。
- **工具层**：`DBUtil` 等 DAO 类对 `SQLException` 采取三种策略之一：记录日志并返回空集合/null、返回人类可读字符串（如 `"Transaction failed. Please try again later."`）、或向上抛出 `SQLException` 让调用方决定。