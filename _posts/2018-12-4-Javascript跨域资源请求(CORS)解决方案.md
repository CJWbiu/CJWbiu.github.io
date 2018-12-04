---
layout:     post
title:      Javascript跨域解决方案
subtitle:   跨域资源请求及兼容
date:       2018-12-04
author:     CJWbiu
header-img: img/post-2018-12-4-cors.jpg
catalog: true
tags:
    - Javascript
    - CORS
    - 跨域
---
# 跨域
> 跨域：当协议、主域名、子域名、端口号中任意一个不相同时都不算同一个域，而在不同域之间请求数据即为跨域请求。      

## 为什么要限制跨域            

造成跨域的原因有两个：
1. DOM同源策略： 禁止对不同源页面DOM进行操作。
2. XmlHttpRequest同源策略：禁止使用XHR对象向不同源的服务器地址发起HTTP请求。
这两个策略主要是处于安全考虑，防止跨站攻击(CSRF)。如果没有同源限制，那么恶意网站可以随时向其他网站发起攻击请求，并附上用户的cookie等信息或者在iframe中嵌套网站通过操作iframe中的dom来获取用户的一些输入信息，会造成很大的危害。

解决方法有以下几种（如有错误欢迎指出）以请求图片url为例：  

### 通过XMLHttpRequest对象实现（IE10以下不支持）

XMLHttpRequest2.0已经实现了对CORS的原生支持，只需要在访问资源的时候使用绝对URL即可，需要在服务器端将头信息“Access-Control-Origin"设为”*“或当前域名的地址：       
```javascript
//html&javascript
<button id="btn">load</button>
  <div id="process"></div>
  <div id="img"></div>

  <script>
    var btn = document.getElementById('btn'),
        pro = document.getElementById('process'),
        pic = document.getElementById('img');

    btn.onclick = function() {
        var xhr = new XMLHttpRequest();
        xhr.onreadystatechange = function() {
        
            if(xhr.readyState == 4 && xhr.status == 200){
           
                var img = document.createElement('img');
                img.src = JSON.parse(xhr.responseText).src;
                pic.appendChild(img);
            }
        }
        xhr.open('GET','http://127.0.0.1:8085/AJAX/ajax/server.php',true);
        xhr.send(null);
    }        
</script>
```          
这个例子是在本地测试的，html文件所在的位置是localhost：8085下，虽然127.0.0.1也是指的localhost，但他们也不是同一个域，可以利用这两个地址来模拟跨域请求。        
```php
//php
<?php
header("Content-Type:text/plain");
header("Access-Control-Allow-Origin:http://localhost:8085");//设置头部，不设置的话请求会被拒绝
echo '{"src":"http://www.pinkbluecp.cn/face_alignment/img/picture.jpg"}';
?>
```      
post请求也差不多：        
```javascript
btn.onclick = function() {
            if(typeof XMLHttpRequest != 'undefined'){
                var xhr = new XMLHttpRequest();
            }else if(typeof ActiveXObject != 'undefined'){
                var xhr = new ActiveXObject("Microsoft.XMLHTTP");
            }

            xhr.onreadystatechange = function() {
                console.log(1)
                if(xhr.readyState == 4 && xhr.status == 200){
                    console.log(2)
                    pic.innerHTML = xhr.responseText;
                }
            }
            xhr.open('POST','http://127.0.0.1:8085/AJAX/ajax/server.php',true);
            xhr.setRequestHeader("Content-Type","application/x-www-form-urlencoded");    //需要设置下头部
            var str = 'active=login';
            xhr.send(str);
```
```php
<?php
header("Content-Type:text/plain");
header("Access-Control-Allow-Origin:http://localhost:8085");

if($_POST['active'] == 'login'){
    echo 'success';
}else {
    echo "error";
}
?>
```

发送数据需要设置一下头部的Content-Type字段。    
### IE（8-10）中实现CORS:     

IE8中引入了XDomainRequest对象，该对象和XHR相似，但它能实现安全可靠的跨域通信，不过在IE11中已经无法使用了。    
```javascript
var btn = document.getElementById('btn'),
            pro = document.getElementById('process'),
            pic = document.getElementById('img');

btn.onclick = function() {
    var xhr = new XDomainRequest();
    xhr.onload = function() {
        var img = document.createElement('img');
        console.log(xhr.responseText)
        // img.src = JSON.parse(xhr.responseText).src;
        img.src = xhr.responseText;
        console.log(img.src);
        pic.appendChild(img);
    };
    xhr.onerror = function() {  //XDR无法确定响应状态，需要添加onerror来检测错误
        alert("Error!");
    }
    xhr.open('GET','http://127.0.0.1:8085/AJAX/ajax/server.php'); //该对象的的请求都是异步，没有第三个参数
    xhr.send(null);
}
```

XDR与XHR不同之处在于：

1. cookie不会随请求发送，也不会随响应返回

2. 只能设置请求头中的Content-Type字段

3. 不能访问响应头部信息

4. 只支持GET和POST请求

两者的方法也略有不同，XDR的open方法只有异步一种状态，所以只要传两个参数method和url，在请求返回之后会触发load事件，但无法获取响应状态码，所以想判断请求是否成功只能添加error事件。它也可以和XHR一样添加超时。

关于XDR的post请求，虽然js高编上有介绍，但是在多次尝试后发现已经行不通了，高编上说XDR有专门的属性contentType来设置请求头信息，但在浏览其中会报错：`对象不支持此操作`。
无法进行设置contentType的操做，后来查阅了一下这篇[博客](http://blog.csdn.net/zhangchao19890805/article/details/52909156)，貌似XDR已经无法在设置请求头了，详情可以去看原博。
### 实现跨浏览器的CORS

js高编上有一段实现跨浏览器的CORS实现函数：     
```javascript
<button id="btn">load</button>
<div id="process"></div>
<div id="img"></div>

<script>
    var btn = document.getElementById('btn'),
        pic = document.getElementById('img');

    btn.onclick = function() {
        console.log(1);
        var xhr = createCORS('GET','http://127.0.0.1:8085/AJAX/ajax/server.php');
        if(xhr){
            console.log(2)
            xhr.onload = function(){
                console.log(3)
                var img = document.createElement('img');
                img.src = xhr.responseText;
                pic.appendChild(img);
            };
            xhr.onerror = function() {
                alert("Error");
            };
            xhr.send(null);
        }
    }

    function createCORS(method,url){  //参考js高编
        console.log('fun')
        var xhr = new XMLHttpRequest();
        if('withCredentials' in xhr){    //检测是否含有凭据属性
            xhr.open(method,url,true);
        }else if(typeof XDomainRequest != 'undefined'){    //兼容ie
            xhr = new XDomainRequest();
            xhr.open(method,url);
        }else {
            xhr = null;
        }
        return xhr;
    }
</script>
```    
除了IE10-外所有浏览器都有withCredentials属性，所以可以根据这个属性来判断是否支持XMLHttpRequest2.0.     
### 图片Ping    

图片Ping就是利用图片的src可以使用跨域资源的特性来实现，但是只能实现简单的单向请求，在img的src中传入一个其他域的url并附带一些参数。   
### JSONP     

JSONP和图片Ping很相似，也是利用script标签中的链接不受同源限制可以向不同域传递信息，但它传递的是一个已经存在的回调函数，服务器接收到后，将响应数据传入该回调函数并执行该函数从而达到获取数据的目的。

先来看下<script>标签中传入src得到的是什么：   
```javascript
<script src="test.txt"></script>
```     
在src中传入一个文本文件             
![博客](https://images2017.cnblogs.com/blog/1141140/201710/1141140-20171015195208746-1333727507.png "blog")     

浏览器中报语法错误，hello world这个变量未定义，服务器响应的数据就时test.txt的内容，而script把这段纯文本当作js代码来解析，当在test.txt中将这段文字加上单引号后：  

![博客](https://images2017.cnblogs.com/blog/1141140/201710/1141140-20171015195448340-184344941.png "blog")

浏览器不报错了，因为把它当作字符串来解析了。现在把链接换成一个php文件：     
```php
<?php
echo "Hello World!";
?>
```      
![博客](https://images2017.cnblogs.com/blog/1141140/201710/1141140-20171015195727824-176062308.png "blog")    
结果一样，浏览器同样也会报错，加上引号后同样也不报错，如果这时服务器端返回的是一个函数名，而该页面正好又有一个同名函数会怎样：      
```php
<?php
echo "callback('hello world')";
?>
```       
html:       
```html
<body>
    <script>
        function callback(res){
            alert(res);
        }
    </script>
    <script src="jsonp.php"></script>
</body>
```      
结果：               

![博客](https://images2017.cnblogs.com/blog/1141140/201710/1141140-20171015200613355-1863472461.png "blog")       
callback函数被执行了，因为后台的响应数据被script标签当作js代码执行了，所以这就能理解jsonp的原理了，利用这个回调函数可以获得后台传来的数据：        
```html
<body>
    <script>
        function jsonp(res){
            for(var i in res){
                console.log("key:"+i+";value:"+res[i]);
            }
        }
    </script>
    <script src="jsonp.php?callback=jsonp"></script>  //可动态创建
</body>
```              
```php
<?php
$str = '{"name":"Lee","age":20}';
$callback = $_GET['callback'];
echo $callback."($str)";
?>
```         
上面一个简单的例子展示了jsonp如何用回调函数获取服务器响应的json数据：       
![博客](https://images2017.cnblogs.com/blog/1141140/201710/1141140-20171015201629934-564613446.png "blog")      
 JSONP的有点在于可以访问到服务器的响应文本，不过缺点就是要从其他域加载代码执行，必须要保证其他域的安全性，不然可能响应信息中会附带恶意脚本，还有一点就是无法确定请求是否失败，即使失败也不会有提示。        

 ## iframe跨域     

 iframe跨域与jsonp相似，也利用了src不受同源限制的特性。当A域下的x.html页面要访问B域下的y.html中的信息可以通过A域下的z.html页面作为代理来获取信息：

 x.html:         

```html
<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>无标题文档</title>
</head>

<body>
<iframe src="http://127.0.0.1:8085/AJAX/ajax/proxya.html" style="display: none"></iframe>
<p id="getText"></p>
<script>
    function callback(text){
        text = JSON.parse(decodeURI(text));
        document.getElementById("getText").innerHTML= '姓名：' + text.name + '; 年龄：' + text.age;    
    }
</script>
</body>
</html>
```       
y.html:           
```html
<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>无标题文档</title>
</head>

<body>
<iframe id="myfarme" src="###"></iframe>
<script>
    window.onload = function(){
        var text = '{"name":"cheng","age":22}';　　//存储一个json格式的数据
        document.getElementById('myfarme').src="http://localhost:8085/AJAX/ajax/proxyb.html?content="+encodeURI(text); //将数据传个代理页面处理，此时src中的地址不受同源限制，与jsonp相似
    }
</script>
</body>
</html>
```            
z.html:          
```html
<!doctype html>
<html>
<head>
<meta charset="utf-8">
<title>无标题文档</title>
<script>
window.onload = function(){
    var text = window.location.href.split('=')[1];　　//通过分解url获取y.html传来的参数
    top.callback(text);　　//调用x.html的callback函数
}
</script>
</head>

<body>
</body>
</html>
```        