---
layout: post
title: "Integrate Octopress with Flickr"
date: 2014-09-22 15:59:15 +0800
comments: true
categories: [Site]
tags: [Site, Web, Photography]  
keywords: OctoPress, Flickr, plugin, Photography  
description: OctoPress-Flickr是一款可以将Flickr中的某个图片插入到你的octopress博文中的插件。

---

Octopress Blog有一个很酷的[插件][github]，可以将Flickr中的某个图片插入到你的博文中。像这样：    
[github]: https://github.com/neilk/octopress-flickr  
\{\% flickr_image 15313566521 o \%\}


还可以把Flickr某个相册中的所有图片都插入进来，像[这样][set]。  
[set]: http://alphawang.com/photography/  
<!--more-->  

使用起来非常简单，只需要在博文中插入类似如下内容：  

```
// 注意：要删除\

// 插入图片  
\{\% flickr_image 15313566521 o \%\}

// 插入相册  
\{\% flickr_set 72157647828539946 q \%\}   
```  

当然，在这之前要安装这个插件。

## 安装octopress-flickr

基本上按照[github][github]上的步骤就可以顺利安装。  
[github]: https://github.com/neilk/octopress-flickr   

主要步骤如下。  

**1、获取Flickr api key**

首先去[Flickr][f]获取一个api key以及对应的secret。然后将它们写入`_config.yml`：  
[f]: http://www.flickr.com/services/developer/api/ 

```
flickr:  
  api_key: <api_key>  
  shared_secret: <shared_secret>  
```  
**2、获取插件**  
git clone [插件][github]到本地，拷贝其中的`.rb`和`.scss`文件到Octopress得相应目录中。   
[github]: https://github.com/neilk/octopress-flickr   
      
  
**3、安装插件**  
在Gemfile文件中新增如下几行：  

``` 
  gem 'flickraw'  
  gem 'builder', '> 2.0.0'  
  gem 'persistent_memoize'  
```  
运行命令：
```
bundle install
```  

并确保`sass/screen.scss`以如下行结尾：  
```
@import "plugins/**/*";
```
  
**4、插入图片**  

至此，你就可以在blog里插入Flickr图片了，只要找到图片id或者相册id，然后像本文开头那样写到文章即可。  

这个插件提供了2个新的tag，分别用来插入单个图片和相册，语法如下：

```  

\{\% flickr_image id [preview-size [alignment [caption]] \%\}  
\{\% flickr_set id [preview-size [desc|nodesc]] \%\}  

```  


`preview-size`用一个字符来表示图片大小： 

- o : "Original", no maximum dimension  
- b : "Large", 1024px
- z : "Medium 640", 640px
- n : "Small 320", 320px
- m : "Small", 240px,
- t : "Thumbnail", 100px
- q : "Large Square", 150px
- s : "Square", 75px



`alignment`可以设为`left`, `right`, 或者`center`。 
 
`caption` 是图片标题。


 
>不过仅仅这样还有个小问题：点击图片，会跳转到Flickr页面去，我们可以用Fancybox达到在当前页面弹出图片。

##安装Fancybox

Fancybox的效果是这样的：  
![ fancybox ](/images/post/2014/09/fancybox.png)  

去[Fancybox官网][fancybox]下载并解压，将`source`目录中的文件拷贝到`octopress/source/fancybox`目录中。  
[fancybox]: http://fancyapps.com/fancybox/  
  
  
接下来将octopress-flickr插件中的`source/_includes/custom/fancybox_head.html`文件拷贝到Octopress相应目录中。  

编辑`source/_includes/head.html`文件，在末尾插入：  
```  
\{\% include custom/fancybox_head.html \%\}   
```    

然后修改`sass/base/_theme.scss`：
```  
body {
-  > div {
+  > #main {
     border-bottom: 1px solid $page-border-bottom;
-    > div {
+    > #content {
       border-right: 1px solid $sidebar-border;
     }
   }
```  

大功告成。  

##设置导航

你还可以在导航栏设置一个页面来专门显示Flickr中的照片流。执行：
```  
rake new_page['photography'] 
```  

按照本文开头的语法，添加一个相册到相应的index.makrdown中。  
```
// 注意：要删除\
\{\% flickr_set 72157647828539946 q \%\}    
```


在`source/_includes/custom/navigation.html`中添加导航:  
```
<li><a href="/photography">Photography</a></li>
```

<!--Google Adsense-->
<p class="meta" style="text-align:center">
	<!-- 789*90 -->
	<script async src="//pagead2.googlesyndication.com/pagead/js/adsbygoogle.js"></script>
	<ins class="adsbygoogle"
	     style="display:inline-block;width:789px;height:90px"
	     data-ad-client="ca-pub-6393503301700908"
	     data-ad-slot="7806666870"></ins>
	<script>
	(adsbygoogle = window.adsbygoogle || []).push({});
	</script>
</p>










