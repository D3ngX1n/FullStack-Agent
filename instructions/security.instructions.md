---
description: "Use when: 编写任何涉及用户输入、数据库查询、HTML 渲染、权限控制、认证授权、文件操作、API 调用的代码。覆盖 SQL 注入、XSS、越权、CSRF、路径穿越、命令注入等 OWASP Top 10 安全风险防护。"
applyTo: ["**/*.ts", "**/*.tsx", "**/*.js", "**/*.jsx", "**/*.java", "**/*.py", "**/*.go", "**/*.vue", "**/*.svelte"]
---

# 安全开发规范

## 核心原则

**所有外部输入都是不可信的**。用户输入、URL 参数、请求头、Cookie、文件上传、第三方 API 响应——在使用前必须验证和净化。

---

## 1. SQL 注入防护

### 强制要求
- **绝对禁止**拼接用户输入到 SQL 语句中
- **必须**使用参数化查询 / 预编译语句 / ORM 的安全查询方法

### 正确做法（按技术栈）

**Java / Spring Boot**:
```java
// ✅ JPA 参数化
@Query("SELECT u FROM User u WHERE u.email = :email")
User findByEmail(@Param("email") String email);

// ✅ MyBatis 用 #{} 不用 ${}
<select id="findUser">
  SELECT * FROM users WHERE id = #{id}
</select>

// ❌ 绝对禁止
String sql = "SELECT * FROM users WHERE id = " + userId;
```

**Node.js**:
```typescript
// ✅ Prisma — 默认参数化
await prisma.user.findFirst({ where: { email } });

// ✅ 原生查询参数化
await db.query('SELECT * FROM users WHERE id = $1', [userId]);

// ❌ 绝对禁止
await db.query(`SELECT * FROM users WHERE id = ${userId}`);
```

**Python**:
```python
# ✅ SQLAlchemy ORM
User.query.filter_by(email=email).first()

# ✅ 原生参数化
cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))

# ❌ 绝对禁止
cursor.execute(f"SELECT * FROM users WHERE id = {user_id}")
```

**Go**:
```go
// ✅ 参数化
db.QueryRow("SELECT * FROM users WHERE id = ?", userID)

// ❌ 绝对禁止
db.QueryRow(fmt.Sprintf("SELECT * FROM users WHERE id = %s", userID))
```

### 动态查询场景
- 排序字段：使用白名单校验，不接受任意字段名
- LIKE 查询：转义 `%` 和 `_` 特殊字符
- IN 查询：使用 ORM 提供的 `IN` 方法，不手动拼接

---

## 2. XSS（跨站脚本）防护

### 强制要求
- **绝对禁止**将未转义的用户输入插入 HTML
- **绝对禁止**使用 `dangerouslySetInnerHTML` / `v-html` / `{@html}` 渲染用户内容（除非经过严格净化）

### 正确做法

**输出编码**（框架默认行为，确保不绕过）：
```tsx
// ✅ React — 默认转义，安全
<p>{userInput}</p>

// ❌ 绕过了 React 的转义
<div dangerouslySetInnerHTML={{ __html: userInput }} />
```

```vue
<!-- ✅ Vue — 默认转义 -->
<p>{{ userInput }}</p>

<!-- ❌ 绕过了 Vue 的转义 -->
<div v-html="userInput"></div>
```

**必须净化的场景**（富文本编辑器输出等）：
```typescript
// 使用 DOMPurify 或同等库
import DOMPurify from 'dompurify';
const clean = DOMPurify.sanitize(userHtml);
```

**HTTP 响应头**（后端必须设置）：
```
Content-Security-Policy: default-src 'self'; script-src 'self'
X-Content-Type-Options: nosniff
X-Frame-Options: DENY
```

### URL 处理
```typescript
// ✅ 校验 URL scheme
const url = new URL(userUrl);
if (!['http:', 'https:'].includes(url.protocol)) throw new Error('非法 URL');

// ❌ 直接使用用户 URL（可能是 javascript: 协议）
<a href={userUrl}>链接</a>
```

---

## 3. 越权（Broken Access Control）防护

### 强制要求
- **每个 API 端点**必须校验当前用户是否有权访问目标资源
- **绝对禁止**仅靠前端隐藏按钮/菜单来控制权限——后端必须独立校验
- **绝对禁止**使用可预测 ID 作为唯一鉴权手段

### 正确做法

**资源级权限校验**（IDOR 防护）：
```typescript
// ✅ 校验资源归属
async getOrder(orderId: string, currentUserId: string) {
  const order = await this.orderRepo.findById(orderId);
  if (order.userId !== currentUserId) throw new ForbiddenException();
  return order;
}

// ❌ 只根据 ID 查询，不校验归属
async getOrder(orderId: string) {
  return this.orderRepo.findById(orderId);
}
```

```java
// ✅ Spring Security + 资源归属校验
@PreAuthorize("hasRole('USER')")
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id, @AuthenticationPrincipal User user) {
    Order order = orderService.findById(id);
    if (!order.getUserId().equals(user.getId())) {
        throw new AccessDeniedException("无权访问");
    }
    return order;
}
```

### 检查清单
- [ ] 每个 API 端点都有认证检查（中间件/注解/装饰器）
- [ ] 资源访问都有归属校验（不只是角色检查）
- [ ] 批量操作也逐条校验权限
- [ ] 管理接口与普通接口严格隔离

---

## 4. 其他关键安全项

### CSRF 防护
- 表单提交必须携带 CSRF Token（框架通常内置支持）
- API 使用 SameSite Cookie 属性 + 自定义请求头双重防护
- Spring Boot: 不要随意 `csrf().disable()`，仅 stateless API 可禁用

### 路径穿越防护
```typescript
// ✅ 校验路径不越界
const safePath = path.resolve(baseDir, userFilename);
if (!safePath.startsWith(baseDir)) throw new Error('非法路径');
```

### 命令注入防护
```typescript
// ✅ 使用数组参数，不拼接命令
execFile('ls', ['-la', userDir]);  // 安全

// ❌ 绝对禁止
exec(`ls -la ${userDir}`);  // 用户可注入 ; rm -rf /
```

### 敏感数据处理
- 密码必须使用 bcrypt/argon2 哈希存储，绝不明文
- API 密钥、数据库凭证不硬编码，使用环境变量或密钥管理
- 日志中不打印密码、Token、信用卡号等敏感信息
- 响应中不返回不必要的用户敏感字段（密码哈希、内部 ID）

### 依赖安全
- 定期运行 `npm audit` / `pip audit` / `mvn dependency-check:check`
- 不使用已知存在漏洞的依赖版本
- 锁定依赖版本（lock 文件必须提交）

---

## 速查表：输入验证模式

| 场景 | 验证方式 |
|------|---------|
| 用户 ID / 资源 ID | UUID 格式校验 + 归属校验 |
| 邮箱 | 格式校验 + 长度限制 |
| 分页参数 | 数值范围限制（page ≥ 1, size ≤ 100） |
| 排序字段 | 白名单校验 |
| 文件上传 | 类型白名单 + 大小限制 + 重命名存储 |
| URL | scheme 白名单（http/https） |
| 富文本 | DOMPurify 净化 |
| 搜索关键词 | 长度限制 + SQL/NoSQL 特殊字符转义 |
