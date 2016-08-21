---
title: 一个小巧的Typing库
date: 2016-08-21 11:05:30
tags: 
- JavaScript
- Typing
---
看到了一个站点：{% link Anyway.FM http://anyway.fm/ %}，发现其顶部的打字动画非常有意思，寻思着怎样实现一个 Typing 库。

# Typing 库的构思

## 定位

Typing 是用于模拟打字动作的一个类库

## 功能

打字一般是这样一个过程：输入文本 - 停顿 - 退格（删除） - 继续输入 。。。
那么 Typing 库应该包含如下功能：
- 将一段文本的内容以字符为单位逐一显示出来
- 可以模拟打字光标
- 可以退格
- 可以控制打字的速度
- 可以模拟打字停顿

# Typing 库的实现

## 尝试

我们先来实现一个文本逐一显示的小功能。
这个功能并不复杂，可以给方法传入一个dom元素和一个字符串，方法中将字符串分割为一个字符数组，并使用`setInterval`来把一个一个字符逐一显示在元素中：
```JavaScript
function typing (el, str) {
  var chars = str.split('');
  var i = 0;
  var timer = setInterval(function () {
    el.innerText = el.innerText + chars[i++];
    if (i === chars.length) {
      clearInterval(timer);
    }
  }, 200)
}
```

看一下demo：
{% iframe //codepen.io/awaw00/embed/PzggGB/?height=265&theme-id=dark&default-tab=js,result&embed-version=2 %}

吼吼~ 文字以200毫秒的速度打印出来了！

## 设计

正如上文中说的，打字是一个过程，我们的库应该能够像流水线一样模拟打字的每个步骤：
```JavaScript
typing.play('我是一句文字。'); // 打字
typing.wait(1000);            // 停顿
typing.play('错误的文字');     // 打字
typing.back(5);               // 退格
typing.wait(1000);            // 思考
typing.play('正确的文字');     // 继续打字
```

还可以更优雅一点：
```JavaScript
typing
  .play('我是一句文字。')
  .wait(1000)
  .play(...)
  ...
```

一个typing应该是针对一个dom元素进行一系列的模拟操作，我们可以设计一个Typing类，一个typing就是一个Typing类的实例对象，每个对象都保存它所操作的dom节点、默认速度等这些值。
此外，每个步骤可以指定模拟操作的速度，让模拟过程更加自然。

## 具体实现
```JavaScript
(function (global) {
  //=================== 类的私有方法 ===================

  // 更新内容
  function updateContent (content, renderCursor) {
    if (typeof renderCursor === 'undefined') {
      renderCursor = true
    }
    this.renderedText = content;
    if (renderCursor) {
      content += this.cursorHTML;
    }
    this.el.innerHTML = content;
  }

  // 等待
  function wait(time) {
    var that = this;
    setTimeout(function() {
      nextTask.call(that);
    }, time)
  }

  // 退格
  function back(count, speed) {
    if (count === 0) {
      count = this.renderedText.length
    } else {
      count = this.renderedText.length > count ? count : this.renderedText.length;
    }
    var that = this;
    var i = 0;
    var timer = setInterval(function() {
      updateContent.call(that, that.renderedText.substring(0, that.renderedText.length - 1));
      i++;
      if (i == count) {
        clearInterval(timer);
        nextTask.call(that);
      }
    }, speed || that.speed)
  }

  // 打字
  function play(text, speed) {
    var chars = text.split('');
    var i = 0;
    var that = this;
    var timer = setInterval(function() {
      updateContent.call(that, that.renderedText + chars[i++]);
      if (i === chars.length) {
        clearInterval(timer);
        nextTask.call(that);
      }
    }, speed || that.speed)
  }

  /*
   * 为了实现按序完成每个步骤，我们需要维护一个任务列表
   * 在实例对象调用 play wait 等方法时，会把操作的名称、参数保存到 tasks 数组中
   * 执行任务时，根据操作名称与参数调用真正的方法来完成模拟操作
  */
  // 添加任务
  function addTask(name, args) {
    this.tasks.push({name: name, args: args});
    if (this.tasks.length === 1) {
      invokeTask.call(this, this.tasks[0]);
    }
  }
  // 执行任务
  function invokeTask(task) {
    var name = task.name;
    var args = task.args;
    switch (name) {
      case 'play': 
        play.apply(this, args);
        break;
      case 'back':
        back.apply(this, args);
        break;
      case 'wait':
        wait.apply(this, args);
        break;
    }
  }
  // 执行下一个任务
  function nextTask() {
    this.tasks.splice(0, 1);
    if (this.tasks.length === 0) {
      return;
    }
    invokeTask.call(this, this.tasks[0]);
  }
  
  // 构造函数
  function Typing (el, speed) {
    this.el = el;
    this.speed = speed || 130;
    this.renderedText = '';
    this.cursorHTML = '<span class="cursor" />';
    this.tasks = [];
  }

  // play 方法，本质是在任务列表添加了一个任务
  Typing.prototype.play = function (text, speed) {
    addTask.call(this, 'play', arguments);
    return this;
  }
  // back 方法，同上
  Typing.prototype.back = function (count, speed) {
    addTask.call(this, 'back', arguments);
    return this;
  }
  // wait 方法，同上
  Typing.prototype.wait = function (time) {
    addTask.call(this, 'wait', arguments)
    return this;
  }
  // 暴露 Typing 类到全局环境
  global.Typing = Typing;
})(window)
```

最后来试一试吧~~
```JavaScript
var typing = new Typing(document.getElementById('main'));
typing
  .play('大家好，我是王小贱', 130)
  .back(3, 300)
  .play('王一韬', 200)
  .play('~~~', 100)
  .wait(1000)
  .back(0, 100)
  .play('我要，', 200)
  .wait(1200)
  .play('用JavaScript，', 100)
  .wait(200)
  .play('赚点零花钱。', 120)
  .back(6, 80)
  .wait(1000)
  .play('改变世界', 500)
  .play('!!!', 100)
```

{% iframe //codepen.io/awaw00/embed/EyJOmB/?height=265&theme-id=dark&default-tab=js,result&embed-version=2 %}

大功告成！！！
