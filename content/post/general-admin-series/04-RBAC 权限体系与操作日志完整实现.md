---
title: 从零构建通用后台管理系统（四）：RBAC 权限体系与操作日志完整实现
date: 2026-09-14
tags: [.NET, React, RBAC, 权限管理, 操作日志, 全栈开发]
categories: [项目实战]
description: 记录通用后台管理系统 RBAC 权限体系的完整实现，包含角色管理、动态菜单、按钮级权限、操作日志和个人中心。
---

## 前言

在前三篇文章中，我们完成了：

- **第一篇**：项目搭建、前后端联调、登录功能
- **第二篇**：用户管理 CRUD 完整实现
- **第三篇**：部署上线（Vercel + Railway + Supabase）

但一个真正的「通用后台管理系统」，仅有登录和用户管理是不够的。**权限控制**才是后台系统的灵魂——没有权限控制，所有登录用户都能做任何操作，这在生产环境中是不可接受的。

本文记录了 RBAC 权限体系的完整实现过程，包括：

- 数据库表设计（用户、角色、权限、菜单）
- 后端权限验证（JWT 中注入角色和权限）
- 动态菜单（侧边栏从数据库读取）
- 前端按钮级权限控制
- 操作日志（审计追踪）
- 个人中心

---

## 一、RBAC 设计思路

RBAC（Role-Based Access Control，基于角色的访问控制）是业界最主流的权限模型。核心思想是：**不直接给用户分配权限，而是给用户分配角色，角色再关联权限。**

### 1.1 核心模型

```text
用户 (User) ──多对多── 角色 (Role) ──多对多── 权限 (Permission)
                              │
                              └──多对多── 菜单 (Menu)
```

### 1.2 为什么需要菜单表？

权限和菜单是两回事：

| 概念 | 作用 | 示例 |
| :--- | :--- | :--- |
| **权限** | 控制 API 访问 | `user:create`（能否创建用户） |
| **菜单** | 控制侧边栏显示 | `/users`（侧边栏是否显示"用户管理"） |

一个用户可能**有权限访问某个页面，但菜单不显示**（比如通过按钮跳转），也可能**菜单显示但无权限操作**（只读模式）。所以需要分开管理。

### 1.3 数据库表结构

```sql
-- 角色表
CREATE TABLE "Roles" (
    "Id" SERIAL PRIMARY KEY,
    "Name" VARCHAR(50) NOT NULL UNIQUE,
    "Description" VARCHAR(200),
    "CreateTime" TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
    "IsActive" BOOLEAN DEFAULT TRUE
);

-- 权限表
CREATE TABLE "Permissions" (
    "Id" SERIAL PRIMARY KEY,
    "Name" VARCHAR(100) NOT NULL UNIQUE,
    "DisplayName" VARCHAR(100) NOT NULL,
    "Category" VARCHAR(50),
    "CreateTime" TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- 菜单表
CREATE TABLE "Menus" (
    "Id" SERIAL PRIMARY KEY,
    "ParentId" INTEGER REFERENCES "Menus"("Id"),
    "Name" VARCHAR(50) NOT NULL,
    "Path" VARCHAR(100) NOT NULL,
    "Icon" VARCHAR(50),
    "SortOrder" INTEGER DEFAULT 0,
    "IsActive" BOOLEAN DEFAULT TRUE
);

-- 关联表
CREATE TABLE "UserRoles" (
    "UserId" INTEGER REFERENCES "Users"("Id") ON DELETE CASCADE,
    "RoleId" INTEGER REFERENCES "Roles"("Id") ON DELETE CASCADE,
    PRIMARY KEY ("UserId", "RoleId")
);

CREATE TABLE "RolePermissions" (
    "RoleId" INTEGER REFERENCES "Roles"("Id") ON DELETE CASCADE,
    "PermissionId" INTEGER REFERENCES "Permissions"("Id") ON DELETE CASCADE,
    PRIMARY KEY ("RoleId", "PermissionId")
);

CREATE TABLE "RoleMenus" (
    "RoleId" INTEGER REFERENCES "Roles"("Id") ON DELETE CASCADE,
    "MenuId" INTEGER REFERENCES "Menus"("Id") ON DELETE CASCADE,
    PRIMARY KEY ("RoleId", "MenuId")
);
```

---

## 二、后端权限验证

### 2.1 JWT 中注入角色和权限

登录成功后，需要把用户的角色和权限写入 JWT Token，前端才能据此判断。

```csharp
private string GenerateJwtToken(User user)
{
    var roles = _context.UserRoles
        .Where(ur => ur.UserId == user.Id)
        .Select(ur => ur.Role.Name)
        .ToList();

    var claims = new List<Claim>
    {
        new Claim(ClaimTypes.Name, user.Username),
        new Claim(ClaimTypes.NameIdentifier, user.Id.ToString())
    };

    foreach (var role in roles)
    {
        claims.Add(new Claim(ClaimTypes.Role, role));
    }

    var permissions = _context.UserRoles
        .Where(ur => ur.UserId == user.Id)
        .SelectMany(ur => ur.Role.RolePermissions)
        .Select(rp => rp.Permission.Name)
        .Distinct()
        .ToList();

    foreach (var perm in permissions)
    {
        claims.Add(new Claim("Permission", perm));
    }

    // 生成 Token...
}
```

### 2.2 API 级别的权限保护

在 Controller 上使用 `[Authorize(Roles = "Admin")]` 保护接口：

```csharp
[Authorize(Roles = "Admin")]
[ApiController]
[Route("api/[controller]")]
public class LogsController : ControllerBase
{
    // ...
}
```

对于更细粒度的权限（如 `user:create`），可以实现自定义权限验证：

```csharp
public class PermissionRequirement : IAuthorizationRequirement
{
    public string Permission { get; }
    public PermissionRequirement(string permission) => Permission = permission;
}

public class PermissionHandler : AuthorizationHandler<PermissionRequirement>
{
    protected override Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        PermissionRequirement requirement)
    {
        var userPermissions = context.User.Claims
            .Where(c => c.Type == "Permission")
            .Select(c => c.Value)
            .ToList();

        if (userPermissions.Contains(requirement.Permission))
            context.Succeed(requirement);

        return Task.CompletedTask;
    }
}
```

---

## 三、动态菜单

### 3.1 后端：根据用户角色返回菜单树

```csharp
public async Task<List<MenuDto>> GetMenusByUserAsync(int userId)
{
    var roleIds = await _context.UserRoles
        .Where(ur => ur.UserId == userId)
        .Select(ur => ur.RoleId)
        .ToListAsync();

    var menuIds = await _context.RoleMenus
        .Where(rm => roleIds.Contains(rm.RoleId))
        .Select(rm => rm.MenuId)
        .Distinct()
        .ToListAsync();

    var menus = await _context.Menus
        .Where(m => menuIds.Contains(m.Id) && m.IsActive)
        .OrderBy(m => m.SortOrder)
        .ToListAsync();

    return BuildMenuTree(menus, null);
}

private List<MenuDto> BuildMenuTree(List<Menu> menus, int? parentId)
{
    return menus
        .Where(m => m.ParentId == parentId)
        .Select(m => new MenuDto
        {
            Id = m.Id,
            Name = m.Name,
            Path = m.Path,
            Icon = m.Icon,
            Children = BuildMenuTree(menus, m.Id)
        })
        .ToList();
}
```

### 3.2 前端：动态渲染侧边栏

使用 shadcn/ui 的 Sidebar 组件：

```tsx
export default function Layout() {
    const [menus, setMenus] = useState<MenuItem[]>([]);

    useEffect(() => {
        const fetchMenus = async () => {
            const response = await api.get('/api/menus/my');
            setMenus(response.data);
        };
        fetchMenus();
    }, []);

    return (
        <SidebarProvider>
            <Sidebar>
                <SidebarContent>
                    <SidebarGroup>
                        <SidebarMenu>
                            {menus.map((item) => {
                                const Icon = iconMap[item.icon];
                                const isActive = location.pathname === item.path;
                                return (
                                    <SidebarMenuItem key={item.id}>
                                        <SidebarMenuButton asChild isActive={isActive}>
                                            <Link to={item.path}>
                                                {Icon && <Icon />}
                                                <span>{item.name}</span>
                                            </Link>
                                        </SidebarMenuButton>
                                    </SidebarMenuItem>
                                );
                            })}
                        </SidebarMenu>
                    </SidebarGroup>
                </SidebarContent>
            </Sidebar>
            <SidebarInset>
                <Outlet />
            </SidebarInset>
        </SidebarProvider>
    );
}
```

---

## 四、前端按钮级权限控制

### 4.1 权限上下文

创建一个全局的 PermissionContext，从 JWT 中解析权限：

```tsx
export function PermissionProvider({ children }: { children: ReactNode }) {
    const [permissions, setPermissions] = useState<string[]>([]);

    useEffect(() => {
        const token = localStorage.getItem('token');
        if (token) {
            const payload = JSON.parse(atob(token.split('.')[1]));
            const perms = payload.Permission;
            setPermissions(Array.isArray(perms) ? perms : [perms]);
        }
    }, []);

    const hasPermission = (permission: string) => permissions.includes(permission);

    return (
        <PermissionContext.Provider value={{ permissions, hasPermission }}>
            {children}
        </PermissionContext.Provider>
    );
}
```

### 4.2 在组件中使用

```tsx
const { hasPermission } = usePermission();

return (
    <div>
        {hasPermission('user:create') && (
            <Button onClick={handleCreate}>新增用户</Button>
        )}
        {hasPermission('user:delete') && (
            <Button variant="ghost" onClick={handleDelete}>
                <Trash2 />
            </Button>
        )}
    </div>
);
```

---

## 五、操作日志（审计追踪）

### 5.1 自动记录所有 API 请求

使用 `IAsyncActionFilter` 全局拦截所有请求，自动记录日志：

```csharp
public class OperationLogFilter : IAsyncActionFilter
{
    public async Task OnActionExecutionAsync(
        ActionExecutingContext context, ActionExecutionDelegate next)
    {
        var stopwatch = Stopwatch.StartNew();

        var request = context.HttpContext.Request;
        var user = context.HttpContext.User;
        var userId = user.FindFirst(ClaimTypes.NameIdentifier)?.Value;
        var username = user.FindFirst(ClaimTypes.Name)?.Value;

        var result = await next();
        stopwatch.Stop();

        var log = new OperationLog
        {
            UserId = int.TryParse(userId, out var id) ? id : null,
            Username = username ?? "Unknown",
            Action = ExtractAction(request.Method),
            Module = ExtractModule(request.Path),
            Method = request.Method,
            Path = request.Path,
            ClientIP = context.HttpContext.Connection.RemoteIpAddress?.ToString(),
            ExecutionTime = (int)stopwatch.ElapsedMilliseconds,
            CreateTime = DateTime.UtcNow
        };

        await _logService.LogAsync(log);
    }
}
```

### 5.2 在 Program.cs 中全局注册

```csharp
builder.Services.AddControllers(options =>
{
    options.Filters.Add<OperationLogFilter>();
});
```

这样每个请求都会自动记录日志，无需在每个 Action 中手动调用。

---

## 六、个人中心

个人中心是用户使用频率最高的功能。提供两个核心能力：

### 6.1 修改邮箱

```csharp
[HttpPut]
public async Task<IActionResult> UpdateProfile([FromBody] UpdateProfileRequest request)
{
    var userId = GetCurrentUserId();
    await _profileService.UpdateProfileAsync(userId, request);
    return NoContent();
}
```

### 6.2 修改密码

```csharp
[HttpPut("password")]
public async Task<IActionResult> ChangePassword([FromBody] ChangePasswordRequest request)
{
    try
    {
        var userId = GetCurrentUserId();
        await _profileService.ChangePasswordAsync(userId, request);
        return NoContent();
    }
    catch (InvalidOperationException ex)
    {
        return BadRequest(new { message = ex.Message });
    }
}
```

**关键点**：修改密码时必须验证旧密码，修改成功后强制重新登录（Token 需要刷新）。

---

## 七、踩坑记录

### 坑 1：依赖注入遗漏

**报错**：`Unable to resolve service for type 'IRoleService'`

**原因**：新增的 `RoleService` 没有在 `Program.cs` 中注册。

**解决**：每次新增 Service 后，记得注册：

```csharp
builder.Services.AddScoped<IRoleService, RoleService>();
```

> 后续可以使用 `Scrutor` 批量注册，避免遗漏。

### 坑 2：[Required] 特性覆盖可空类型

**现象**：`public int? UserId` 已经改为可空，但生成的数据库列仍是 `NOT NULL`。

**原因**：`[Required]` 特性优先于类型可空性。

**解决**：确保既使用可空类型，又移除 `[Required]`：

```csharp
// ✅ 正确
public int? UserId { get; set; }

// ❌ 错误：即使类型可空，[Required] 也会强制 NOT NULL
[Required]
public int? UserId { get; set; }
```

### 坑 3：权限名称格式不统一

**现象**：菜单不显示，但用户确实有权限。

**原因**：数据库里的权限名是 `user.view`，而代码里检查的是 `user:view`。

**解决**：统一使用冒号 `:` 作为权限名称分隔符，并保持代码和数据库一致。

### 坑 4：EF Core 空迁移

**现象**：`dotnet ef migrations add` 生成的迁移文件 `Up()` 方法是空的。

**原因**：`ModelSnapshot` 中已包含该实体，EF Core 认为没有变化。

**解决**：如果数据库和实体不同步，最干净的方式是重置迁移：

```bash
# 1. 清空数据库
DROP SCHEMA public CASCADE;
CREATE SCHEMA public;

# 2. 删除所有迁移文件
# 3. 重新生成
dotnet ef migrations add InitialCreate
dotnet ef database update
```

---

## 八、成果展示

完成 RBAC 后，系统现在具备：

| 功能 | 状态 |
| :--- | :--- |
| 用户管理（增删改查 + 分配角色） | ✅ |
| 角色管理（增删改查 + 分配权限 + 分配菜单） | ✅ |
| 动态菜单（按角色显示） | ✅ |
| 前端按钮级权限控制 | ✅ |
| 后端 API 权限验证 | ✅ |
| 操作日志（自动记录 + 查询） | ✅ |
| 个人中心（修改邮箱 / 修改密码） | ✅ |

### 角色编辑界面

![角色编辑界面](/images/role-edit.png)

### 操作日志界面

![操作日志界面](/images/operation-logs.png)

---

## 九、下一步计划

- 实现代码生成器（根据数据库表自动生成 CRUD 代码）
- 系统设置（系统名称、Logo 等配置）
- 数据字典（管理下拉框选项）

---

## 后记

RBAC 权限体系是后台管理系统的核心。完成这一部分后，你的系统就从一个「能用的 Demo」升级为「可以交付给客户的正式产品」。

这套权限体系的设计思路，可以直接复用到几乎所有的后台项目中。当你接私活时，只需要根据客户需求调整角色和权限，不需要重新设计架构。

**项目代码**：[GitHub 仓库链接](https://github.com/yourname/general-admin)

---

*本文首发于我的个人博客，记录了通用后台管理系统 RBAC 权限体系的完整实现过程。*