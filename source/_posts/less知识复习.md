---
title: Less知识复习
date: 2020-05-29 10:44:39
tags: less
---

Sass和Less都属于Css预处理器，但是说来惭愧，使用了这么久用的最多的特性只有他们的嵌套，变量之类的都比较少用。

正好最近我在开发公司内部使用的组件，有些组件会根据传入的参数不同，显示不同的颜色。

```tsx
...

<Notice type='success'> // 展示为绿色
<Notice type='warn'> // 展示为橙色
...
```

那么组件内部实现应该类似于这样

```tsx
// Notice.js
const Notice = ({type}) => {
  return <div classname={`notice-${type}`}>
  ...
  </div>
}

```

接着开始写less文件,希望最终的生成结果应该是这样：

```less
.notice-success {
  color: green
}

.notice-warn {
  color: red
}

```
然后去温习一遍less变量和遍历的使用：
```less
// 官方文档
.generate-columns(4);

.generate-columns(@n, @i: 1) when (@i =< @n) {
  .column-@{i} {
    width: (@i * 100% / @n);
  }
  .generate-columns(@n, (@i + 1));
}

// 生成的css
.column-1 {
  width: 25%;
}
.column-2 {
  width: 50%;
}
.column-3 {
  width: 75%;
}
.column-4 {
  width: 100%;
}
```

然后着手开发：

```less

// 定义变量：
@types: warn, info, success, error;
@backgroudColors: #fff5e6, #f0f5ff, #effae4, #ffeae6;

// 遍历：
.loop(@n, @i: 1) when (@i < @n) {
  .NoticeBar-@{extract(@types, @i)} { // warn：less编译错误
    background-color: extract(@backgroudColors, @i);
    border: 1px solid extract(@borderColors, @i);
  }

  .loop(@n, (@i + 1));
}

.loop(5);


```

我反复排查了less的语法，发现文档上类名中使用变量就是这么写的（less版本：3.10），但是编译报错，我在网上查了很久没有查出原因，后来尝试重新定义了一个变量解决，很简单的解决办法，但是也花了很久排查，算踩了一个小坑，记录一下，修改后的代码

```less
.loop(@n, @i: 1) when (@i < @n) {
    @name: extract(@types, @i);

  .NoticeBar-@{name} {
    background-color: extract(@backgroudColors, @i);
    border: 1px solid extract(@borderColors, @i);
  }

  .loop(@n, (@i + 1));
}
```