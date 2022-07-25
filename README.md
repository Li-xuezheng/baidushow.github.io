百度前端训练营结课大作业  
姓名：李学正 学校：南京大学 专业：软件工程 年级：大二 qq：1597286793  


## 简易MVVM实现

这里实现过程包括如下几个步骤：

- MVVM的实现演示
- MVVM的流程设计
- 中介者模式的实现
- 数据劫持的实现
- 数据双向绑定的实现
- 简易视图指令的编译过程的实现
- ViewModel的实现
- MVVM的实现


### MVVM的实现演示


MVVM示例的使用如下所示，包括`browser.js`(View视图的更新)、`mediator.js`(中介者)、`binder.js`(MVVM的数据绑定引擎)、`view.js`(视图)、`hijack.js`(数据劫持)以及`mvvm.js`(MVVM实例)。

``` javascript
<div id="app">
 <input type="text" b-value="input.message" b-on-input="handlerInput">
 <div>{{ input.message }}</div>
 <div b-text="text"></div>
 <div>{{ text }}</div>
 <div b-html="htmlMessage"></div>
</div>

<script src="./browser.js"></script>
<script src="./mediator.js"></script>
<script src="./binder.js"></script>
<script src="./view.js"></script>
<script src="./hijack.js"></script>
<script src="./mvvm.js"></script>


<script>
 let vm = new Mvvm({
    el: '#app',
    data: {
      input: {
        message: 'Hello Input!'
      },
      text: 'ziyi2',
      htmlMessage: `<button>提交</button>`
    },
    methods: {
      handlerInput(e) {
        this.text = e.target.value
      }
    }
  })
</script>
```








### MVVM的流程设计



这里简单的描述一下MVVM实现的运行机制。

#### 初始化流程

- 创建MVVM实例对象，初始化实例对象的`options`参数
- `proxyData`将MVVM实例对象的`data`数据代理到MVVM实例对象上
- `Hijack`类实现数据劫持功能（对MVVM实例跟视图对应的响应式数据进行监听，这里和Vue运行机制不同，干掉了`getter`依赖搜集功能）
- 解析视图指令，对MVVM实例与视图关联的DOM元素转化成文档碎片并进行绑定指令解析（`b-value`、`b-on-input`、`b-html`等，其实是Vue编译的超级简化版），
- 添加数据订阅和用户监听事件，将视图指令对应的数据挂载到**Binder**数据绑定引擎上（数据变化时通过Pub/Sub模式通知**Binder**绑定器更新视图）
- 使用Pub/Sub模式代替Vue中的Observer模式
- **Binder**采用了命令模式解析视图指令，调用`update`方法对View解析绑定指令后的文档碎片进行更新视图处理
- `Browser`采用了外观模式对浏览器进行了简单的兼容性处理


#### 响应式流程


- 监听用户输入事件，对用户的输入事件进行监听
- 调用MVVM实例对象的数据设置方法更新数据
- 数据劫持触发`setter`方法
- 通过发布机制发布数据变化
- 订阅器接收数据变更通知，更新数据对应的视图


### 中介者模式的实现

最简单的中介者模式只需要实现发布、订阅和取消订阅的功能。发布和订阅之间通过事件通道（channels）进行信息传递，可以避免观察者模式中产生依赖的情况。中介者模式的代码如下：

``` javascript
class Mediator {
  constructor() {
    this.channels = {}
    this.uid = 0
  }

  /** 
   * @Desc:   订阅频道
   * @Parm:   {String} channel 频道
   *          {Function} cb 回调函数 
   */  
  sub(channel, cb) {
    let { channels } = this
    if(!channels[channel]) channels[channel] = []
    this.uid ++ 
    channels[channel].push({
      context: this,
      uid: this.uid,
      cb
    })
    console.info('[mediator][sub] -> this.channels: ', this.channels)
    return this.uid
  }

  /** 
   * @Desc:   发布频道 
   * @Parm:   {String} channel 频道
   *          {Any} data 数据 
   */  
  pub(channel, data) {
    console.info('[mediator][pub] -> chanel: ', channel)
    let ch = this.channels[channel]
    if(!ch) return false
    let len = ch.length
    // 后订阅先触发
    while(len --) {
      ch[len].cb.call(ch[len].context, data)
    }
    return this
  }

  /** 
   * @Desc:   取消订阅  
   * @Parm:   {String} uid 订阅标识 
   */  
  cancel(uid) {
    let { channels } = this
    for(let channel of Object.keys(channels)) {
      let ch = channels[channel]
      if(ch.length === 1 && ch[0].uid === uid) {
        delete channels[channel]
        console.info('[mediator][cancel][delete] -> chanel: ', channel)
        console.info('[mediator][cancel] -> chanels: ', channels)
        return
      }
      for(let i=0,len=ch.length; i<len; i++) {
          if(ch[i].uid === uid) {
            ch.splice(i,1)
            console.info('[mediator][cancel][splice] -> chanel: ', channel)
            console.info('[mediator][cancel] -> chanels: ', channels)
            return
          }
      }
    }
  }
}
```

在每一个MVVM实例中，都需要实例化一个中介者实例对象，中介者实例对象的使用方法如下：

``` javascript
let mediator = new Mediator()
// 订阅channel1
let channel1First = mediator.sub('channel1', (data) => {
  console.info('[mediator][channel1First][callback] -> data', data)
})
// 再次订阅channel1
let channel1Second = mediator.sub('channel1', (data) => {
  console.info('[mediator][channel1Second][callback] -> data', data)
})
// 订阅channel2
let channel2 = mediator.sub('channel2', (data) => {
  console.info('[mediator][channel2][callback] -> data', data)
})
// 发布(广播)channel1,此时订阅channel1的两个回调函数会连续执行
mediator.pub('channel1', { name: 'ziyi1' })
// 发布(广播)channel2，此时订阅channel2的回调函数执行
mediator.pub('channel2', { name: 'ziyi2' })
// 取消channel1标识为channel1Second的订阅
mediator.cancel(channel1Second)
// 此时只会执行channel1中标识为channel1First的回调函数
mediator.pub('channel1', { name: 'ziyi1' })
```


### 数据劫持的实现

#### 对象的属性

对象的属性可分为数据属性（特性包括`[[Value]]`、`[[Writable]]`、`[[Enumerable]]`、`[[Configurable]]`）和存储器/访问器属性（特性包括`[[ Get ]]`、`[[ Set ]]`、`[[Enumerable]]`、`[[Configurable]]`），对象的属性只能是数据属性或访问器属性的其中一种，这些属性的含义：

- `[[Configurable]]`: 表示能否通过 `delete` 删除属性从而重新定义属性，能否修改属性的特性，或者能否把属性修改为访问器属性。
- `[[Enumerable]]`:  对象属性的可枚举性。
- `[[Value]]`: 属性的值，读取属性值的时候，从这个位置读；写入属性值的时候，把新值保存在这个位置。这个特性的默认值为 `undefined`。
- `[[Writable]]`: 表示能否修改属性的值。
- `[[ Get ]]`: 在读取属性时调用的函数。默认值为 `undefined`。
- `[[ Set ]]`: 在写入属性时调用的函数。默认值为 `undefined`。

> 数据劫持就是使用了`[[ Get ]]`和`[[ Set ]]`的特性，在访问对象的属性和写入对象的属性时能够自动触发属性特性的调用函数，从而做到监听数据变化的目的。

对象的属性可以通过ES5的设置特性方法`Object.defineProperty(data, key, descriptor)`改变属性的特性，其中`descriptor`传入的就是以上所描述的特性集合。

#### 数据劫持

``` javascript
let hijack = (data) => {
  if(typeof data !== 'object') return
  for(let key of Object.keys(data)) {
    let val = data[key]
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: false,
      get() {
        console.info('[hijack][get] -> val: ', val)
        // 和执行 return data[key] 有什么区别 ？
        return val
      },
      set(newVal) {
        if(newVal === val) return
        console.info('[hijack][set] -> newVal: ', newVal)
        val = newVal
        // 如果新值是object, 则对其属性劫持
        hijack(newVal)
      }
    })
  }
}


```



### 数据双向绑定的实现



数据双向绑定主要包括数据的变化引起视图的变化（**Model** -> 监听数据变化 -> **View**）、视图的变化又改变数据（**View** -> 用户输入监听事件 -> **Model**），从而实现数据和视图之间的强联系。

在实现了数据监听的基础上，加上用户输入事件以及视图更新，就可以简单实现数据的双向绑定（其实就是一个最简单的**Binder**，只是这里的代码耦合严重）：


``` htmlbars
<input id="input" type="text">
<div id="div"></div>
```

``` javascript
// 监听数据变化
function hijack(data) {
  if(typeof data !== 'object') return
  for(let key of Object.keys(data)) {
    let val = data[key]
    Object.defineProperty(data, key, {
      enumerable: true,
      configurable: false,
      get() {
        console.log('[hijack][get] -> val: ', val)
        // 和执行 return data[key] 有什么区别 ？
        return val
      },
      set(newVal) {
        if(newVal === val) return
        console.log('[hijack][set] -> newVal: ', newVal)
        val = newVal
        
        // 更新所有和data.input数据相关联的视图
        input.value = newVal
        div.innerHTML = newVal

        // 如果新值是object, 则对其属性劫持
        hijack(newVal)
      }
    })
  }
}

let input = document.getElementById('input')
let div = document.getElementById('div')

// model
let data = { input: '' }

// 数据劫持
hijack(data)

// model -> view
data.input = '11111112221'

// view -> model
input.oninput = function(e) {
  // model -> view
  data.input = e.target.value
}
```



### 简易视图指令的编译过程实现

在MVVM的实现演示中，可以发现使用了`b-value`、`b-text`、`b-on-input`、`b-html`等绑定属性（这些属性在该MVVM示例中自行定义的，并不是html标签原生的属性，类似于vue的`v-html`、`v-model`、`v-text`指令等），这些指令只是方便用户进行Model和View的同步绑定操作而创建的，需要MVVM实例对象去识别这些指令并重新渲染出最终需要的DOM元素，例如

``` javascript
<div id="app">
  <input type="text" b-value="message">
</div>
```

最终需要转化成真实的DOM

``` javascript
<div id="app">
  <input type="text" value='Hello World' />
</div>
```

那么实现以上指令解析的步骤主要如下：

- 获取对应的`#app`元素
- 转换成文档碎片（从DOM中移出`#app`下的所有子元素）
- 识别出文档碎片中的绑定指令并重新修改该指令对应的DOM元素
- 处理完文档碎片后重新渲染`#app`元素

HTML代码如下：

``` htmlbars
<div id="app">
 <input type="text" b-value="message" />
 <input type="text" b-value="message" />
 <input type="text" b-value="message" />
</div>

<script src="./browser.js"></script>
<script src="./binder.js"></script>
<script src="./view.js"></script>
```

在`view.js`中实现了`#app`下的元素转化成文档碎片以及对所有子元素进行属性遍历操作（用于`binder.js`的绑定属性解析）

``` javascript
class View {
  constructor(el, model) {
    this.model = model
    // 获取需要处理的node节点
    this.el = el.nodeType === Node.ELEMENT_NODE ? el : document.querySelector(el)
    if(!this.el) return
    // 将已有的el元素的所有子元素转成文档碎片
    this.fragment = this.node2Fragment(this.el)
    // 解析和处理绑定指令并修改文档碎片
    this.parseFragment(this.fragment)
    // 将文档碎片重新添加到dom树
    this.el.appendChild(this.fragment)
  }

  /** 
   * @Desc:   将node节点转为文档碎片 
   * @Parm:   {Object} node Node节点 
   */  
  node2Fragment(node) {
    let fragment = document.createDocumentFragment(),
        child;
    while(child = node.firstChild) {
      // 给文档碎片添加节点时，该节点会自动从dom中删除
      fragment.appendChild(child)
    }    
    return fragment
  }

  /** 
   * @Desc:   解析文档碎片(在parseFragment中遍历的属性，需要在binder.parse中处理绑定指令的解析处理) 
   * @Parm:   {Object} fragment 文档碎片 
   */  
  parseFragment(fragment) {
    // 类数组转化成数组进行遍历
    for(let node of [].slice.call(fragment.childNodes)) {
      if(node.nodeType !== Node.ELEMENT_NODE) continue
      // 绑定视图指令解析
      for(let attr of [].slice.call(node.attributes)) {
        binder.parse(node, attr, this.model)
        // 移除绑定属性
        node.removeAttribute(attr.name)
      }
      // 遍历node节点树
      if(node.childNodes && node.childNodes.length) this.parseFragment(node)
    }
  }
}
```


接下来查看`binder.js`如何处理绑定指令，这里以`b-value`的解析为示例

``` javascript
(function(window, browser){
  window.binder = {
    /** 
     * @Desc:   判断是否是绑定属性 
     * @Parm:   {String} attr Node节点的属性 
     */  
    is(attr) {
      return attr.includes('b-')
    },
    /** 
     * @Desc:   解析绑定指令
     * @Parm:   {Object} attr html属性对象
     *          {Object} node Node节点
     *          {Object} model 数据
     */  
    parse(node, attr, model) {
	  // 判断是否是绑定指令，不是则不对该属性进行处理
      if(!this.is(attr.name)) return
      // 获取model数据
      this.model = model 
      // b-value = 'message'， 因此attr.value = 'message'
      let bindValue = attr.value,
	      // 'b-value'.substring(2) = value
          bindType = attr.name.substring(2)
      // 绑定视图指令b-value处理
      // 这里采用了命令模式
      this[bindType](node, bindValue.trim())
    },
    /** 
     * @Desc:   值绑定处理(b-value)
     * @Parm:   {Object} node Node节点
     *          {String} key model的属性
     */  
    value(node, key) {
      this.update(node, key)
    },
    /** 
     * @Desc:   值绑定更新(b-value)
     * @Parm:   {Object} node Node节点
     *          {String} key model的属性
     */  
    update(node, key) {
	  // this.model.getData是用于获取model对象的属性值
	  // 例如 model = { a : { b : 111 } }
	  // <input type="text" b-value="a.b" />
	  // this.model.getData('a.b') = 111
	  // 从而可以将input元素更新为<input type="text" value="111" />
	  browser.val(node, this.model.getData(key))
    }
  }
})(window, browser)
```


```javascript
let browser = {
  /** 
   * @Desc:   Node节点的value处理 
   * @Parm:   {Object} node Node节点   
   *          {String} val 节点的值
   */  
  val(node, val) {
	// 将b-value转化成value，需要注意的是解析完后在view.js中会将b-value属性移除
    node.value = val || ''
    console.info(`[browser][val] -> node: `, node)
    console.info(`[browser][val] -> val: `, val)
  }
}
```


### ViewModel的实现

**ViewModel**(内部绑定器**Binder**)的作用不仅仅是实现了**Model**到**View**的自动同步（Sync Logic）逻辑（以上视图绑定指令的解析的实现只是实现了一个视图的绑定指令初始化，一旦**Model**变化，视图要更新的功能并没有实现），还实现了**View**到**Model**的自动同步逻辑，从而最终实现了数据的双向绑定。



因此只要在视图绑定指令的解析的基础上增加**Model**的数据监听功能（数据变化更新视图）和**View**视图的`input`事件监听功能（监听视图从而更新相应的**Model**数据，注意**Model**的变化又会因为数据监听从而更新和**Model**相关的视图）就可以实现**View**和**Model**的双向绑定。同时需要注意的是，数据变化更新视图的过程需要使用发布/订阅模式。


在**简易视图指令的编译过程实现**的基础上进行修改，首先是HTML代码

``` htmlbars
<div id="app">
 <input type="text" id="input1" b-value="message">
 <input type="text" id="input2" b-value="message">
 <input type="text" id="input3" b-value="message">
</div>

<!-- 新增中介者 -->
<script src="./mediator.js"></script>
<!-- 新增数据劫持 -->
<script src="./hijack.js"></script>
<script src="./view.js"></script>
<script src="./browser.js"></script>
<script src="./binder.js"></script>
```

>`mediator.js`不再叙述，具体回看**中介者模式的实现**，`view.js`和`browser.js`也不再叙述，具体回看**简易视图指令的编译过程实现**。




首先看下数据劫持，在** 数据劫持的实现**的基础上，增加了中介者对象的发布数据变化功能（在抽象视图的**Binder**中会订阅这个数据变化）

``` javascript
var hijack = (function() {

  class Hijack {
    /** 
     * @Desc:   数据劫持构造函数
     * @Parm:   {Object} model 数据 
     *          {Object} mediator 发布订阅对象 
     */  
    constructor(model, mediator) {
      this.model = model
      this.mediator = mediator
    }
  
    /** 
     * @Desc:   model数据劫持
     * @Parm:   
     *          
     */  
    hijackData() {
      let { model, mediator } = this
      for(let key of Object.keys(model)) {
        let val = model[key]
        Object.defineProperty(model, key, {
          enumerable: true,
          configurable: false,
          get() {
            return val
          },
          set(newVal) {
            if(newVal === val) return
            val = newVal
            // 发布数据劫持的数据变化信息
            console.log('[mediator][pub] -> key: ', key)
            mediator.pub(key)
          }
        })
      }
    }
  }

  return (model, mediator) => {
    if(!model || typeof model !== 'object') return
    new Hijack(model, mediator).hijackData()
  }
})()
```

接着重点来看`binder.js`中的实现

``` javascript
(function(window, browser){
  window.binder = {
    /** 
     * @Desc:   判断是否是绑定属性 
     * @Parm:   {String} attr Node节点的属性 
     */  
    is(attr) {
      return attr.includes('b-')
    },

    /** 
     * @Desc:   解析绑定指令
     * @Parm:   {Object} attr html属性对象
     *          {Object} node Node节点
     *          {Object} model 数据
     *          {Object} mediator 中介者
     */  
    parse(node, attr, model, mediator) {
      if(!this.is(attr.name)) return
      this.model = model 
      this.mediator = mediator
      let bindValue = attr.value,
          bindType = attr.name.substring(2)
      // 绑定视图指令处理
      this[bindType](node, bindValue.trim())
    },
    
    /** 
     * @Desc:   值绑定处理(b-value)
     * @Parm:   {Object} node Node节点
     *          {String} key model的属性
     */  
    value(node, key) {
      this.update(node, key)
      // View -> ViewModel -> Model
      // 监听用户的输入事件
      browser.event.add(node, 'input', (e) => {
        // 更新model
        let newVal = browser.event.target(e).value
        // 设置对应的model数据(因为进行了hijack(model))
        // 因为进行了hijack(model)，对model进行了变化监听，因此会触发hijack中的set，从而触发set中的mediator.pub
        this.model.setData(key, newVal)
      })

	  // 一旦model变化，数据劫持会mediator.pub变化的数据		
      // 订阅数据变化更新视图(闭包)
      this.mediator.sub(key, () => {
        console.log('[mediator][sub] -> key: ', key)
        console.log('[mediator][sub] -> node: ', node)
        this.update(node, key)
      })
    },
    
    /** 
     * @Desc:   值绑定更新(b-value)
     * @Parm:   {Object} node Node节点
     *          {String} key model的属性
     */  
    update(node, key) {
      browser.val(node, this.model.getData(key))
    }
  }
})(window, browser)
```



### MVVM的实现

在**ViewModel的实现**的基础上：

- 新增了`b-text`、`b-html`、`b-on-*`(事件监听)指令的解析
- 代码封装更优雅，新增了MVVM类用于约束管理之前示例中零散的实例对象（建造者模式）
- `hijack.js`实现了对**Model**数据的深层次监听
- `hijack.js`中的发布和订阅的`channel`采用HTML属性中绑定的指令对应的值进行处理(例如`b-value="a.b.c.d"`，那么`channel`就是`'a.b.c.d'`，这里是将Vue的观察者模式改成中介者模式后的一种尝试，只是一种实现方式，当然采用观察者模式关联性更强，而采用中介者模式会更解耦)。
- `browser.js`中新增了事件监听的兼容处理、`b-html`和`b-text`等指令的DOM操作api等
