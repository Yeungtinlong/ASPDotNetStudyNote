# Asp.net core 学习

基于.Net core 3.1的学习笔记

## 项目文件的`AspNetCoreHostingModel`属性
- InProcess: 进程内托管，使用IIS服务器
- OutOfProcess: 进程外托管，使用Kestrel服务器

## 通过依赖注入`IConfiguration`访问到json里的属性

```c#
private readonly IConfiguration _configuration;

public Startup(IConfiguration configuration) {
   _configuration = configuration;
}
```

