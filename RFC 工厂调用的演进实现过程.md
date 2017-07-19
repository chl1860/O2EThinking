**开始阅读前，请确保项目中已引入 RFC 调用相关的 SAP 引用及命名空间**

企业级应用开发过程中时常会遇到与 SAP 进行数据交互的场景。RFC 作为 SAP 提供的远程数据调用接口，为这样的数据交互场景带来了很大的方便。因此，自然让人想到，如果能够复用里边的方法和流程将为开发带来极大的方便。问题来了，如何复用？又该怎样进行复用？

通过对项目中原有 RFC 代码进行分析，发现 RFC 调用可以大致分为以下步骤：
1. 初始化 RFC 远程连接
2. 传入 RFC 名称以及参数
3. 获得 RFC 返回值

其中第一步可以通过以下代码完成:
```CSharp
/**
* 以下代码通过 DestinationName 初始化一个 RFC 远程连接，返回一个 RfcDestination 对象
*/
var rfcDestination =  RfcDestinationManager.GetDestination(DestinationName);

``` 
第二步可以拆分为几个子步骤：

- 创建 RfcFunction 以备调用，其实现伪码如下：

```csharp
//通过向 RfcRepository 中的 CreateFunction 方法传入 RFC 名称,创建 RFC 实例
var rfcFunc = repo.CreateFunction(_rfcName)

```
- 向 RFC 中传入参数：

```csharp
 //传参过程类似于数据库操作传参
 rfcFunc.SetValue("the_rfc_param", your_value);
```
最后，获取到 SAP 系统的返回值
```csharp
//通过向 RfcFunction 的invoke方法中传入对应的 rfcDestination, 获取 SAP 返回值
rfcFunc.invoke(rfcDestination)
``` 

进一步分析发现，第一步和最后一步基本上是固定的，而第二步主要变化的部分是 RFC 参数，但不同参数传入，最后都得到同种类型(RFCFunction)的返回值。这样的场景不难让人想到工厂模式。问题来了，我们要怎样实现这个工厂？

分析 RFC 调用过程发现， SAP 为我们提供了一个 IRfcFunction 接口，所有的 RfcFunction 都是对这个接口的实现，于是项目中所有的 RfcFunction 都可以用 IRfcFunction 替换 (这不就是传说中的里氏替换么？)。综合以上分析可以得到一个工厂主方法的框架

```csharp
    public IRfcFunction InvokeRfc()
    {
        try{
            //step 1: 初始化 RfcDestination
            var rfcDestination =  InitRfcDestination();

            //step 2: 构造 IRfcFunction
            var rfcFunc = GenerateFunc()
                
            //step3: 通过调用 RFC
            rfcFunc.Invoke(rfcDestination);

            return rfcFunc;
        }
        catch (Exception ex){
            if (ex.InnerException != null){
                throw ex.InnerException;
            }
            else{
                throw ex;
            }
        }
    }

```

对于主方法中的可变部分(RFC 参数传递部分) 通过工厂接口指定规则。
```csharp
    /// <summary>
    /// 定义创建 IRfcFunction 的接口
    /// </summary>
    public interface IRfcFuncGenerator
    {
        /// <summary>
        /// 创建 IRfcFunction 对象并返回
        /// </summary>
        /// <param name="repo">RfcRepository</param>
        /// <returns>IRfcFunction</returns>
        IRfcFunction GetRfcFunc(RfcRepository repo);
    }
```
同时对主方法进行进一步改进：

```csharp
// 工厂主方法实现伪码
public class SAP_Post
{
    private RfcDestination _rfcDefination;
    private IRfcFuncGenerator _iRfcGenerator;

    /// <summary>
    /// Rfc 工厂方法构造函数
    /// </summary>
    /// <param name="iRfcGenerator">IRfcFuncGenerator</param>
    public SAP_Post(IRfcFuncGenerator iRfcGenerator)
    {
        _iRfcGenerator = iRfcGenerator
    }
    
    /// <summary>
    /// 调用 Rfc 并返回调用后的 IRfcFunction
    /// </summary>
    /// <returns>IRfcFunction</returns>
    public IRfcFunction InvokeRfc()
    {
        try
        {
            InitRfcDefination(); // 实例化 _rfcDefination
            var rfcFunc = _iRfcGenerator.GetRfcFunc(_rfcDefination.Repository);
            rfcFunc.Invoke(_rfcDefination);
            return rfcFunc;
        }
        catch (Exception ex)
        {
            throw ex;
            
        }
    }
```
IRfcGenerator 的接口实现实例如下：
```csharp
public RfcGenerator:IRfcFuncGenerator{
    //..your property
    private string _rfcName；
    public RfcGenerator(string rfcName){
        //...some assignment
        _rfcName = rfcName;
    }

    public IRfcFunction GetRfcFunc(RfcRepository repo){
        var rfcFunc = repo.CreateFunction(_rfcName); //这一步是创建 RFCFunctioin 是必须的
        //your seting value code..
        rfcFunc.SetValue("the_rfc_param", your_value);
        //...
        return rfcFunc
    }
}
```
至此，便可以使用如下统一方式调用 RFC：

```csharp
var sap_post = new SAP_Post(RfcGenerator); // 传入IRfcGenerator 实例化 SAP_Post
var func = sap_post.InvokeRfc();
```
以上便是整个 RFC 工厂调用的演进实现过程 