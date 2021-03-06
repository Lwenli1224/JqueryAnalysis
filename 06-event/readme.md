### 由 JS 事件引入

事件是 JS DOM 中极具活力的内容，你可以随时监听 DOM 的变化，并对它们及时的做出反应，如果你不是太懂 JS 中的事件，建议你先去看一些相关介绍的文章，直接看 jQuery 中的事件委托头会头大的。

事件的处理顺序由两个环节，一个是捕获环节，一个是冒泡环节，借用别人的一张图：
![event](https://github.com/moveondo/JqueryLearn/blob/master/06-event/event.jpg)  

事件冒泡与捕获

如果把处理也算进的话，整个事件分为三个阶段，分别是捕获阶段，目标处理阶段和冒泡阶段。捕获阶段由外向内寻找 target，冒泡阶段由内向外直到根结点。这只是一个事件，当这三个阶段中又穿插着更多的事件时，还需要将事件的执行顺序考虑进去。

而 jQuery 事件委托的概念：事件目标自身不处理事件，而是将其委托给父元素或祖先元素或根元素，而借助事件的冒泡性质（由内向外）来达到最终处理事件。

jQuery 中的事件优化

首先必须要知道，绑定事件越多，浏览器内存占用越大，就会间接的影响性能。而且一旦出现 ajax，局部刷新导致重新绑定事件。

使用事件委托可以解决以上带来的问题，借助事件的冒泡，尤其当一个父元素的子元素过多，而且子元素绑定的事件非常多时，委托事件的作用就体现出来了。



在早期的 jQuery 版本，使用的是 .delegate()、.bind()、.live()等方法来实现事件监听，当然也包括.click()方法，随着 jQuery 的发展，像 live 方法已经明确从 jQuery 中删除，而其余的方法，比如 bind 方法也将在 3.0 之后的版本陆续删除，取而代之的是 .on()方法。而且剩下的其它方法都是通过 on 方法来间接实现的，如果介绍，只需要看 on 的源码即可。

on 函数在 jQuery 中的用法也很简单，.on( events [, selector ] [, data ], handler(eventObject) )events 表示绑定的事件，比如 "click" 或 "click mouseleave"，selector 和 data 是可选的，分别表示要绑定事件的元素和要执行的数据，handler 表示事件执行函数。

off 函数的用法 .off( events [, selector ] [, handler ] )，events 代表要移除的事件，selector 表示选择的 dom，handler 表示事件处理函数。还有更残暴的比如 .off()不接受任何参数，表示着移除所有 on 绑定的函数。

on off 函数源码

虽然我分析的源码时 jQuery 3.1.1，但这个时候 bind 和 delegate 函数并没有从源码中移除呢，先来看看它们怎么调用 on：
```
jQuery.fn.extend( {
  bind: function( types, data, fn ) {
    return this.on( types, null, data, fn );
  },
  unbind: function( types, fn ) {
    return this.off( types, null, fn );
  },
  delegate: function( selector, types, data, fn ) {
    return this.on( types, selector, data, fn );
  },
  undelegate: function( selector, types, fn ) {
    // ( namespace ) or ( selector, types [, fn] )
    return arguments.length === 1 ?
      this.off( selector, "**" ) :
      this.off( types, selector || "**", fn );
  }
} );
```
可以看得出来，全都被 on 和 off 这两个函数来处理了。
```
jQuery.fn.extend( {
  on: function (types, selector, data, fn) {
    // on 又依托于全局的 on 函数
    return on(this, types, selector, data, fn);
  }
} );
function on( elem, types, selector, data, fn, one ) {
  var origFn, type;

  // 支持 object 的情况
  if ( typeof types === "object" ) {

    // ( types-Object, selector, data )
    if ( typeof selector !== "string" ) {

      // ( types-Object, data )
      data = data || selector;
      selector = undefined;
    }
    // 一次执行 object 的每一个
    for ( type in types ) {
      on( elem, type, selector, data, types[ type ], one );
    }
    return elem;
  }
  // 参数为两个的情况
  if ( data == null && fn == null ) {

    // ( types, fn )
    fn = selector;
    data = selector = undefined;
  } else if ( fn == null ) {
    if ( typeof selector === "string" ) {

      // ( types, selector, fn )
      fn = data;
      data = undefined;
    } else {

      // ( types, data, fn )
      fn = data;
      data = selector;
      selector = undefined;
    }
  }
  if ( fn === false ) {
    // returnFalse 是一个返回 false 的函数
    fn = returnFalse;
  } else if ( !fn ) {
    return elem;
  }

  if ( one === 1 ) {
    origFn = fn;
    fn = function( event ) {

      // Can use an empty set, since event contains the info
      jQuery().off( event );
      return origFn.apply( this, arguments );
    };

    // Use same guid so caller can remove using origFn
    fn.guid = origFn.guid || ( origFn.guid = jQuery.guid++ );
  }
  return elem.each( function() {
    // 关键
    jQuery.event.add( this, types, fn, data, selector );
  } );
}
```
是的，你没有看错，这个全局的 on 函数，其实只是起到了校正参数的作用，而真正的大头是：
```
jQuery.event = {
  global = {},
  add: function(){...},
  remove: function(){...},
  dispatch: function(){...},
  handlers: function(){...},
  addProp: function(){...},
  fix: function(){...},
  special: function(){...}
}
off 函数：

jQuery.fn.off = function (types, selector, fn) {
  var handleObj, type;
  if (types && types.preventDefault && types.handleObj) {
    // ( event )  dispatched jQuery.Event
    handleObj = types.handleObj;
    jQuery(types.delegateTarget).off(
      handleObj.namespace ? handleObj.origType + "." + handleObj.namespace : handleObj.origType,
      handleObj.selector,
      handleObj.handler
    );
    return this;
  }
  if (typeof types === "object") {
    // ( types-object [, selector] )
    for (type in types) {
      this.off(type, selector, types[type]);
    }
    return this;
  }
  if (selector === false || typeof selector === "function") {
    // ( types [, fn] )
    fn = selector;
    selector = undefined;
  }
  if (fn === false) {
    fn = returnFalse;
  }
  return this.each(function() {
    // 关键
    jQuery.event.remove(this, types, fn, selector);
  });
}
```



可见 jQuery 对于参数的放纵导致其处理起来非常复杂，不过对于使用者来说，却非常大便利。

委托事件也带来了一些不足，比如一些事件无法冒泡，load、submit 等，会加大管理等复杂，不好模拟用户触发事件等。



### 一些遗留问题

前面介绍 bind、delegate 和它们的 un 方法的时候，经提醒，忘记提到一些内容，却是我们经常使用的。比如 $('body').click，$('body').mouseleave等，它们是直接定义在原型上的函数，不知道怎么，就把它们给忽略了。
```
jQuery.each( ( "blur focus focusin focusout resize scroll click dblclick " +
  "mousedown mouseup mousemove mouseover mouseout mouseenter mouseleave " +
  "change select submit keydown keypress keyup contextmenu" ).split( " " ),
  function( i, name ) {

  // Handle event binding
  jQuery.fn[ name ] = function( data, fn ) {
    return arguments.length > 0 ?
      this.on( name, null, data, fn ) :
      this.trigger( name );
  };
} );
```
这个构造也是十分巧妙的，这些方法组成的字符串通过 split(" ") 变成数组，而后又通过 each 方法，在原型上对应每个名称，定义函数，这里可以看到，依旧是 on，还有 targger：
```
jQuery.fn.extend( {
  trigger: function(type, data){
    return this.each(function (){
      // // 依旧是 event 对象上的方法
      jQuery.event.trigger(type, data, this);
    })
  }
} )
```
还缺少一个 one 方法，这个方法表示绑定的事件同类型只执行一次，.one()：
```
jQuery.fn.extend( {
  one: function( types, selector, data, fn ) {
    // 全局 on 函数
    return on( this, types, selector, data, fn, 1 );
  },
} );
```
DOM 事件知识点

发现随着 event 源码的不断的深入，我自己出现越来越多的问题，比如没有看到我所熟悉的 addEventListener，还有一些看得很迷糊的 events 事件，所以我决定还是先来看懂 JS 中的 DOM 事件吧。

早期 DOM 事件

在 HTML 的 DOM 对象中，有一些以 on 开头的熟悉，比如 onclick、onmouseout 等，这些就是早期的 DOM 事件，它的最简单的用法，就是支持直接在对象上以名称来写函数：
```
document.getElementsByTagName('body')[0].onclick = function(){
  console.log('click!');
}
document.getElementsByTagName('body')[0].onmouseout = function(){
  console.log('mouse out!');
}
```
onclick 函数会默认传入一个 event 参数，表示触发事件时的状态，包括触发对象，坐标等等。

这种方式有一个非常大的弊端，就是相同名称的事件，会前后覆盖，后一个 click 函数会把前一个 click 函数覆盖掉：
```
var body = document.getElementsByTagName('body')[0];
body.onclick = function(){
  console.log('click1');
}
body.onclick = function(){
  console.log('click2');
}
// "click2"
body.onclick = null;
// 没有效果
```
#### DOM 2.0

随着 DOM 的发展，已经来到 2.0 时代，也就是我所熟悉的 addEventListener 和 attachEvent(IE)，JS 中的事件冒泡与捕获。这个时候和之前相比，变化真的是太大了，MDN addEventListener()。

变化虽然是变化了，但是浏览器的兼容却成了一个大问题，比如下面就可以实现不支持 addEventListener 浏览器：
```
(function() {
  // 不支持 preventDefault
  if (!Event.prototype.preventDefault) {
    Event.prototype.preventDefault=function() {
      this.returnValue=false;
    };
  }
  // 不支持 stopPropagation
  if (!Event.prototype.stopPropagation) {
    Event.prototype.stopPropagation=function() {
      this.cancelBubble=true;
    };
  }
  // 不支持 addEventListener 时候
  if (!Element.prototype.addEventListener) {
    var eventListeners=[];
    
    var addEventListener=function(type,listener /*, useCapture (will be ignored) */) {
      var self=this;
      var wrapper=function(e) {
        e.target=e.srcElement;
        e.currentTarget=self;
        if (typeof listener.handleEvent != 'undefined') {
          listener.handleEvent(e);
        } else {
          listener.call(self,e);
        }
      };
      if (type=="DOMContentLoaded") {
        var wrapper2=function(e) {
          if (document.readyState=="complete") {
            wrapper(e);
          }
        };
        document.attachEvent("onreadystatechange",wrapper2);
        eventListeners.push({object:this,type:type,listener:listener,wrapper:wrapper2});
        
        if (document.readyState=="complete") {
          var e=new Event();
          e.srcElement=window;
          wrapper2(e);
        }
      } else {
        this.attachEvent("on"+type,wrapper);
        eventListeners.push({object:this,type:type,listener:listener,wrapper:wrapper});
      }
    };
    var removeEventListener=function(type,listener /*, useCapture (will be ignored) */) {
      var counter=0;
      while (counter<eventListeners.length) {
        var eventListener=eventListeners[counter];
        if (eventListener.object==this && eventListener.type==type && eventListener.listener==listener) {
          if (type=="DOMContentLoaded") {
            this.detachEvent("onreadystatechange",eventListener.wrapper);
          } else {
            this.detachEvent("on"+type,eventListener.wrapper);
          }
          eventListeners.splice(counter, 1);
          break;
        }
        ++counter;
      }
    };
    Element.prototype.addEventListener=addEventListener;
    Element.prototype.removeEventListener=removeEventListener;
    if (HTMLDocument) {
      HTMLDocument.prototype.addEventListener=addEventListener;
      HTMLDocument.prototype.removeEventListener=removeEventListener;
    }
    if (Window) {
      Window.prototype.addEventListener=addEventListener;
      Window.prototype.removeEventListener=removeEventListener;
    }
  }
})();
```
虽然不支持 addEventListener 的浏览器可以实现这个功能，但本质上还是通过 attachEvent 函数来实现的，在理解 DOM 早期的事件如何来建立还是比较捉急的。

#### addEvent 库

addEvent库的这篇博客发表于 2005 年 10 月，所以这篇博客所讲述的 addEvent 方法算是经典型的，就连 jQuery 中的事件方法也是借鉴于此，故值得一提：
```
function addEvent(element, type, handler) {
  // 给每一个要绑定的函数添加一个标识 guid
  if (!handler.$$guid) handler.$$guid = addEvent.guid++;
  // 在绑定的对象事件上创建一个事件对象
  if (!element.events) element.events = {};
  // 一个 type 对应一个 handlers 对象，比如 click 可同时处理多个函数
  var handlers = element.events[type];
  if (!handlers) {
    handlers = element.events[type] = {};
    // 如果 onclick 已经存在一个函数，拿过来
    if (element["on" + type]) {
      handlers[0] = element["on" + type];
    }
  }
  // 防止重复绑定，每个对应一个 guid
  handlers[handler.$$guid] = handler;
  // 把 onclick 函数替换成 handleEvent
  element["on" + type] = handleEvent;
};
// 初始 guid
addEvent.guid = 1;

function removeEvent(element, type, handler) {
  // delete the event handler from the hash table
  if (element.events && element.events[type]) {
    delete element.events[type][handler.$$guid];
  }
  // 感觉后面是不是要加个判断，当 element.events[type] 为空时，一起删了
};

function handleEvent(event) {
  // grab the event object (IE uses a global event object)
  event = event || window.event;
  // 这里的 this 指向 element
  var handlers = this.events[event.type];
  // execute each event handler
  for (var i in handlers) {
    // 这里有个小技巧，为什么不直接执行，而是先绑定到 this 后执行
    // 是为了让函数执行的时候，内部 this 指向 element
    this.$$handleEvent = handlers[i];
    this.$$handleEvent(event);
  }
};
```
如果能将上面 addEvent 库的这些代码看懂，那么在看 jQuery 的 events 源码就明朗多了。

还有一个问题，所谓事件监听，是将事件绑定到父元素或 document 上，子元素来响应，如何实现？

要靠 event 传入的参数 e：
```
var body = document.getElementsByTagName('body')[0];
body.onclick = function(e){
  console.log(e.target.className);
}
```
这个 e.target 对象就是点击的那个子元素了，无论是捕获也好，冒泡也好，貌似都能够模拟出来。接下来，可能要真的步入正题了。


对于 addEvent 库，它的兼容性超级棒，据说对于 IE4、5 都有很好的兼容性，这和 jQuery 的原理是一致的，而在 jQuery 中，有一个对象与其相对于，那就是 event。

前面就已经说过了，这个 jQuery.fn.on这个函数最终是通过 jQuery.event对象的 add 方法来实现功能的，当然，方法不局限于 add，下面就要对这些方法进行详细的介绍。

### 关于 jQuery.event

这是一个在 jQuery 对象上的方法，它做了很多的优化事件，比如兼容性问题，存储优化问题。
```
jQuery.event = {
  global = {},
  add: function(){...},
  remove: function(){...},
  dispatch: function(){...},
  handlers: function(){...},
  addProp: function(){...},
  fix: function(){...},
  special: function(){...}
}
```
上面是 event 上的一些函数，其中：

add() 是添加事件函数，在当前 dom 上监听事件，生成处理函数；
remove() 移除事件；
dispatch() 是实际的事件执行者；
handlers() 在 dispatch 执行的时候，对事件进行校正，区分原生与委托事件；
addProp() 是绑定参数到对象上；
fix() 将原生的 event 事件修复成一个可读可写且有统一接口的对象；
special() 是一个特殊事件表。
说 add 函数之前，还是忍不住要把 Dean Edwards 大神的 addEvent 库来品味一下，尽管之前已经谈过了：
```
function addEvent(element, type, handler) {
  // 给每一个要绑定的函数添加一个标识 guid
  if (!handler.$$guid) handler.$$guid = addEvent.guid++;
  // 在绑定的对象事件上创建一个事件对象
  if (!element.events) element.events = {};
  // 一个 type 对应一个 handlers 对象，比如 click 可同时处理多个函数
  var handlers = element.events[type];
  if (!handlers) {
    handlers = element.events[type] = {};
    // 如果 onclick 已经存在一个函数，拿过来
    if (element["on" + type]) {
      handlers[0] = element["on" + type];
    }
  }
  // 防止重复绑定，每个对应一个 guid
  handlers[handler.$$guid] = handler;
  // 把 onclick 函数替换成 handleEvent
  element["on" + type] = handleEvent;
};
// 初始 guid
addEvent.guid = 1;

function removeEvent(element, type, handler) {
  // delete the event handler from the hash table
  if (element.events && element.events[type]) {
    delete element.events[type][handler.$$guid];
  }
  // 感觉后面是不是要加个判断，当 element.events[type] 为空时，一起删了
};

function handleEvent(event) {
  // grab the event object (IE uses a global event object)
  event = event || window.event;
  // 这里的 this 指向 element
  var handlers = this.events[event.type];
  // execute each event handler
  for (var i in handlers) {
    // 这里有个小技巧，为什么不直接执行，而是先绑定到 this 后执行
    // 是为了让函数执行的时候，内部 this 指向 element
    this.$$handleEvent = handlers[i];
    this.$$handleEvent(event);
  }
};
```
如果我们对这个浏览器支持 addEventListener，我们就可以对 addEvent 函数就行稍微对小修改（暂不考虑 attachEvent 兼容），在 addEvent 函数的最后，如果代码修改成以下：
```
//element["on" + type] = handleEvent;
if(!element.hasAddListener){
  element.addEventListener(type,handleEvent,false);
  // 监听事件只需添加一次就好了
  element['hasAddListener'] = true;
}
```

虽然以上的做法有点重复，需要对基本逻辑进行判断，但这已经非常接近 jQuery 中的事件 add 方法。换句话说，以前的逻辑是把所有的监听方法都通过 addEventListener 来绑定，绑定一个，绑定两个，现在的思路变了：如果我们写一个处理函数(handleEvent)，这个处理函数用来处理绑定到 DOM 上的事件，并通过 addEventListener 添加（只需添加一次），这就是 jQuery 中事件处理的基本逻辑（我所理解的，欢迎指正）。

懂了上面，还需要清楚委托事件的本质：在父 DOM 上监听事件，事件处理函数找到对应的子 DOM 来处理。

jQuery.event.add 函数分析

好了，来直接看源码，我已经不止一次的提到，学习源码最好的方式是调试，调试最好的办法是代码覆盖率 100% 的测试用例。
```
jQuery.event = {
  global: {},
  add: function( elem, types, handler, data, selector ) {

    var handleObjIn, eventHandle, tmp,
      events, t, handleObj,
      special, handlers, type, namespaces, origType,
      // jQuery 专门为事件建立一个 data cache：dataPriv
      elemData = dataPriv.get( elem );

    // elem 有问题，直接退出
    if ( !elemData ) {
      return;
    }

    // 如果 handler 是一个事件处理对象，且有 handler 属性
    if ( handler.handler ) {
      handleObjIn = handler;
      handler = handleObjIn.handler;
      selector = handleObjIn.selector;
    }

    // Ensure that invalid selectors throw exceptions at attach time
    // Evaluate against documentElement in case elem is a non-element node (e.g., document)
    if ( selector ) {
      jQuery.find.matchesSelector( documentElement, selector );
    }

    // guid。
    if ( !handler.guid ) {
      handler.guid = jQuery.guid++;
    }

    // 初始化 data.elem 的 events 和 handle
    if ( !( events = elemData.events ) ) {
      events = elemData.events = {};
    }
    if ( !( eventHandle = elemData.handle ) ) {
      eventHandle = elemData.handle = function( e ) {

        // 最终的执行在这里，点击 click 后，会执行这个函数
        return typeof jQuery !== "undefined" && jQuery.event.triggered !== e.type ?
          jQuery.event.dispatch.apply( elem, arguments ) : undefined;
      };
    }

    // 处理多个事件比如 ("click mouseout")，空格隔开
    types = ( types || "" ).match( rnothtmlwhite ) || [ "" ];
    t = types.length;
    // 每个事件都处理
    while ( t-- ) {
      tmp = rtypenamespace.exec( types[ t ] ) || [];
      type = origType = tmp[ 1 ];
      namespaces = ( tmp[ 2 ] || "" ).split( "." ).sort();

      // There *must* be a type, no attaching namespace-only handlers
      if ( !type ) {
        continue;
      }

      // 特殊处理
      special = jQuery.event.special[ type ] || {};

      // If selector defined, determine special event api type, otherwise given type
      type = ( selector ? special.delegateType : special.bindType ) || type;

      // Update special based on newly reset type
      special = jQuery.event.special[ type ] || {};

      // handleObj is passed to all event handlers
      handleObj = jQuery.extend( {
        type: type,
        origType: origType,
        data: data,
        handler: handler,
        guid: handler.guid,
        selector: selector,
        needsContext: selector && jQuery.expr.match.needsContext.test( selector ),
        namespace: namespaces.join( "." )
      }, handleObjIn );

      // 如果 click 事件之前没有添加过，
      if ( !( handlers = events[ type ] ) ) {
        handlers = events[ type ] = [];
        handlers.delegateCount = 0;

        // Only use addEventListener if the special events handler returns false
        // addEventListener 事件也只是添加一次
        if ( !special.setup ||
          special.setup.call( elem, data, namespaces, eventHandle ) === false ) {

          if ( elem.addEventListener ) {
            // eventHandle 才是事件处理函数
            elem.addEventListener( type, eventHandle );
          }
        }
      }

      if ( special.add ) {
        special.add.call( elem, handleObj );

        if ( !handleObj.handler.guid ) {
          handleObj.handler.guid = handler.guid;
        }
      }

      // 添加到事件列表, delegates 要在前面
      if ( selector ) {
        handlers.splice( handlers.delegateCount++, 0, handleObj );
      } else {
        handlers.push( handleObj );
      }

      // 表面已经添加过了
      jQuery.event.global[ type ] = true;
    }

  }
}
```
来从头理一下代码，之前就已经说过 jQuery 中 Data 的问题，这里有两个：

// 用于 DOM 事件 
var dataPriv = new Data();
// jQuery 中通用
var dataUser = new Data();
dataPriv 会根据当前的 elem 缓存两个对象，分别是 events 和 handle，这个 handle 就是通过 addEventListener 添加的那个回掉函数，而 events 存储的东西较多，比如我绑定了一个 click 事件，则 events['click'] = []，它是一个数组，这个时候无论我绑定多少个点击事件，只需要在这个数组里面添加内容即可，添加的时候要考虑一定的顺序。那么，数组的每个子元素长什么样：
```
handleObj = {
  type: type,
  origType: origType,
  data: data,
  handler: handler,
  guid: handler.guid,
  selector: selector,
  needsContext: selector && jQuery.expr.match.needsContext.test( selector ),
  namespace: namespaces.join( "." )
}
```
兼容性，兼容性，兼容性，其实 add 事件主要还是考虑到很多细节的内容，如果把这些都抛开不开，我们来比较一下 event.add 函数和 addEvent 函数相同点：

// 左边是 addEvent 函数中内容
addEvent == jQuery.event.add;
handleEvent == eventHandle;
handler.$$guid == handler.guid;
element.events == dataPriv[elem].events;
非常像！

jQuery.event.dispatch 函数分析

当有 selector 的时候，add 函数处理添加事件，而事件的执行，要靠 dispatch。举个例子，在 $('body').on('click','#test',fn)，我们点击 body，会被监听，dispatch 函数是会执行的，但是 fn 不执行，除非我们点击 #test。大概 dispath 就是用来判断 event.target 是不是需要的那个。
```
jQuery.event.extend( {
  dispatch: function( nativeEvent ) {

    // 把 nativeEvent 变成可读写，jQuery 认可的 event
    var event = jQuery.event.fix( nativeEvent );

    var i, j, ret, matched, handleObj, handlerQueue,
      args = new Array( arguments.length ),
      // 从 data cache 中搜索处理事件
      handlers = ( dataPriv.get( this, "events" ) || {} )[ event.type ] || [],
      special = jQuery.event.special[ event.type ] || {};

    // Use the fix-ed jQuery.Event rather than the (read-only) native event
    args[ 0 ] = event;

    for ( i = 1; i < arguments.length; i++ ) {
      args[ i ] = arguments[ i ];
    }

    event.delegateTarget = this;

    // Call the preDispatch hook for the mapped type, and let it bail if desired
    if ( special.preDispatch && special.preDispatch.call( this, event ) === false ) {
      return;
    }

    // 对 handlers 处理，区分事件类型，并按照顺序排好
    handlerQueue = jQuery.event.handlers.call( this, event, handlers );

    // Run delegates first; they may want to stop propagation beneath us
    i = 0;
    while ( ( matched = handlerQueue[ i++ ] ) && !event.isPropagationStopped() ) {
      event.currentTarget = matched.elem;

      j = 0;
      // 按照 handlers 排好的顺序，一次执行
      while ( ( handleObj = matched.handlers[ j++ ] ) &&
        !event.isImmediatePropagationStopped() ) {

        // Triggered event must either 1) have no namespace, or 2) have namespace(s)
        // a subset or equal to those in the bound event (both can have no namespace).
        if ( !event.rnamespace || event.rnamespace.test( handleObj.namespace ) ) {

          event.handleObj = handleObj;
          event.data = handleObj.data;
          // 最终的执行在这里
          ret = ( ( jQuery.event.special[ handleObj.origType ] || {} ).handle ||
            handleObj.handler ).apply( matched.elem, args );

          if ( ret !== undefined ) {
            if ( ( event.result = ret ) === false ) {
              // 下面两个函数是经过 event 改造后的事件
              event.preventDefault();
              event.stopPropagation();
            }
          }
        }
      }
    }

    // Call the postDispatch hook for the mapped type
    if ( special.postDispatch ) {
      special.postDispatch.call( this, event );
    }

    return event.result;
  }
} );
```

以前，我只知道，只有当对浏览器中的元素进行点击的时候，才会出发 click 事件，其它的事件也一样，需要人为的鼠标操作。

后来随着学习的不断深入，才知道原来 JS 可以写函数来控制事件的执行，这样子写代码才有意思。记得很久很久以前一些恶意网站，明明鼠标没有点击，却被网站强行的点击了某个链接，大概实现的方式就是这样的吧。

原生事件

其实 JS 的原生事件已经做得挺好了，只是 jQuery 将其进行封装，做的更好。

关于 document.createEvent，下面是一个简单的事件点击的例子：
```
var fn = function(){
  console.log('click');
}

var button = document.getElementById('#id');
button.addEventListener('click', fn);

// 点击事件 MouseEvent
var myClick = document.createEvent('MouseEvent');
myClick.initMouseEvent('click', false, false, null);

// 执行
button.dispatchEvent(myClick); // 'click'
除了鼠标事件，还可以自定义事件：

// 随便自定义一个事件 test.click
button.addEventListener('test.click', fn);

var testEvent = document.createEvent('CustomEvent');
// customEvent 也可以初始化为鼠标事件，不一定非要自定义事件
testEvent.initCustomEvent('test.click', false, false, null);
```
button.dispatchEvent(testEvent); // 'click'
JS 原生的模拟事件，使用起来还是很方便的，以上便是原生的。

不过 jQuery 也有自己的一套自定义事件方案。

jQuery.trigger

jQuery.trigger 可以和 HTMLElement.dispatchEvent 事件拿来对比，他们都是用来模拟和执行监听的事件。

如何使用

关于使用，则比较简单了.trigger():
```
var $body = $(document.body);

// 先绑定事件
$body.on('click', function(){
  console.log('click');
})

// 执行
$body.trigger('click'); //'click'
trigger 还支持更多的参数，同样可以自定义事件：

$body.on('click.test', function(e, data1, data2){
  console.log(data1 + '-' + data2);
})

$body.trigger('click.test', ['hello', 'world']);
trigger 源码

trigger 的源码有些简单，因为还是要借助于 jQuery.event 来处理：

jQuery.fn.extend( {
  trigger: function(type, data){
    return this.each(function(){
      jQuery.event.trigger(type, data, this);
    })
  },
  // triggerHandler 处理第一个且不触发默认事件
  triggerHandler: function( type, data ) {
    var elem = this[ 0 ];
    if ( elem ) {
      return jQuery.event.trigger( type, data, elem, true );
    }
  }
} );
```
所以 trigger 事件的起点又回到了 jQuery.event。

jQuery.event.trigger

其实 trigger 和 add + handler 函数很类似，大致都是从 data cache 中搜索缓存，执行回调函数。需要考虑要不要执行默认事件，即第四个参数为 true 的情况。
```
jQuery.extend(jQuery.event, {
  // onleyHandlers 表示不考虑冒泡事件
  trigger: function( event, data, elem, onlyHandlers ) {

    var i, cur, tmp, bubbleType, ontype, handle, special,
      eventPath = [ elem || document ],
      type = hasOwn.call( event, "type" ) ? event.type : event,
      namespaces = hasOwn.call( event, "namespace" ) ? event.namespace.split( "." ) : [];

    cur = tmp = elem = elem || document;

    // Don't do events on text and comment nodes
    if ( elem.nodeType === 3 || elem.nodeType === 8 ) {
      return;
    }

    // 异步不冲突
    if ( rfocusMorph.test( type + jQuery.event.triggered ) ) {
      return;
    }

    if ( type.indexOf( "." ) > -1 ) {

      // Namespaced trigger; create a regexp to match event type in handle()
      namespaces = type.split( "." );
      type = namespaces.shift();
      namespaces.sort();
    }
    ontype = type.indexOf( ":" ) < 0 && "on" + type;

    // 改装原生的 event 事件
    event = event[ jQuery.expando ] ?
      event :
      new jQuery.Event( type, typeof event === "object" && event );

    // 判断是否只执行当前 trigger 事件，不冒泡
    event.isTrigger = onlyHandlers ? 2 : 3;
    event.namespace = namespaces.join( "." );
    event.rnamespace = event.namespace ?
      new RegExp( "(^|\\.)" + namespaces.join( "\\.(?:.*\\.|)" ) + "(\\.|$)" ) :
      null;

    // Clean up the event in case it is being reused
    event.result = undefined;
    if ( !event.target ) {
      event.target = elem;
    }

    // Clone any incoming data and prepend the event, creating the handler arg list
    data = data == null ?
      [ event ] :
      jQuery.makeArray( data, [ event ] );

    // Allow special events to draw outside the lines
    special = jQuery.event.special[ type ] || {};
    if ( !onlyHandlers && special.trigger && special.trigger.apply( elem, data ) === false ) {
      return;
    }

    // 向 document 冒泡并把冒泡结果存储到 eventPath 数组中
    if ( !onlyHandlers && !special.noBubble && !jQuery.isWindow( elem ) ) {

      bubbleType = special.delegateType || type;
      if ( !rfocusMorph.test( bubbleType + type ) ) {
        cur = cur.parentNode;
      }
      for ( ; cur; cur = cur.parentNode ) {
        eventPath.push( cur );
        tmp = cur;
      }

      // Only add window if we got to document (e.g., not plain obj or detached DOM)
      if ( tmp === ( elem.ownerDocument || document ) ) {
        eventPath.push( tmp.defaultView || tmp.parentWindow || window );
      }
    }

    // 按需求来执行
    i = 0;
    while ( ( cur = eventPath[ i++ ] ) && !event.isPropagationStopped() ) {

      event.type = i > 1 ?
        bubbleType :
        special.bindType || type;

      // 从 data cache 中获得回调函数
      handle = ( dataPriv.get( cur, "events" ) || {} )[ event.type ] &&
        dataPriv.get( cur, "handle" );
      if ( handle ) {
        // 执行
        handle.apply( cur, data );
      }

      // Native handler
      handle = ontype && cur[ ontype ];
      if ( handle && handle.apply && acceptData( cur ) ) {
        event.result = handle.apply( cur, data );
        if ( event.result === false ) {
          event.preventDefault();
        }
      }
    }
    event.type = type;

    // If nobody prevented the default action, do it now
    if ( !onlyHandlers && !event.isDefaultPrevented() ) {

      if ( ( !special._default ||
        special._default.apply( eventPath.pop(), data ) === false ) &&
        acceptData( elem ) ) {

        // Call a native DOM method on the target with the same name as the event.
        // Don't do default actions on window, that's where global variables be (#6170)
        if ( ontype && jQuery.isFunction( elem[ type ] ) && !jQuery.isWindow( elem ) ) {

          // Don't re-trigger an onFOO event when we call its FOO() method
          tmp = elem[ ontype ];

          if ( tmp ) {
            elem[ ontype ] = null;
          }

          // Prevent re-triggering of the same event, since we already bubbled it above
          jQuery.event.triggered = type;
          elem[ type ]();
          jQuery.event.triggered = undefined;

          if ( tmp ) {
            elem[ ontype ] = tmp;
          }
        }
      }
    }

    return event.result;
  },
})
```
总结

在 jQuery.event.trigger 中，比较有意思的是模拟冒泡机制，大致的思路就是：

 * 把当前 elem 存入数组；
 * 查找当前 elem 的父元素，如果符合，push 到数组中，重复第一步，否则下一步；
 * 遍历数组，从 data cache 中查看是否绑定 type 事件，然后依次执行。
 * 冒泡事件就是就是由内向外冒泡的过程，这个过程不是很复杂。




