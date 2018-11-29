#### 实现双向绑定的基本原理

###### angular.js 脏值检查（效率低）

> 无法检测数据是否发生改变，必须通过一些特定的条件，在条件执行的时候检测，如果数据改变就执行更新方法

例如：检测公交车人数是否发生变化

1. 通过setInterval----每秒检测

2. 可以通过在公交车关门的时候，就可以检测公交车上的人数是否发生变化



###### vue.js 数据劫持（Object.defineProperty）

Object.defineProperty 访问属性 --- enumerable, get, set方法



观察者模式（发布订阅模式）

优势：

1. 确认了一对多的依赖关系

2. 解耦

例子：

每天都在问是否有商品？

变成：

老板有商品了就联系我

```js
var boss ={
    //收集客户信息
    list:{},
    on(id, info){
        if(!this.list[id]){ //看是否收集过信息
            this.list[id] =[]
        }
        this.list[id].push(info)
        console.log(id + '预定成功')
    },
    emit(id){
        var infos = this.list[id]
        infos.forEach(item=>{
            console.log('用户')
            item()
            console.log('到了')
        })
    }
}

//收集你要的东西和信息 订阅
boss.on('mrlgm',function(){
    console.log('的商品')
})

//有东西了就联系你 发布
boss.emit('mrlgm')
```



###### MVVM的实现

vue的实现

```js
//创建一个vue对象的方式
let vue = new Vue({
    el: '#app',
    data:{
        a: 1,
        b: 2
    }
})

//开始创建一个vue构造函数
function Vue(options = {}){
    this.$options = options
    // data 用来保存options中的 data
    var data= this.$data  = this.$options.data
    // 监听data的变化
    observe(data)
    // 将this.$data上的数据绑定到this上
    for(let name in data){
        Object.defineProperty(this, name, {
            enumerable: true, //不可配置，get/set不能重写
            get(){
                //this.a 获取的时候返回 this.$data.a
                return this.$data[name]
            },
            set(newVal){
                // 设置 this.a = 99 的时候相当于设置 this.$data.a = 99
                this.$data[name] = newVal
            }
        })
    }
    //实现模版编译---可以理解为实现DOM渲染（不准确）
    new Compile(this.$options.el, this)
}

// 实现模版编译 el:当前Vue实例挂载的元素， vm：当前Vue实例上data，已代理到 this.$data
function Compile(el, vm){
    // $el 表示替换的范围
    vm.$el = document.querySelector(el)
    // 创建文档片段----放在内存中
    let fragment = document.createDocumentFragment()
    // 将$el中的内容转移到内存中，遍历
    let child
    while(child = vm.$el.firstChild){ //一直拿第一个节点，直到没有firstChild
        fragment.appendChild(child)
    }
    //替换双大括号{{}}
    replace(fragment)
    function replace(fragment){
        // 把类数组转化成数组
        Array.from(fragment.childNodes).forEach( function (node){
            let text = node.textContent
            // 正则上场，匹配到双大括号
            let reg = /\{\{(.*)\}\}/
            //文本节点并且能匹配上{{}}
            if(node.nodeType === 3 && reg.test(text)){
                // RegExp $1-$9 表示 最后使用的9个正则
                let thisexp = RegExp.$1.trim() // 获得上次正则匹配到的第一个小分组
                let val = vm[thisexp] // 赋值，拿到a，b
                // 监听
                new Watcher(vm, RegExp.$1, function(newVal){
                    node.textContent = text.replace(reg, newVal)
                })
                // 将 {{ a }} 替换为真实的数值
                node.textContent = text.replace(reg, val)
            }
            vm.$el.appendChild(fragment)
            //如果当前节点还有子节点，进行递归操作
            if(node.childNodes){
                replace(node)
            }
        })
    }
}

// 观察数据，给data中的数据object.defineProperty
function observe(data){
    if(typeof data !== 'object'){
        return;
    }
    return new Observe(data)
}

function Observe(data){
    // 开启发布订阅模式
   let dep = new Dep()
   // 在set值的时候触发视图更新 --- 数据劫持
   for(let key in data){
       let val = data[key]
       Object.defineProperty(data, key, {
           enumerable: true,
           configurable: false,
           get(){
               Dep.target && dep.addSub(Dep.target)
               return val
           },
           set(newVal){
               if(newVal === val){
                   return;
               }
               //赋值操作
               val = newVal
               // 实现赋值后的对象监测功能
               // 执行所有用户的 update 方法
               dep.notify()
           }
       })
   }
}

//发布订阅模式
function Dep(){
    this.subs = []
}

//订阅
Dep.prototype.addSub = function(sub){
    this.subs.push(sub)
}

//发布
Dep.prototype.notify = function(sub){
    this.subs.forEach(sub=>{
        sub.update()
    })
}

// watcher ,vm就是当前vue实例,exp就是data中的属性表达式,fn就是传入的回调函数,赋值的时候,值一旦改变就要执行的方法
function Watcher(vm, exp, fn){
    this.vm = vm
    this.exp = exp
    this.fn = fn
    //将watch添加到订阅中
    Dep.target = this
    let val = this.vm[exp]
}

Watcher.prototype.update= function(){
    let val = this.vm[this.exp]
    //需要传入newVal
    this.fn(val)
}
```

