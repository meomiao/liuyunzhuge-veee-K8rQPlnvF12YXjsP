[合集 \- OverallAuth2\.0 权限管理系统(6\)](https://github.com)[1\.从0到1搭建权限管理系统系列四 .net8 中Autofac的使用（附源码）09\-29](https://github.com/cyzf/p/18439606)[2\.从0到1搭建权限管理系统系列一 .net8 使用Swagger（附当前源码）09\-12](https://github.com/cyzf/p/18410483)[3\.从0到1搭建权限管理系统系列二 .net8 使用JWT鉴权（附当前源码）09\-18](https://github.com/cyzf/p/18417965):[蓝猫机场](https://fenfang.org)[4\.从0到1搭建权限管理系统系列三 .net8 JWT创建Token并使用09\-23](https://github.com/cyzf/p/18422784)[5\.（系列五）.net8 中使用Dapper搭建底层仓储连接数据库（附源码）10\-06](https://github.com/cyzf/p/18448855)6\.（系列六）.net8 全局异常捕获机制10\-12收起
**说明**


    该文章是属于OverallAuth2\.0系列文章，每周更新一篇该系列文章（从0到1完成系统开发）。


    该系统文章，我会尽量说的非常详细，做到不管新手、老手都能看懂。


    说明：OverallAuth2\.0 是一个简单、易懂、功能强大的权限\+可视化流程管理系统。


友情提醒：本篇文章是属于系列文章，看该文章前，建议先看之前文章，可以更好理解项目结构。


**有兴趣的朋友，请关注我吧(\*^▽^\*)。**


**![](https://img2024.cnblogs.com/blog/1158526/202408/1158526-20240824140446786-404771438.png)**


**关注我，学不会你来打我**


**为什么要用全局异常捕获？**


对于一个系统来说，全局异常捕获是必不可少的，它不仅可以把异常信息精简后反馈给用户，还能帮助程序员减少解决问题的时间，以及记录系统中任何一处发生异常的信息。


**你是否依然有以下苦恼？**


你是否还在为怎么记录系统异常日志而苦恼？


你是否还在为系统报错位置和报错信息苦恼？


你是否还在每个接口处增加日志记录操作？


如果你有，那么本篇文章正好可以解决你的难题。


**什么是全局异常捕获机制？**


全局异常捕获，顾名思义就是系统无论在那个位置发生错误都会被捕获，从而进行处理。


**创建接口返回模型**


创建一个接口返回模型：ReceiveStatus.cs 


它的主要作用是把接口返回的数据、信息推送给前端。




```
 /// 
 /// 接口返回实体模型
 /// 
 public class ReceiveStatus
 {
     /// 
     /// 编码
     /// 
     public CodeStatuEnum code { get; set; }

     /// 
     /// 信息
     /// 
     public string msg { get; set; }

     /// 
     /// 是否成功
     /// 
     public bool success { get; set; }

     /// 
     /// 构造函数
     /// 
     public ReceiveStatus()
     {
         code = CodeStatuEnum.Successful;
         success = true;
         msg = "操作成功";
     }
 }
 /// 
 /// 接口返回结果集
 /// 
 /// 
 public class ReceiveStatus : ReceiveStatus
 {
     /// 
     /// 数据
     /// 
     public List data { get; set; }

     /// 
     /// 总数量
     /// 
     public int total { get; set; }
 }
```



```
CodeStatuEnum.cs枚举值如下
```



```
 /// 
 /// 代码状态枚举
 /// 
 public enum CodeStatuEnum
 {
     /// 
     /// 操作成功
     /// 
     Successful = 200,

     /// 
     ///  警告
     /// 
     Warning = 99991,

     /// 
     /// 操作引发错误
     /// 
     Error = 99992
 }
```


创建好接口返回模型后，我们创建一个异常帮助类，它的主要用途，是区分【系统异常】还是用户自定义的【业务异常】。




```
/// 
/// 异常帮助类
/// 
public class ExceptionHelper
{
    /// 
    /// 自定义异常(会写入错误日志表)
    /// 
    /// 
    public static void ThrowBusinessException(string msg)
    {
        throw new Exception(msg);
    }

    /// 
    /// 自定义业务异常(不会写入错误日志表)
    /// 
    /// 信息信息
    /// 异常状态
    /// 返回结果集
    public static ReceiveStatus CustomException(string msg, CodeStatuEnum codeStatu = CodeStatuEnum.Warning)
    {
        ReceiveStatus receiveStatus = new();
        receiveStatus.code = codeStatu;
        receiveStatus.msg = msg;
        receiveStatus.success = false;
        return receiveStatus;
    }

}

/// 
/// 异常帮助类（返回数据）
/// 
/// 
public class ExceptionHelper : ExceptionHelper
{
    /// 
    /// 自定义业务异常(不会写入错误日志表)
    /// 
    /// 信息信息
    /// 异常状态
    /// 返回结果集
    public static ReceiveStatus CustomExceptionData(string msg, CodeStatuEnum codeStatu = CodeStatuEnum.Warning)
    {
        ReceiveStatus receiveStatus = new();
        receiveStatus.code = codeStatu;
        receiveStatus.msg = msg;
        receiveStatus.success = false;
        receiveStatus.data = new System.Collections.Generic.List();
        return receiveStatus;
    }
}
```


**创建全局异常捕获中间件**


在wenApi启动项目中创建一个类：ExceptionPlugIn.cs 


它的主要作用就是捕获系统中发生异常对代码和记录异常日志。


它需要继承一个接口：IAsyncExceptionFilter




```
/// 
/// 全局异常捕获中间件
/// 
public class ExceptionPlugIn : IAsyncExceptionFilter
{
    /// 
    /// 全局异常捕获接口
    /// 
    /// 
    /// 
    public Task OnExceptionAsync(ExceptionContext context)
    {
        //异常信息
        Exception ex = context.Exception;

        //异常位置
        var DisplayName = context.ActionDescriptor.DisplayName;

        //异常行号
        int lineNumber = 0;
        const string lineSearch = ":line ";
        var index = ex.StackTrace.LastIndexOf(lineSearch);
        if (index != -1)
        {
            var lineNumberText = ex.StackTrace.Substring(index + lineSearch.Length);
            lineNumber = Convert.ToInt32(lineNumberText.Substring(0, lineNumberText.IndexOf("\r\n")));
        }

        // 如果异常没有被处理则进行处理
        if (context.ExceptionHandled == false)
        {
            string exceptionMsg = "错误位置：" + DisplayName + "\r\n" + "错误行号：" + lineNumber + "\r\n" + "错误信息：" + ex.Message;
            // 定义返回类型
            var result = new ReceiveStatus<string>
            {
                code = CodeStatuEnum.Error,
                msg = "错误信息：" + exceptionMsg,
                success = false,
            };
            context.Result = new ContentResult
            {
                // 返回状态码设置为200，表示
                StatusCode = StatusCodes.Status500InternalServerError,
                // 设置返回格式
                ContentType = "application/json;charset=utf-8",
                Content = JsonConvert.SerializeObject(result)
            };
            //记录日志

        }
        // 设置为true，表示异常已经被处理了
        context.ExceptionHandled = true;
        return Task.CompletedTask;
    }
}
```


可以在OnExceptionAsync方法中添加记录日志、异常类型、异常分析等代码。


**添加到服务中**


编写好异常捕获机制后，我们需要把该类添加到Program.cs的服务中




```
//自定义全局异常处理
builder.Services.AddControllers(a =>
{
    a.Filters.Add(typeof(ExceptionPlugIn));
});
```


![](https://img2024.cnblogs.com/blog/1158526/202410/1158526-20241012155147615-155535681.png)


**测试全局异常捕获机制**


添加一个异常测试接口


**![](https://img2024.cnblogs.com/blog/1158526/202410/1158526-20241012155805775-1651091964.png)**


运行测试


![](https://img2024.cnblogs.com/blog/1158526/202410/1158526-20241012160016921-1039579902.png)


以上就是全局异常捕获机制，感兴趣的可以下载项目，修改吧。


**源代码地址：https://gitee.com/yangguangchenjie/overall\-auth2\.0\-web\-api**


**预览地址：http://139\.155\.137\.144:8880/swagger/index.html**


**帮我Star，谢谢。**


**有兴趣的朋友，请关注我微信公众号吧(\*^▽^\*)。**


**![](https://img2024.cnblogs.com/blog/1158526/202408/1158526-20240824140446786-404771438.png)**


关注我：一个全栈多端的宝藏博主，定时分享技术文章，不定时分享开源项目。关注我，带你认识不一样的程序世界


