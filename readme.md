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

## 用户机密文件

- secrets.json是用户机密文件，不会保存在项目目录中，而是保存在用户本地

## 中间件

### 添加静态文件中间件

- 添加后可以通过目录访问wwwRoot下的静态文件

```c#
app.UseStaticFiles();
```

- 添加默认文件中间件

```c#
app.UseDefaultFiles();
```
