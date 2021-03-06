## 1. Authorize

Authorize用于授权验证，但是并不指定用户应该如何认证。这样有利于Identity与其他Asp.Net应用框架无缝对接。

## 2. HttpContext

Asp.Net 平台提供上下文信息HttpContext,用于授权验证：

* HttpContext.User实现IPrincial接口

> Identity属性：实现Identity接口，描述请求用户信息
>
> IsInRole:用户角色判断

* IIdentity接口提供当前用户的基础信息

> AuthenticationType :用户认证机制
>
> IsAuthenticated:用户是否认证
>
> Name:当前用户名

.Net Core使用浏览器发送的Cookie信息判断用户是否已经被认证。如果用户被认证，IsAuthentiacted总是为true，否则为false，并默认重定向到/Account/Login。浏览器请求Account/Login，如果应用程序没有提供，则返回404。可以改变默认的重定向URL：

```
//Identity系统不依赖路由系统，因此这里必须写硬编码地址
services.AddIdentity<AppUser, IdentityRole>(opts => {
    opts.Cookies.ApplicationCookie.LoginPath = "/Users/Login";
})
```

## 3. AccountController实现

* ReturnURL:用户请求被限制访问的地址时，会被重定向到登录地址，登录后需要跳转到之前的地址。
* ValidateAntiForgeryToken:跨站脚本攻击

## 4. 用户认证

```
[HttpPost]
[AllowAnonymous]
[ValidateAntiForgeryToken]
public async Task<IActionResult> Login(LoginModel details,
string returnUrl) {
    if (ModelState.IsValid) {
        AppUser user = await userManager.FindByEmailAsync(details.Email);
        if (user != null) {
            await signInManager.SignOutAsync();
            Microsoft.AspNetCore.Identity.SignInResult result =
            await signInManager.PasswordSignInAsync(user, details.Password, false, false);
            if (result.Succeeded) {
                return Redirect(returnUrl ?? "/");
            }
        }
        ModelState.AddModelError(nameof(LoginModel.Email),"Invalid user or password");
        }
       return View(details);        
    }
}
```

* 首先，查询用户

```
AppUser user = await userManager.FindByEmailAsync(details.Email);
```

* 然后，认证用户

```
await signInManager.SignOutAsync();
Microsoft.AspNetCore.Identity.SignInResult result =
await signInManager.PasswordSignInAsync(user, details.Password, false, false);
```

`SignOutAsync`用于清除用户现存的session信息；PasswordSignInAsync执行实际认证。作为认证的过程，Identity在响应里增加了Cookie信息，随后浏览器的请求都会包含这部分cookie信息。基于Identity使用时，Cookie是自动处理的，我们可以不用关注。

## 5. 角色管理

* IdentityRole:内置的角色模型
* RoIeManage&lt;T&gt;：角色管理服务

## 6. Seeding Database

解决管理员默认创建的问题

* appsetting.json增加管理员用户配置

```
//appsettings.json File
{
    "Data": {
    "AdminUser": {
    "Name": "Admin",
    "Email": "admin@example.com",
    "Password": "secret",
    "Role": "Admins"
    },
    "SportStoreIdentity": {
        "ConnectionString": "Server=(localdb)\\MSSQLLocalDB;Database=IdentityUsers;Trusted_Con
        nection=True;MultipleActiveResultSets=true"
        }
    }
}

```

```
//静态方法读取配置，生成管理员
public class AppIdentityDbContext : IdentityDbContext<AppUser> {
    public AppIdentityDbContext(DbContextOptions<AppIdentityDbContext> options)
    : base(options) { }
    public static async Task CreateAdminAccount(IServiceProvider serviceProvider,
    IConfiguration configuration) {
        UserManager<AppUser> userManager = serviceProvider.GetRequiredService<UserManager<AppUser>>();
        RoleManager<IdentityRole> roleManager = serviceProvider.GetRequiredService<RoleManager<IdentityRole>>();

        string username = configuration["Data:AdminUser:Name"];
        string email = configuration["Data:AdminUser:Email"];
        string password = configuration["Data:AdminUser:Password"];
        string role = configuration["Data:AdminUser:Role"];
        if (await userManager.FindByNameAsync(username) == null) {
            if (await roleManager.FindByNameAsync(role) == null) {
                await roleManager.CreateAsync(new IdentityRole(role));
            }
            AppUser user = new AppUser {
                UserName = username,
                Email = email
            };
            IdentityResult result = await userManager.CreateAsync(user, password);
            if (result.Succeeded) {
            await userManager.AddToRoleAsync(user, role);
            }
        }
    }
}
```

```
//Starup Configure方法调用
public void Configure(IApplicationBuilder app) {
    app.UseStatusCodePages();
    app.UseDeveloperExceptionPage();
    app.UseStaticFiles();
    app.UseIdentity();
    app.UseMvcWithDefaultRoute();
    AppIdentityDbContext.CreateAdminAccount(app.ApplicationServices,
    Configuration).Wait();
}
```



