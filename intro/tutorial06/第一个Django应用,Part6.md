# 开发第一个Django应用,Part6

   本教程上接[Part5](https://github.com/jhao104/django-chinese-docs-1.10/blob/master/intro/tutorial05/%E7%AC%AC%E4%B8%80%E4%B8%AADjango%E5%BA%94%E7%94%A8%2CPart5.md)。前面已经建立一个网页投票应用并且测试通过，现在主要讲述如何添加样式表和图片。
   
   除由服务器生成的HTML文件外，网页应用一般还需要提供其它必要的文件——比如图片、JavaScript脚本和CSS样式表。这样才能为用户呈现出一个完整的网站。 在Django中，这些文件统称为“静态文件”。
   
   如果是在小型项目中，这只是个小问题，因为你可以将它们放在网页服务器可以访问到的地方。 但是呢，在大一点的项目中——尤其是由多个应用组成的项目，处理每个应用提供的多个静态文件集合还是比较麻烦的。
   
   但是Django提供了`django.contrib.staticfiles`：它收集每个应用（和任何你指定的地方）的静态文件到一个单独的位置，使得这些文件很容易维护。
   
## 自定义应用外观

　　