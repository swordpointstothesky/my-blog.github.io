---
title: 从零构建通用后台管理系统（二）：用户管理 CRUD 完整实现
date: 2026-09-01
tags: [.NET, React, CRUD, 通用后台, 全栈开发]
categories: [项目实战]
description: 从分层架构设计到前后端联调，完整记录用户管理增删改查功能的实现过程。
---

## 前言

在上一篇博客中，我们完成了项目搭建、登录功能实现，并接入了 PostgreSQL 数据库。现在系统已经有了“入口”——用户可以通过登录页进入后台。

但登录之后，后台还是空的。任何一个后台管理系统，最核心、最基础的功能就是**用户管理**（增删改查，即 CRUD）。这个功能不仅本身重要，它还是未来所有业务模块的“模板”——学会了怎么做用户管理，就学会了怎么做商品管理、订单管理、内容管理等。

本文记录了我在通用后台系统中实现用户管理 CRUD 的完整过程，包括：

- 后端分层架构设计（Controller → Service → DbContext）
- DTO（数据传输对象）的分离
- 前端 Shadcn Table + Dialog 实战
- 前后端联调与测试

---

## 一、后端分层架构：为什么要分层？

在开始写代码之前，我先做了一个重要的决策：**采用分层架构**，而不是把所有逻辑都写在 Controller 里。

### 1.1 分层结构

backend/
├── Controllers/ # 表现层：接收请求，返回响应
├── Services/ # 业务层：核心业务逻辑
├── Data/ # 数据层：DbContext
├── Models/ # 实体：数据库映射
├── DTOs/ # 数据传输对象：API 请求/响应
└── Program.cs # 服务注册

### 1.2 各层职责

| 层级 | 职责 | 示例 |
| :--- | :--- | :--- |
| **Controller** | 接收请求、参数验证、调用 Service、返回响应 | `UsersController.GetUsers()` |
| **Service** | 业务逻辑、事务处理、调用 DbContext | `UserService.GetUsersAsync()` |
| **DbContext** | 数据访问（EF Core 已内置） | `_context.Users.Where(...)` |
| **DTO** | 定义 API 的输入/输出格式 | `UserDto`, `CreateUserRequest` |

### 1.3 为什么不用 Repository 层？

EF Core 的 `DbContext` 本身已经是仓储模式（Repository Pattern）+ 工作单元模式（Unit of Work）的实现。对于现阶段的项目，直接使用 `DbContext` 已经足够，不需要再额外封装一层。

> 如果未来需要切换数据源（比如从 PostgreSQL 换到 MongoDB），或者需要单元测试时 Mock 数据层，再考虑引入 Repository 层。**过早优化是效率的敌人。**

---

## 二、DTO 设计：API 的“合同”

DTO（Data Transfer Object，数据传输对象）是前后端之间的“合同”——它定义了 API 请求和响应的数据结构。

**为什么需要 DTO？**

- **安全性**：不要把 `User` 实体（包含 `PasswordHash`）直接暴露给前端
- **灵活性**：API 响应可以只包含前端需要的字段，不包含内部字段
- **版本控制**：DTO 变化不影响实体模型

### 2.1 UserDto（响应）

```csharp
public class UserDto
{
    public int Id { get; set; }
    public string Username { get; set; } = string.Empty;
    public string? Email { get; set; }
    public DateTime CreateTime { get; set; }
    public bool IsActive { get; set; }
}
```

### 2.2 CreateUserRequest（新增请求）

```csharp
public class CreateUserRequest
{
    public string Username { get; set; } = string.Empty;
    public string Password { get; set; } = string.Empty;
    public string? Email { get; set; }
    public bool? IsActive { get; set; }
}
```

### 2.3 UpdateUserRequest（更新请求）

```csharp
public class UpdateUserRequest
{
    public string? Username { get; set; }
    public string? Password { get; set; }
    public string? Email { get; set; }
    public bool? IsActive { get; set; }
}
```

注意：更新请求中的属性都是可空的。前端如果只传 { email: "new@example.com" }，后端只更新邮箱，其他字段保持不变。

---

## 三、Service 层：业务逻辑的核心

### 3.1 接口定义

```csharp
public interface IUserService
{
    Task<List<UserDto>> GetUsersAsync();
    Task<UserDto?> GetUserByIdAsync(int id);
    Task<UserDto> CreateUserAsync(CreateUserRequest request);
    Task<bool> UpdateUserAsync(int id, UpdateUserRequest request);
    Task<bool> DeleteUserAsync(int id);
}
```

### 3.2 实现要点

#### 3.2.1 密码哈希（BCrypt）

用户密码绝不能明文存储。我使用 BCrypt.Net-Next 进行密码哈希：

```csharp
// 注册时：哈希密码
user.PasswordHash = BCrypt.HashPassword(request.Password);

// 登录时：验证密码（在 AuthService 中）
bool isValid = BCrypt.Verify(password, user.PasswordHash);
```

BCrypt 是行业标准的密码哈希算法，自带盐值（Salt），可以有效抵抗彩虹表攻击。

#### 3.2.2 用户名唯一性校验

```csharp
if (await _context.Users.AnyAsync(u => u.Username == request.Username))
    throw new InvalidOperationException("用户名已存在");
```

#### 3.2.3 更新时的部分字段更新

```csharp
if (!string.IsNullOrEmpty(request.Username))
    user.Username = request.Username;

if (!string.IsNullOrEmpty(request.Password))
    user.PasswordHash = BCrypt.HashPassword(request.Password);

if (!string.IsNullOrEmpty(request.Email))
    user.Email = request.Email;

if (request.IsActive.HasValue)
    user.IsActive = request.IsActive.Value;
```

---

## 四、Controller 层：RESTful API 端点

```csharp
[Authorize]
[ApiController]
[Route("api/[controller]")]
public class UsersController : ControllerBase
{
    private readonly IUserService _userService;

    public UsersController(IUserService userService)
    {
        _userService = userService;
    }

    [HttpGet]
    public async Task<IActionResult> GetUsers()
    {
        var users = await _userService.GetUsersAsync();
        return Ok(users);
    }

    [HttpGet("{id}")]
    public async Task<IActionResult> GetUser(int id)
    {
        var user = await _userService.GetUserByIdAsync(id);
        if (user == null) return NotFound();
        return Ok(user);
    }

    [HttpPost]
    public async Task<IActionResult> CreateUser([FromBody] CreateUserRequest request)
    {
        try
        {
            var user = await _userService.CreateUserAsync(request);
            return CreatedAtAction(nameof(GetUser), new { id = user.Id }, user);
        }
        catch (InvalidOperationException ex)
        {
            return BadRequest(new { message = ex.Message });
        }
    }

    [HttpPut("{id}")]
    public async Task<IActionResult> UpdateUser(int id, [FromBody] UpdateUserRequest request)
    {
        var result = await _userService.UpdateUserAsync(id, request);
        if (!result) return NotFound();
        return NoContent();
    }

    [HttpDelete("{id}")]
    public async Task<IActionResult> DeleteUser(int id)
    {
        var result = await _userService.DeleteUserAsync(id);
        if (!result) return NotFound();
        return NoContent();
    }
}
```

### RESTful API 设计规范

| 方法 | 路径 | 说明 |
| ------- | ------- | ------- |
| GET | /api/users | 获取用户列表 |
| GET | /api/users/{id} | 获取单个用户 |
| POST | /api/users | 创建用户 |
| PUT | /api/users/{id} | 更新用户 |
| DELETE | /api/users/{id} | 更新用户 |

| 用 [Authorize] 保护所有端点，确保只有登录用户才能访问。

## 五、前端实现：Shadcn Table + Dialog

### 5.1 技术选型

| 功能 | 使用的组件 |
| ------- | ------- |
| 表格展示 | Shadcn Table |
| 新增/编辑弹窗 | Shadcn Dialog |
| 删除确认弹窗 | Shadcn AlertDialog |
| HTTP 请求 | Axios |
| 图标 | Lucide React |

### 5.2 安装依赖

```bash
npx shadcn@latest add table dialog alert-dialog input label button
pnpm add lucide-react
```

### 5.3 核心功能点

#### 5.3.1 获取用户列表

```tsx
const fetchUsers = async () => {
    const token = localStorage.getItem('token');
    const response = await axios.get('/api/users', {
        headers: { Authorization: `Bearer ${token}` },
    });
    setUsers(response.data);
};
```

#### 5.3.2 新增用户（POST 请求）

```tsx
const handleSave = async () => {
    const token = localStorage.getItem('token');
    await axios.post('/api/users', formData, {
        headers: { Authorization: `Bearer ${token}` },
    });
    fetchUsers(); // 刷新列表
};
```

#### 5.3.3 编辑用户（PUT 请求）

编辑时，密码字段留空表示不修改——这是对用户体验的优化，用户不必每次修改资料都重新输入密码。

```tsx
if (editingUser) {
    const updateData = {
        username: formData.username,
        email: formData.email,
        isActive: formData.isActive,
    };
    if (formData.password) {
        updateData.password = formData.password;
    }
    await axios.put(`/api/users/${editingUser.id}`, updateData);
}
```

#### 5.3.4 删除用户（带确认弹窗）

删除操作不可逆，必须让用户二次确认：

```tsx
<AlertDialog open={deleteDialogOpen}>
    <AlertDialogContent>
        <AlertDialogTitle>确认删除</AlertDialogTitle>
        <AlertDialogDescription>
            确定要删除用户 {deletingUser?.username} 吗？此操作不可撤销。
        </AlertDialogDescription>
        <AlertDialogFooter>
            <AlertDialogCancel>取消</AlertDialogCancel>
            <AlertDialogAction onClick={handleConfirmDelete}>
                确认删除
            </AlertDialogAction>
        </AlertDialogFooter>
    </AlertDialogContent>
</AlertDialog>
```

### 5.4 效果展示

## 六、踩坑记录

### 坑 1：CORS 跨域问题

前端 localhost:5173 调用后端 localhost:5001 时，浏览器会拦截跨域请求。

解决方案：在 Program.cs 中配置 CORS：

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin().AllowAnyMethod().AllowAnyHeader();
    });
});
app.UseCors("AllowAll");
```

| 生产环境建议限制具体域名，不要用 AllowAnyOrigin。

### 坑 2：JWT Token 过期导致 401

前端登录后 Token 存储在 localStorage，但 Token 过期后请求会返回 401。

解决方案：在 Axios 响应拦截器中处理 Token 过期：

```tsx
axios.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 401) {
            localStorage.removeItem('token');
            window.location.href = '/login';
        }
        return Promise.reject(error);
    }
);
```

### 接口返回 IAsyncEnumerable 导致序列化失败

在 Service 中直接返回 IQueryable 可能导致 EF Core 的延迟加载在 Controller 序列化时执行，造成意外行为。

解决方案：在 Service 中始终使用 ToListAsync() 将数据加载到内存再返回：

```csharp
public async Task<List<UserDto>> GetUsersAsync()
{
    return await _context.Users
        .Select(u => new UserDto { ... })
        .ToListAsync(); // 必须 ToList，不能直接返回 IQueryable
}
```