# Why JSX

和很多人一样，第一次看到 JSX 是排斥的。毕竟将 dom 揉到 JS 代码里的方式，让很多人不适应或是不喜欢，也少了尝试的勇气。但JSX真的如此难堪？

在某一次项目修改中，我发现一个简单的弹出选择框，说他简单，是因为基本上有点前端经验的人都可以随手捏一个这样的东东。实际上也是如此，在项目的各个角落都能看到不同的开发人员对此的实现。仔细分析了这些功能发现，其实这玩意儿的前端显示逻辑是完全可以统一的，于是顺手写了一个公共的 dom + js 实现。但有个略微绊脚的问题是，在不同的业务层面上，js 会伴随业务逻辑做一些调整。为了解决这种问题，我用了注入，通过注入在一定层面上使问题得到了缓解，剩下问题是，我的 dom 和 js 是分离的，虽然在功能层面上我把它们看着是一个 component， 但这实际上也并不怎么好维护，我依旧需要为此维护两套东西(dom + js)。直到看到 ReactJs。但这里我只想说一下 JSX 如何帮我解决那个遗留问题，并带来了哪些利好。

### 1.  JSX 使得前端的 component更存粹。（这里的更存粹是指：知识体系统一，功能单一）.

以 button 举例，传统的 dom + js 方式是这样：

```html
    //button 标签
    <button onclick = 'btnFun()'>Traditional Button</button>
    ....
    <script>
        function btnFun(){
            //your stuff doing here
            ....
        }
    <script>
```
这里你需要维护 dom 文件以及相关的 js 文件，随着项目规模的增长，这会让你觉得越来越难以应付

JSX 方式
```javascript
    //button component
    const btnComponent = ({clickFunc,text}) =>(<button onClick = (e)=>{clickFunc(e);}>{text}</button>)
```
现在对于我们来说，btnComponent是一个存粹的组件。从维护上，我们只需要维护这一个 JS 文件即可；从功能上看，btnComponent 只负责button的显示, 而对于其具体内容以及业务逻辑并不关心，这使得这个组件的可复用性增强

### 2. JSX 通过 VDOM 在一定程度上提高了浏览器的渲染效率

VDOM 其实这并不是一个全新的概念，其实质就是 dom 节点的 JSON 表示。，这也意味着你的操作可能仅是在JS层面上. 

VDOM 的主要渲染过程如下 *(这里只是粗略演示, 对此感兴趣的可以去看一下 [hyperscript](https://github.com/hyperhype/hyperscript) 库)*： 

- VDOM 的生成 

```javascript
const h = (nodeName, attributions, ...args) =>{
    let children = args.length ? [].concat(...args):[];
    return {
        nodeName,
        attributions,
        children
    };
}
// let vnode = h('div',{id:'test'},[])
// {nodeName:'div', attributions:{id:'test'}, }
````
- VDOM => DOM

```javascript
const render =(vnode)=>{
    if(vnode.split){
        if(vnode.split){
        return document.createElement(vnode);
    }
    let n = document.createElement(vnode.nodeName);
    let a = vnode.attributions || {};
    Object.keys(a).forEach(k => k !=='innerText'?n.setAttribute(k,a[k]):n.innerText=a[k]);

    (vnode.children || []).forEach(c => n.appendChild(render(c)));
    return n;
    }
}

//let vnode = h('div',{id:'test'},[])
//let dom = render(vonode)

//the result is: <div id='test'></div>

```
*这里的 render 只是为了演示原理，因此并没有做算法上的优化。（实际上大部分含有 VDOM 框架的 Render 实现，都采用了一些智能算法，以减少 dom 操作次数）*

### 3. 从源头杜绝了 XSS 攻击

从本质上看 VDOM 只是一个 JS 的 plain object，因此不会被浏览器当做 DOM 直接解析，最终得到的 DOM 是通过 render 函数解析后得到，所以直接向 DOM 文件中植入可执行代码是木有用滴.... 

### 4. 对 SEO 友好