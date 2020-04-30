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

通过索引取得json属性

```c#
var configValue = _configuration["MyKey"];
```

注入日志组件`ILogger`打印日志，了解中间件执行顺序。
- 执行到终端中间件则逆转传递方向，传出响应。

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

添加静态文件中间件后，可以如跟目录一样访问wwwRoot下的静态文件

```c#
app.UseStaticFiles();
```

修改访问网站响应的默认静态文件

```c#
DefaultFilesOptions defaultFilesOption = new DefaultFilesOptions();
defaultFilesOptions.DefaultFilesNames.Clear();
defaultFilesOptions.DefaultFilesNames.Add("index.html");

app.UseDefaultFiles(defaultFilesOptions);
```

- 如果直接使用`UseDefaultFiles()`不填写参数，则会默认查找`Index.htm`,`Index.html`,`Default.htm`,`Default.html`

或者用`FileServerOptions`类

```c#
FileServerOptions fileServerOptions = new FileServerOptions();
fileServerOptions.DefaultFilesOptions.DefaultFileNames.Clear();
fileServerOptions.DefaultFilesOptions.DefaultFileNames.Add("52abp.html");

app.UseFileServer(fileServerOptions);
```

- 各种中间件通过输入`app.Use`的语法补偿可以找到
- Asp.Net Core 默认不支持静态文件服务
- `UseDefalutFiles()`必须注册在`UseStaticFiles()`前
- `UseFileServer`结合了`UseStaticFiles`,`UseDefaultFiles`,`UseDirectoryBrowser`中间件的功能，不推荐使用，`UseDirectoryBrowser`会暴露文件目录

## 开发者异常

抛出异常

```c#
throw new Exception("抛出异常");
```

异常中间件

```c#
if(env.IsDevelopment()) {
    app.UseDevelopmentExceptionPage();
}
```

- `app.UseDevelopmentExceptionPage()` 必须趁早调用，在异常中间件调用前`throw`的异常不会抛出

## 配置环境变量

通过默认注入的`IWebHostEnvironment env`可以配置相关信息

.net 自带`env.IsDevelopment()`,`env.IsStaging()`,`env.IsProduction()`

```c#
if(env.IsDevelopment()) {
    app.UseDeveloperExceptionPage();
}
```

- 可以通过自定义`env.IsEnvironment("")`来判断当前是否自定义的环境
- 在开发环境中，在launchsettings.json文件设置环境变量
- 而Staging或者Production变量，尽量在操作系统中配置

## MVC

- `View`: 包含显示逻辑，用于显示Controller提供给它的模型中数据
- `Controller`: 处理Http请求,调用模型，选择对应的视图来呈现该模型
- `Model`: 包含一组数据的类和管理该数据的逻辑信息

用户展示层(MVC在此)、业务逻辑层、数据访问读取层

### 安装MVC

注入MVC服务

```c#
public void ConfigureServices(IServiceCollection services) {
    services.AddMvc(option => {
        option.EnableEndpointRouting = false;
    });
}

public void Configure(IApplicationBuilder app, IWebHostEnvironment env) {
    if(env.IsDevelopment()) {
        app.UseDeveloperExceptionPage();
    }

    app.UseRouting();
    app.UseStaticFiles();
    // 启用MVC中间件
    app.UseMvcWithDefaultRoute();
    app.UseEndpoints(endpoints => {
        endpoints.MapGet("/", async context => {
        await context.Response.WriteAsync("Hosting Environment: " + env.EnvironmentName);
        });
    });
}
```

创建`Controllers`文件夹，添加-控制器

- 启用MVC后，网址如`localhost:5000/home/index`，会寻找`HomeController`中的`Index`方法，如果找不到则会由终端中间件处理

- `AddMvcCore()`方法只会添加最核心的MVC服务
- `AddMvc()`方法添加了所有必须的MVC服务
- `AddMvc()`方法会在内部调用`AddMvcCore()`方法

## 从控制器传递数据到视图

### ViewData

- 是弱类型的字典(dictionary)对象
- 使用string类型的键值，存储和查找ViewData字典中的数据
- 运行时动态解析
- 没有智能提示，编译时也没有类型检测

```c#
// HomeController
public IActionResult Details() {
    Student model = _studentRepository.GetStudent(2);
    ViewData["Page Title"] = "Student Details";
    ViewData["Student"] = model;
    return View();
}
```

```html5
@using StudentManagement.Models;

<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <h3>@ViewData["PageTitle"]</h3>
    @{
        var student = ViewData["Student"] as Student;
    }
    <div>
        姓名: @student.Name
    </div>
    <div>
        班级: @student.ClassName
    </div>
    <div>
        邮箱: @student.Email
    </div>
</body>
</html>
```

### ViewBag`

- ViewBag是ViewData的包装器
- ViewData使用字符串键名来存储和查询数据
- ViewBag使用动态属性来存储和查询数据
- 均是在运行时动态解析

```c#
public IActionResult Details() {
    Student model = _studentRepository.GetStudent(2);
    ViewBag.PageTitle = "Student Details";
    ViewBag.Student = model;
    return View();
}
```

```html5
@using StudentManagement.Models;

<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <h3>@ViewBag.PageTitle</h3>
    <div>
        姓名: @ViewBag.Student.Name
    </div>
    <div>
        班级: @ViewBag.Student.ClassName
    </div>
    <div>
        邮箱: @ViewBag.Student.Email
    </div>
</body>
</html>
```

### 强类型视图

- 首选采用强类型进行传输

```c#
public IActionResult Details() {
    Student model = _studentRepository.GetStudent(2);

    return View(model);
}
```

```html5
// 没有这句没有智能提示
@model StudentManagement.Models.Student
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <h3>@ViewBag.PageTitle</h3>
    <div>
        姓名: @Model.Name
    </div>
    <div>
        班级: @Model.ClassName
    </div>
    <div>
        邮箱: @Model.Email
    </div>
</body>
</html>
```

#### 数据传输对象DTO

当使用强类型时，若要增加数据可以使用`ViewModel`

```c#
public class HomeDetailsViewModel {
    public Student Student { get; set; }
    public string PageTitle { get; set; }
}
```

对`Model`进行一层封装，此处的`PageTile`正是新加入的数据

#### 使用迭代器遍历学生数据

```c#
public IActionResult Index() {
    IEnumerable<Student> model = _studentRepository.GetAllStudents();
    return View(model);
}
```

```html5
@model IEnumerable<StudentManagement.Models.Student>
<!DOCTYPE html>
<html>
<head>
    <title></title>
</head>
<body>
    <table>
        <thead>
            <tr>
                <th>ID</th>
                <th>名字</th>
                <th>邮箱</th>
                <th>班级名称</th>
            </tr>
        </thead>
        <tbody>
            @{
                foreach(var student in Model) {
                    <tr>
                        <td>
                            @student.Id
                        </td>
                        <td>
                            @student.Name
                        </td>
                        <td>
                            @student.Email
                        </td>
                        <td>
                            @student.Id
                        </td>
                    </tr>
                }
            }
        </tbody>
    </table>
</body>
</html>
```

## View布局视图

- 让web应用程序中所有的视图保持外观一致性
- 布局视图看起来像`ASP.NET Web Form`中的母版页
- 布局视图也具有`.cshtml`扩展名
- 在`ASP.NET Core MVC`中，默认情况下布局文件名为`_Layout.cshtml`
- 布局视图文件通常放在`Views/Shared`
- 在一个应用程序中可以包含多个布局视图文件

```html5
// _Layout.cshtml
<!DOCTYPE html>
<html>
<head>
    <meta name="viewport" content="width=device-width" />
    <title>@ViewBag.Title</title>
</head>
<body>
    <div>
        @RenderBody()
    </div>
</body>
</html>
```

```html5
@model StudentManagement.ViewModels.HomeDetailsViewModel
@{ 
    // Layout所指路径布局页面的@RenderBody()会呈现本页面的内容
    Layout = "~/Views/Shared/_Layout.cshtml";
}
    <h3>@Model.PageTitle</h3>
    <div>
        姓名: @Model.Student.Name
    </div>
    <div>
        班级: @Model.Student.ClassName
    </div>
    <div>
        邮箱: @Model.Student.Email
    </div>
```

## Section

在特定`html`节点添加`JS`节点

```html5
@model StudentManagement.ViewModels.HomeDetailsViewModel
@{
    Layout = "~/Views/Shared/_Layout.cshtml";
    //ViewBag.Tile = "学生详情页";
}

<h3>@Model.PageTitle</h3>
<div>
    姓名: @Model.Student.Name
</div>
<div>
    班级: @Model.Student.ClassName
</div>
<div>
    邮箱: @Model.Student.Email
</div>
// 定义该代码片段为Scripts
@section Scripts{
    <script src="~/js/CustomScript.js"></script>
}
```

```html5
<!DOCTYPE html>
<html>
<head>
    <meta name="viewport" content="width=device-width" />
    <title>@ViewBag.Title</title>
</head>
<body>
    <div>
        @RenderBody()
    </div>
    // Scripts在此处引入
    @if(IsSectionDefined("Scripts")) {
        @RenderSection("Scripts")
    }
</body>
</html>
```

## ViewStart

```
@{
    Layout = "_Layout";
}
```

- 创建于`Views`根目录下，帮助所有页面加载`_Layout.cshtml`的布局
- 若再创建一个`_ViewStart.cshtml`于`Views`目录的子目录下，则该目录的视图优先使用该目录的`_ViewStart.cshtml`

- ViewStart中的代码会在单个视图中的代码之前执行
- 移动公用代码到ViewStart视图中，如给布局视图文件设置属性
- ViewStart减少了代码冗余，提高了可维护性
- ViewStart文件支持分层

## ViewImports

- `_ViewImports.cshtml`文件通常放在Views文件夹中
- 包含公共命名空间

```
@using StudentManagement.Models
@using StudentManagement.ViewModels
```

## 路由

### 常规路由

如`http://localhost:3290/Home/Index`这个Url，会映射到`HomeController`类中的`Index()`操作方法

```c#
public class HomeController : Controller {
    private readonly IStudentRepository _studentRepository;

    public HomeController(IStudentRepository studentRepository) {
        _studentRepository = studentRepository;
    }

    public IActionResult Index() {
        IEnumerable<Student> model = _studentRepository.GetAllStudents();
        return View(model);
    }
}
```

如果是`http://localhost:3290/Home/Index/1`则会为`Index()`操作方法传入参数

- 默认的路由映射规则在中间件`app.UseMvcDefaultRoute()`已经配置好

- 也可以使用`app.UseMvc()`手动配置

```c#
app.UseMvc(routes => {
    routes.MapRoute("default", "{controller=home}/{action=Index}/{id?}");
});
```

### 属性路由

- 在`控制器类`或者`操作方法`上加上路由特性
- MVC会直接找到对应操作方法，而忽略Controller的类名

```c#
[Route("Home/Index/Details/{id?}")]
    public IActionResult Details(int? id) {
    HomeDetailsViewModel homeDetailsViewModel = new HomeDetailsViewModel() {
        Student = _studentRepository.GetStudent(id??1),
        PageTitle = "学生详细信息"
    };

    return View(homeDetailsViewModel);
}
```

```c#
public class HomeController : Controller {
    private readonly IStudentRepository _studentRepository;

    public HomeController(IStudentRepository studentRepository) {
        _studentRepository = studentRepository;
    }

    [Route("")]
    [Route("Home")]
    [Route("Home/Index")]
    public IActionResult Index() {
        IEnumerable<Student> model = _studentRepository.GetAllStudents();
        return View("~/Views/Home/Index.cshtml", model);
    }

    [Route("Home/Details/{id?}")]
    public IActionResult Details(int? id) {
        HomeDetailsViewModel homeDetailsViewModel = new HomeDetailsViewModel() {
            Student = _studentRepository.GetStudent(id??1),
            PageTitle = "学生详细信息"
        };

        return View("~/Views/Home/Details.cshtml", homeDetailsViewModel);
    }
}
```

## LibMan包管理器

- 右键项目`添加-客户端库`
- 会自动生成`libman.json`文件在项目根目录下
- 编辑`libman.json`文件可以修改版本号，右键`libman.json`文件可以清理、还原客户端库

## TagHelper

传统HtmlHelper写法：

```
// 生成一个a标签
@Html.ActionLink("查看", "details", "home", new { id = student.Id })
// 设置a标签的URL
<a href="@Url.Action("details", "home", new { id = student.Id })"><\a>
```

TagHelper写法:

```html
<a asp-controller="home" asp-action="details" asp-route-id="@student.Id" class="btn btn-primary">查看</a>
```

### Image Tag Helper

- `image TagHelper` 增强了`<img>`标签，为静态图像文件提供了`缓存破坏服务`

- 唯一的散列值并将其附加到图片的URL。此唯一字符串会提示浏览器从服务器重新加载图片，而不是从浏览器缓存重新加载

### Environment Tag Helper

用于在不同环境变量下加载不同的html代码块

```html
<environment include="Development">
    <link href="~/lib/twitter-bootstrap/css/bootstrap.css" rel="stylesheet" />
</environment>
<environment exclude="Development">
    <link rel="stylesheet" href="https://cdn.jsdelivr.net/npm/bootstrap@4.4.1/dist/css/bootstrap.min.css"
        integrity="sha384-Vkoo8x4CGsO3+Hhxv8T/Q5PaXtkKtu6ug5TOeNV6gBiFeWPGFN9MuhOf23Q9Ifjh"
        crossorigin="anonymous"
          >
</environment>
```

- `<link>`元素上的`integrity`用于检查`子资源完整性`
- SRI是一种安全功能

