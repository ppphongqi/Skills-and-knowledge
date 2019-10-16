## Vue之MVVM原理

### 什么是MVVM？

**MVVM是Model-View-ViewModel**的缩写，它是一种基于前端开发的架构模式，View和Model之间并没有直接的联系，而是通过ViewModel进行交互，其核心是ViewModel通过双向数据绑定将View和Model连接起来了，这使得View数据的变化会同步到Model中，而Model数据的变化也会立即反应到View上

### MVVM具体实现原理？

在Vue中使用**数据劫持**，采用**Object.defineProperty**的**getter**和**setter**，并结合**观察者模式**来实现数据绑定。当把一个js对象传给**Vue实例**来作为它的**data**属性时，Vue会遍历它的属性，用**Object.defineProperty**将它们赋予set和get，在内部它们让Vue追踪依赖，当属性被访问和修改时通知变化

> Observer：数据监听器，能够对数据对象的所有属性进行监听，如有变动可拿到最新值并通知订阅者，内部采用Object.defineProperty的getter和Setter来实现的。


>Compile：模板编译，它的作用对每个元素节点的指令和文本节点进行扫描和解析，根据指令模板替换数据，以及绑定相应的更新函数。

>Watcher：订阅者，作为连接Observer和Compile的桥梁，能够订阅并收到每个属性变动的通知，执行指令绑定的相应回调函数。

>Dep：消息订阅器，内部定义了一个数组，用来收集订阅者（Watcher），数据变动触发notify函数，再调用订阅者的update方法。

分析上图：当执行new Vue()时，Vue就进入了初始化阶段，一方面Vue会遍历data选项中的属性，并用Object.defineProperty将它们转换为**getter/setter**，实现数据变化监听功能；另一方面，Vue的模板编译Compile对元素节点的指令和文本节点进行扫描和解析，初始化视图，Object.defineProperty在get钩子中addSub订阅Watcher并添加到消息订阅器（Dep）中，初始化完成。
当数据发生变化时，Observer中的setter方法被触发，setter会立即调用Dep.notify()，Dep开始遍历所有的订阅者，并调用订阅者的update方法，订阅者收到通知后对视图进行相应的更新。

### 具体代码实现？
#### Vue类实现

在入口Vue类中调用Observer进行数据劫持，将数据变成响应式数据；调用Compile模板编译，找到需要替换数据的元素，进行编译及初始化；最后进行数据代理，实现vm.school而不用使用vm.$data.school调用数据

具体代码如下：
```

class Vue{
    constructor(options){
        this.$el=options.el;
        this.$data=options.data;
        // 如果$el存在，那么可以找到上面的HTML模块
        if(this.$el){
            // 把数据变成响应式 当 new Observer后，school就变成了响应式数据
            new Observer(this.$data)
            // 现在也需要让vm代理this.$data
            this.proxyVm(this.$data)
            // console.log(this.$data)
            // 需要找到模块中需要替换数据的元素，编译模板
            new Compiler(this.$el,this)           
        }
    }
    // 让vm代理data
    proxyVm(data){
        for(let key in data){ //data: {school:{name:beida,age:100}}
        // console.log(this)  this vm实例
        Object.defineProperty(this,key,{
            get(){
                return data[key]
            }
        })

        }
    }
}
```



#### Compile--模板编译类实现

主要分三步：

1.把真实的dom利用文档碎片（fragment）移入到内存中，减小内存消耗，操作dom速度加快;
补充：将el中的内容移入到文档碎片fragment中是一个进出栈的过程，el的子元素被移到fragment，出栈后，el的下一个子元素就会变成firstChild

2.编译--遍历元素节点和文本节点v-model...,{{}}，然后执行相应的操作。 具体操作：
- 获取元素的节点，提取其中的的指令或者模板**{{}}**
- 分类编译指令的方法compileElement和编译文本**{{}}**方法

3.把编译好的fragment放回到原页面中

具体代码如下:

```
class Compiler{
    constructor(el,vm){
        this.el=this.isElementNode(el)?el:document.querySelector(el);
        this.vm=vm;
        // console.log(this.el)
        let fragment=this.node2fragment(this.el);
        // console.log(fragment);
        // 替换操作 (编译模板) 用数据来编译
        this.compile(fragment);

        // 把替换完的数据重新给网页
        this.el.appendChild(fragment)



    }
    // 判断一个属性是否是一个指令
    isDirective(attrName){
        return attrName.startsWith("v-");   //返回的是boolean值
    }

    // 编译元素节点
    compileElement(node){
        let attributes=node.attributes;   //得到某个元素的属性节点  是个伪数组
        // console.log(attributes)
        [...attributes].forEach(attr=>{
            let {name,value:expr}=attr;   //解构赋值
            // console.log(expr)
            if(this.isDirective(name)){
                // console.log(name+"是一个指令");  //v-model
                let [,directive]=name.split('-');
                // console.log(directive) //model，将v-去掉
                ComplierUtil[directive](node,expr,this.vm);
            }
        })

    }
    // 编译文本节点
    compileText(node){
        let content=node.textContent;
        let reg=/\{\{(.+?)\}\}/;
        //reg.test(content) 如果content满足我们写的正则，返回true，否则false
        if(reg.test(content)){
            ComplierUtil["text"](node,content,this.vm);
        }

    }
    // 编译
    compile(node){
        // childNodes并不包含li得到的仅仅是子节点
        // console.log(node.childNodes) [text, input, text, div, text, div, text, ul, text]
        let childNodes=node.childNodes; 
        // console.log(Array.isArray(childNodes))  //得到的childNodes是一个伪数组
        [...childNodes].forEach(child=>{  //[...childNodes]将伪数组childNodes转变为真正数组
            if(this.isElementNode(child)){
                // console.log(child+"是一个元素节点")
                this.compileElement(child);
                // 可能一个元素节点中嵌套其他的元素节点，还可能嵌套文本节点
                // 如果child内部还有其他节点，需要利用递归重新编译
                this.compile(child);
            }else{
                // console.log(child+"得到的是文本节点")
                this.compileText(child);
            }
        })


    }
    // 判断一个节点是否是元素节点
    isElementNode(node){
        return node.nodeType===1;
    }
    // 将网页的HTML移到文档碎片中
    node2fragment(node){
        // 创建一个文档碎片
        let fragment=document.createDocumentFragment();
        let firstChild;
        while(firstChild=node.firstChild){
            fragment.appendChild(firstChild);
        }
        return fragment;
    }
}



// 写一个对象{}，包含了不同的指令对应不同的处理方法
ComplierUtil={
    getVal(vm,expr){
        // console.log(expr.split("."))   // ["school","name"]
        // 第一次data是vm.$data即 school:{name:xx,age:xx}，current 是school
        // 第二次data是school，current是name 即return data[current]==> school[current]
        return expr.split(".").reduce((data,current)=>{
            return data[current];
        },vm.$data);
        
    },
    setVal(vm,expr,value){
        // console.log(expr.split("."))   // ["school","name"]
        // 第一次data是vm.$data即 school:{name:xx,age:xx}，current 是school，index是0，arr是["school","name"]
        // 第二次data是undefined（没有处理累加，默认是undefined），current是name，index是1，arr是["school","name"]
        expr.split(".").reduce((data,current,index,arr)=>{
            // console.log(data)
            if(index==arr.length-1){
                // console.log(current)
                // console.log(data)
                return data[current]=value;
                // console.log(data[current])
                // console.log(111)
            }
            return data[current];
            
        },vm.$data)
    },
    model(node,expr,vm){  //node是带指令的元素节点，expr是表达式，vm是vue对象
        let value=this.getVal(vm,expr)
        let fn=this.updater["modelUpdater"]
        // 给输入框添加一个观察者，如果后面数据发生改变了，就通知观察者
        new Watcher(vm,expr,(newVal)=>{
            fn(node,newVal);
        })
        // 给input添加一个input事件，
        node.addEventListener("input",(e)=>{
            let value=e.target.value;
            this.setVal(vm,expr,value);
        })
        fn(node,value)
        
    },
    html(){

    },
    // 得到新的内容
    getContentValue(vm,expr){
        return expr.replace(/\{\{(.+?)\}\}/g,(...args)=>{
            return this.getVal(vm,args[1])
        });
    },
    text(node,expr,vm){
        // console.log(node) //"{{school.name}}"
        // console.log(expr) //{{school.name}} {{school.age}}
        // console.log(vm)
        let content=expr.replace(/\{\{(.+?)\}\}/g,(...args)=>{
            // console.log(vm)
            // console.log(args)
            new Watcher(vm,args[1],()=>{
                fn(node,this.getContentValue(vm,expr));
            })
            return this.getVal(vm,args[1])  //baida 100
        })
        let fn=this.updater["textUpdater"];

        fn(node,content)

    },
    // 更新数据
    updater:{
        modelUpdater(node,value){
            node.value=value;
        },
        htmlUpdater(){

        },
        // 处理文本节点
        textUpdater(node,value){
            // textContent得到文本节点中内容
            node.textContent=value
        }

    }
}


```

#### Observer--数据劫持-实现双向数据绑定
```
// 实现数据的响应式--->数据劫持，当获取修改数据时，需要感应到（set和get）
class Observer{
    constructor(data){
        this.observer(data)
    }
    // 把上面的数据变成响应式数据，把一个对象数据做出响应式
    observer(data){
        if(data&& typeof data=='object'){
            // console.log(data) //{school: {name: "beida", age: 100}}
            // for in 循环一个js对象
            for(let key in data){
                // console.log(key) //school
                // console.log(data[key]) //{name: "beida", age: 100}
                this.defindReactive(data,key,data[key])


            }
        }
    }
    defindReactive(obj,key,value){
        this.observer(value)  //如果一个数据是一个对象，也需要将其变成响应式
        // Object.defineProperty(obj,prop,descriptor)函数会直接在obj上定义一个新属性或修改一个新属性
        // obj要在其上定义属性或修改的对象，prop要定义或修改的属性名称，descriptor 将被定义或修改的属性描述符
        // 这是要修改obj对象的school属性
        let dep=new Dep();   //不同的watcher放到不同的dep中
        Object.defineProperty(obj,key,{
            // 修改如下  当获取school时，会调用get
            get(){
                Dep.target&&dep.subs.push(Dep.target)
                // console.log("get...")
                return value
            },
            // 当设置school时，会调用set
            set:(newVal)=>{
                if(newVal!=value){
                    // console.log("set...")
                    this.observer(newVal)
                    value=newVal;
                    // 值改变时，通知观察者
                    dep.notify();

                }
            }

        })

    }
}


```


#### Watcher--订阅者


```
// 观察者
class Watcher{
    constructor(vm,expr,cb){
        this.vm=vm;
        this.expr=expr;
        this.cb=cb;
        // 刚开始需要一个老的状态
        this.oldValue=this.get();

    }
    get(){
        Dep.target=this;
        let value=ComplierUtil.getVal(this.vm,this.expr);
        Dep.target=null;
        return value;
    }
    // 当状态改变后，会调用观察者的update
    update(){
        let newVal=ComplierUtil.getVal(this.vm,this.expr);
        if(newVal!=this.oldValue){
            this.cb(newVal)
        }
    }
}

```


#### Dep--消息订阅器

```
// 存储观察者的类Dep
class Dep{
    constructor(){
        this.subs=[];  //在subs中存放所以的watcher
    }
    // 添加watcher即订阅
    addSub(watcher){
        this.subs.push(watcher)
    }
    // 通知 发布 通知subs容器中的所有观察者
    notify(){
        this.subs.forEach(watcher=>watcher.update())
    }
}


```