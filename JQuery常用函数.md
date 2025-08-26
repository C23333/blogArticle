```
title: JQuery常用操作
date: 2022-10-21 11:09:25
tags:
  - JQuery
  - 前端
keywords: JQuery
categories: 理论
```



## JQuery常用操作

### 操作元素属性(class、id等)

下面是获取或设置元素的DOM属性的方法，我只挑出一些和操作class属性相关的写一些小demo，其余的自己直接看教程就行了：[https://www.w3school.com.cn/jquery/jquery_ref_attributes.asp]()

| 方法            | 描述                     |
| ------------- | ---------------------- |
| addClass()    | 向匹配的元素添加指定的类名          |
| attr()        | 设置或返回匹配元素的属性和值         |
| hasClass()    | 检查匹配的元素是否拥有指定的类        |
| html()        | 设置或返回匹配的元素集合中的 HTML 内容 |
| removeAttr()  | 从所有匹配的元素中移除指定的属性       |
| removeClass() | 从所有匹配的元素中删除全部或者指定的类    |
| toggleClass() | 从匹配的元素中添加或删除一个类        |
| val()         | 设置或返回匹配元素的值            |

### 选择器

1. 基本选择器
   
   ```js
   $("#id")                             //ID选择器
   $("div")                             //元素选择器
   $(".classname")                      //类选择器
   $(".classname,.classname1,#id1")     //组合选择器
   ```

2. 层次选择器
   
   ```js
   $("#id>.classname ")      //子元素选择器
   $("#id .classname ")      //后代元素选择器
   $("#id + .classname ")    //紧邻下一个元素选择器
   $("#id ~ .classname ")    //兄弟元素选择器
   ```

3. 过滤选择器
   
   ```js
   $("li:first")    //第一个li
   $("li:last")     //最后一个li
   $("li:even")     //挑选下标为偶数的li
   $("li:odd")      //挑选下标为奇数的li
   $("li:eq(4)")    //下标等于 4 的li(第五个 li 元素)
   $("li:gt(2)")    //下标大于 2 的li
   $("li:lt(2)")    //下标小于 2 的li
   $("li:not(#runoob)") //挑选除 id="runoob" 以外的所有li
   ```
   
   * 内容过滤选择器
     
     ```js
     $("div:contains('Runob')")    // 包含 Runob文本的元素
     $("td:empty")                 //不包含子元素或者文本的空元素
     $("div:has(selector)")        //含有选择器所匹配的元素
     $("td:parent")                //含有子元素或者文本的元素
     ```
   
   * 可见性过滤选择器
     
     ```js
     $("li:hidden")       //匹配所有不可见元素，或type为hidden的元素
     $("li:visible")      //匹配所有可见元素
     ```
   
   * 属性过滤选择器
     
     ```js
     $("div[id]")                 //所有含有 id 属性的 div 元素
     $("div[id='123']")           // id属性值为123的div 元素
     $("div[id!='123']")          // id属性值不等于123的div 元素
     $("div[id^='qq']")           // id属性值以qq开头的div 元素
     $("div[id$='zz']")           // id属性值以zz结尾的div 元素
     $("div[id*='bb']")           // id属性值包含bb的div 元素
     $("input[id][name$='man']") //多属性选过滤，同时满足两个属性的条件的元素
     ```
   
   * 状态过滤选择器
     
     ```js
     $("input:enabled")    // 匹配可用的 input
     $("input:disabled")   // 匹配不可用的 input
     $("input:checked")    // 匹配选中的 input
     $("option:selected")  // 匹配选中的 option
     ```

4. 表单选择器
   
   ```js
   $(":input")      //匹配所有 input, textarea, select 和 button 元素
   $(":text")       //所有的单行文本框，$(":text") 等价于$("[type=text]")，推荐使用$("input:text")效率更高，下同
   $(":password")   //所有密码框
   $(":radio")      //所有单选按钮
   $(":checkbox")   //所有复选框
   $(":submit")     //所有提交按钮
   $(":reset")      //所有重置按钮
   $(":button")     //所有button按钮
   $(":file")       //所有文件域
   ```

### 对过高元素进行折叠

* 效果
  
  * 折叠后
    
    ![](C:\Users\liuyo\AppData\Roaming\marktext\images\2022-08-25-14-43-06-image.png)
  
  * 折叠前
    
    ![](C:\Users\liuyo\AppData\Roaming\marktext\images\2022-08-25-14-43-32-image.png)
  
  * 代码实现
    
    ![](D:\Snipaste\temp\autosave\Snipaste_2022-08-25_14-56-03.png)

### 页面加载完毕事件

* 使用:
  
   $(document).ready(function(){
  
  });
  
  解释:$(document)把原生的 document这个  dom对   当页面加载完毕后执行里面的函数,这一种相象转换为 jQuery对象，转换完成后才能调用   对简单，用得最多。
  
  其中ready方法:ready(fn),表示的是页面结构被加载完毕后执行传入函数fn

* 使用:
  
  $(function(){
  
  });
  
  解释:当页面加载完毕后执行里面的函数,这一种相对简单,用的最多;
  
  上面两种方式与window.onload的区别:
  
  1、jQuery中的页面加载完毕事件，表示的是页面结构被加载完毕。
  
  2、window.onload  表示的是页面被加载完毕。如: onload必须等等页面中的图片、声音、图像等远程资源被加载完毕后才调用而 jQuery中只需要页面结构被加载完毕。

### 元素height获取

```js
$(选择器).height();    //返回元素height（px）
```
