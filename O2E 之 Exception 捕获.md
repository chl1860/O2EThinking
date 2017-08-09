异常捕获，是项目中常用的代码段。充分体现了面向对象语言的鲁棒性同时保证了代码的健壮性。项目中的异常捕获代码常常是这样：
```csharp
try{
    //do your stuff
}catch(Exception ex){
    throw ex;
}finally{
    //do your finally stuff
}
```
从上面的代码段可以看出，异常捕获的语法是简单直观的，但是有时简单直观只是表象，中间有可能会夹带些坑害淫民群众的小细节。

例如，异常信息并不总是在 Exception 实例的 Message 中，有时可能会中 Exception 的 InnerException 实例的 Message 中，常见的是在通过 repository 做数据库操作的情况：

```csharp
//repository.cs
public class Repository{
    public void DbAction(){
        try{
            //do your db action
        }catch(Exception ex){
            throw ex;
        }finally{
            //do your finally staff
        }
    }
}

// service.cs
public class Service{
    var _repository = new Repository(); 
    public void DbServAction(){
        try{
           _repository.DbAction(); 
        }catch(Exception ex){
            throw ex
        }
    }
}

//contrller.cs 中调用
public class Controller{
    var _service = new Service();
    public ActionResult DoAction(){
        try{
            _service.DbServAction();
            
            return Json(new {
                msg:'Ok'
            });
        }catch(Exception ex){
            return Json(new {
                msg:ex.Message //假如数据库报错，错误信息会被包含中 InnerException 实例的 Message 中
            });
        }
    }
}

```
如上，代码段中最后调用时，如果是数据库报错，则错误信息会被包含中 InnerException 的 Message 属性中，于是可能就有如下代码：

```csharp
string msg = "";
try{
    //do your stuff
}catch(Exception ex){
    if(ex.InnerException != null){
        msg = ex.InnerException.Message;
    }else{
        msg = ex.Message;
    }
}
```
于是问题出现了，如果这样写，是不是每个 try catch 都得这样做？答案当然是否定的

因为 Exception 是可以冒泡的，冒泡意味着我们可以让最里层的异常沿着异常作用链向外走到最外层。说人话就是我们可以中最外层捕获最里面的异常。举个栗子：

```csharp
public class A {
    public void Ado(){
        try{
            //do your stuff
        }catch(Exception ex){
            throw ex;
        }
    }
}

public class B {
    var _a = new A();

    public void Do(){
        try{
            _a.Ado();
        }catch(Exception ex){
            //here can catch the exception from A
        }
    }
}
```
于是我们可以在内层嵌套中简单抛出 Exception，然后在最外层调用中进行捕获即可。

而对于可能出现在 Exception 实例或 InnerException 实例中 Message，我们可以通过一个简单的扩展来完成收敛(当然也可以有其他方式)：

```csharp
//ExceptionExtension.cs
public static class ExceptionExtension{
    public static Exception GetOrgException(this Exception ex){
        return ex.InnerException != null ? ex.InnerException : ex;
    }
}

//contrller.cs 中调用
public class Controller{
    var _service = new Service();
    public ActionResult DoAction(){
        try{
            _service.DbServAction();
            
            return Json(new {
                msg:'Ok'
            });
        }catch(Exception ex){
            return Json(new {
                msg:ex.GetOrgException().Message //这样获取具体 Message
            });
        }
    }
}

```
如此一来，我们可以在项目中得到更简单方便的异常捕获代码。
