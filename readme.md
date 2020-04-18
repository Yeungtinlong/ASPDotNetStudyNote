# Asp.net core 学习

基于.Net core 3.1的学习笔记

## 项目文件的`AspNetCoreHostingModel`属性
- InProcess: 进程内托管，使用IIS服务器
- OutOfProcess: 进程外托管，使用Kestrel服务器

## 依赖注入

注入`IConfiguration`接口，访问appsetting.json\secrets.json\launchSettings.json里的属性

```c#
private readonly IConfiguration _configuration;

public Startup(IConfiguration configuration) {
    _configuration = configuration;
}
```

- 通过索引取得json属性

```c#
var configValue = _configuration["MyKey"];
```

注入日志组件`ILogger`打印日志，了解中间件执行顺序。
- 执行到终端中间件则逆转传递方向，传出响应

```c#
public void Configure(IApplicationBuilder app, IWebHostEnvironment env, ILogger<Startup> logger) {
        if(env.IsDevelopment()) {
            app.UseDeveloperExceptionPage();
        }

        app.UseRouting();

        app.Use(async (context, next) => {
            context.Response.ContentType = "text/plain;charset=utf-8";

            logger.LogInformation("MW1: 传入请求");

            await context.Response.WriteAsync("中间件一号");
            await next();
            logger.LogInformation("MW1: 传出响应");
        });

        app.Use(async (context, next) => {
            context.Response.ContentType = "text/plain;charset=utf-8";

            logger.LogInformation("MW2: 传入请求");

            await context.Response.WriteAsync("中间件一号");
            await next();
            logger.LogInformation("MW2: 传出响应");
        });
        
        app.UseEndpoints(endpoints => {
            var processName = System.Diagnostics.Process.GetCurrentProcess().ProcessName;
            var configValue = _configuration["MyKey"];

            endpoints.MapGet("/", async context => {
                await context.Response.WriteAsync("Hello World!");
                logger.LogInformation("MW3: 处理请求并生成响应");
            });
        });
    }
}        
```

## 用户机密文件

- secrets.json是用户机密文件，不会保存在项目目录中，而是保存在用户本地

## 中间件

### 添加静态文件中间件

- 添加后可以如跟目录一样访问wwwRoot下的静态文件

```c#
app.UseStaticFiles();
```

### 添加默认文件中间件

- 修改访问网站响应的默认静态文件

```c#
DefaultFilesOptions defaultFilesOption = new DefaultFilesOptions();
defaultFilesOptions.DefaultFilesNames.Clear();
defaultFilesOptions.DefaultFilesNames.Add("index.html");

app.UseDefaultFiles(defaultFilesOptions);
```

- 如果直接使用`UseDefaultFiles()`不填写参数，则会默认查找`Index.htm`,`Index.html`,`Default.htm`,`Default.html`

- 或者用`FileServerOptions`类

```c#
FileServerOptions fileServerOptions = new FileServerOptions();
fileServerOptions.DefaultFilesOptions.DefaultFileNames.Clear();
fileServerOptions.DefaultFilesOptions.DefaultFileNames.Add("52abp.html");

app.UseFileServer(fileServerOptions);
```

- 各种中间件通过输入`app.Use`的语法补偿可以找到
- Asp.Net Core 默认不支持静态文件服务
- `UseDefalutFiles()`必须注册在`UseStaticFiles()`前
