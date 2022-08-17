---
layout: post
title: "How to Pass POJO Array to Spring MVC Controller"
date: 2014-10-26 15:59:15 +0800
comments: true
categories: [Java]
tags: [Spring MVC, Java, ajax]  
keywords: POJO Array, Spring MVC   
description: 如何在前端用ajax将基本数组传递给SpringMVC Controller? 如何在前端用ajax将Map对象传递给SpringMVC Controller? 如何在前端用ajax将对象数组传递给SpringMVC Controller?


---

## 问题
最近遇到个需求，需要将一个对象数组从前端用ajax传递给后端的Spring MVC Controller，例如传递如下数据：  
```
{
  products: [
    {
      category: 100,
      name: 'iPad',
      attributes: {
        volume: '32G',
        color: 'white'
      }
    },
    {
      category: 200,
      name: 'xx t-shirt',
      attributes: {
        size: 'M',
        color: 'red'
      }
    }
    
  ],
  locale: 'zh_CN'
}  
```  
<!--more-->
其中locale是一个字符串，products是一个对象数组。该对象的attributes属性名是不固定的，例如有的产品的attributes是volume+color，而有的则是size+color；所以这个attributes属性我们用Map来表示。  

## 传递基本数组
首先看看如何传递基本类型数组，例如Long[]。  
在后端接收该参数时很简单，只需要使用Long[]类型的@RequestParam即可，例如：  
```
@RequestMapping(value = "/test/findproduct", method = RequestMethod.POST)
@ResponseBody
public List<Product> findProduct(
  @RequestParam(value="objectValues", required=false) String[] objectValues) {
 //注意value="objectValues"

}

```

前端代码如下：  

```
                $.ajax({
                    type: "POST",
                    contentType: 'application/x-www-form-urlencoded; charset=utf-8;',
                    url: '/kcci/test/findproduct',
                    data: {
                        objectValues: [1000, 1001, 1002]
                    },
                    traditional: true,
                    success: function(items){ },
                    error: function(ejqXHR, textStatus, errorThrown) { }
                });  
```  

### 关于traditional: true  
注意ajax中有个参数`traditional: true`，**网上很多文章说只有这样才能让Java后台识别数组参数，但貌似并非如此**。
traditional默认值为false，jquery会深度序列化参数对象，以适应如PHP和Ruby on Rails框架。序列化后的格式为：  

```
objectValues[]:1000  
objectValues[]:1001  
objectValues[]:1002  
```

但servelt api无法处理，我们可以通过设置traditional: true阻止深度序列化，这样序列化结果如下：  
```
objectValues:1000
objectValues:1001
objectValues:1002
```

区别仅仅是参数名不一样，那么是否将后台的`@RequestParam(value="objectValues"）`改成`@RequestParam(value="objectValues[]")`，就能接收traditional:false的参数了呢？测试下来貌似确实是可以的！


## 传递Map

前文说到要传递的对象中有个Map属性，那么如果只仅仅传递Map参数应该怎么做呢？如果直接使用Map类型的RequestParam，例如`@RequestParam(required = false) Map<String, String> params`，那么这个params参数实际是所有传入参数的集合。如果我们发送的参数是：  
```
{
 objectValues: [1000, 1001, 1002],
 attributes: {
  COLOR: 'red',
  SIZE: 'M'
 }
}
```  
那么接收到的params为：  
![params icon](/images/post/2014/10/mapParams.png) 

可见这个params里不仅有attributes这个Map，还有其他参数。  

如果要让params仅仅包含我们关心的attributes map改怎么处理呢？我们需要自己新建一个Wrapper类：  
```
@Data
public class AttributeMapWrapper {
    Map<String, String> attributes;
}
```  
然后在Controller方法参数里这样写：  
```
@ModelAttribute(value = "attributes") AttributeMapWrapper attrWrapper
```


## 传递对象数组

前面讲了如何传递基本数组和Map，那么现在问题来了，如何才能传递对象数组呢？  
网络上能找到一些方法，例如下面这两个：   

- http://stackoverflow.com/questions/14459171/spring-mvc-and-jquery-post-request-with-array-of-pojo   
- http://stackoverflow.com/questions/17987234/passing-json-array-from-javascript-to-spring-mvc-controller  


基本思路是通过`@RequestBody`来获取Map，但是我在本地没有测试成功，可能原因是这里输入的并不是单一的数组，还有一个和数组并列的locale变量。  

### HttpServletRequest + GSon
可以利用HttpServletRequest获取传回的string，然后利用Gson转换为对象。  
``` 
    public List<Item> matchItems(
  @RequestParam(required = false, value = "Locale") String locale
  HttpServletRequest request) {
    Gson gson = new Gson();
    List<ParameterWrapper> param = gson.fromJson(request.getParameter("paramList"), 
    new TypeToken<List<ParameterWrapper>>(){}.getType());
    ...

```  

这里为参数对象写了一个Wrapper类：
```
@Data
public class ParameterWrapper {
    Long transactionId;
    Long categoryId;
    String barcode;
    String productName;
    String itemName;
//    @SerializedName("attributes")
    Map<String, String> attributes;
}
```
这样就能成功接收本文开头那种对象数组参数了。  

### @RequestBody  
HttpServletRequest方法显得太过于底层，过于面向细节。上文说到因为和数组并列有个locale变量，导致无法使用@RequestBody，那是否可以通过修改API输入参数的设计来解决呢，把locale变量放到`ParameterWrapper`中是否就可以使用@RequestBody了呢？  
当然可以，代码如下：  

```
    public List<Item> matchItems(@RequestBody ParameterWrapper[] params) {

    ...

```

```
@Data
public class ParameterWrapper {
    Long transactionId;
    Long categoryId;
    String barcode;
    String productName;
    String itemName;
    String locale;
//    @SerializedName("attributes")
    Map<String, String> attributes;
}
```

如果不能移动locale的位置该怎么办呢？可以再建一个类，其中包含两个变量：  
- `ParameterWrapper[]`   
- `String locale`   

## 总结

本文讨论了如何往Spring MVC Controller中传入基本数组、传入Map、传入对象数组。当遇到技术瓶颈时，不妨换个思路，有时修改一下设计就能豁然开朗。






