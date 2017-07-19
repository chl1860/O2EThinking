#### 通过正则表达式简化 JSON 时间格式转换操作

项目中常常遇到传出格式为 **/Date(xxxxx)/** 这样的 Json 时间字符串，有时为了方便进一步处理，需要将其转换为通用的 Date 类型。转换的步骤其实很简单：
1. 取出字符串中的整数
2. 利用 Date构造函数，实例化即可。

如此，我们可以有以下手段来解决：
- 通过字符串截取方式 (这是大部分人首先会想到的)

```javascript
    /**
    * 1. 通过截取字符串判定 Date 类型
    * 2. 调用 2 次 replace 获取字符串中的整数 
    */ 
    function convertToDate(jsonDate){
        if(jsondate.substr(0, 5) == "/Date"){
            var date = new Date(parseInt(jsondate.replace("/Date(", "").replace(")/", ""), 10));
            return date;
        }
        return jsonDate
    }
```
以上代码，发生了一次字符串截取和两次替换操作。两次重复的 replace 操作，对于强迫症来说是不可接受的，是否可以减少？当然是可以的

- 通过正则表达式可以减少替换操作次数，于是有如下代码：

```javascript

    /**
    * 1. 通过截取字符串判定 Date 类型
    * 2. 通过正则 /[^0-9]+/ 减少 replace 操作次数 
    */ 
    function convertToDate(jsonDate){
        if(jsondate.substr(0, 5) == "/Date"){
            var date = new Date(parseInt(jsondate.replace(/[^0-9]+/g,''), 10));
            return date;
        }
        return jsonDate
    }
```
ok，替换函数的调用次数减少为了 1 次。既然都使用正则表达式了，字符串截取似乎也有点不能忍。于是有如下代码：

```javascript
    /**
    * 1. 通过正则 /^\/Date\(\d+\)\/$/ 避免字符串截取操作
    * 2. 通过正则 /[^0-9]+/ 减少 replace 操作次数 
    */ 
    function convertToDate(jsonDate){
        if(/^\/Date\(\d+\)\/$/gi.test(jsonDate)){
            var date = new Date(parseInt(jsondate.replace(/[^0-9]+/g,''), 10));
            return date;
        }
        return jsonDate
    }

```
这里似乎只是替换了许多字符串操作，但代码并么有明显简化相关操作，不满足啊。秉着不抛弃不放弃的原则，最后来一剂猛药：

```javascript
    /**
    * 1. 通过 /^\/Date\((\d+)\)\/$/ 判断时间格式同时提取整数
    * 2. 摒弃了字符串截取和查找替换过程
    */
    function convertToDate(jsonDate){
        var ls = /^\/Date\((\d+)\)\/$/gi.exec(jsonDate);
        return ls ? new Date(parseInt(ls[1],10)) : jsonDate;
    }
```
至此，相对原有的方法，通过正则表达得到了一个更加精简的方式。




