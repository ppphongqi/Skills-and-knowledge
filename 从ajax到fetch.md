## 从ajax到fetch、axios

前端是个发展迅速的领域，前端请求自然也发展迅速，从原生的XHR到jquery ajax，再到现在的axios和fetch。

### jquery ajax

```

$.ajax({
    type : 'post',
    url  :  url , 
    data :  data,
    dataType : dataType,
    success : function(){},
    error : function(){}
})

```

ajax是对原生XHR的封装，还支持JSONP，非常方便；但是随着react，vue等前端框架的兴起，juery早已不复当年之勇。很多情况下我们只需要使用ajax，但是却需要引入整个jquery，这非常的不合理，于是便有了fetch的解决方案。

### fetch

fetch号称是ajax的代替品，它的API是基于Promise设计的，旧版本的浏览器不支持Promise，需要
使用polyfill es6-promise

举个栗子：
```

//原生XHR
var xhr = new XMLHttpRequest();
xhr.open('GET',url);
xhr.onreadystatechange = function(){
    if(xhr.readyState === 4 && xhr.status === 200) {
        console.log(xhr.responseText) //从服务器获取数据
    }
}
xhr.send()

//fetch
fetch(url)
    .then(response => {
    if(response.ok){
        return response.json();
    }
    })
    .then(data => console.log(data))
    .catch(err => console.log(err)
    
```

then链就像是之前熟悉的callback。
在MDN上，讲到它跟juery ajax的区别，这也是fetch很奇怪的地方：

>当接收到一个代表错误的 HTTP 状态码时，从 fetch()返回的 Promise 不会被标记为 reject， 即使该 HTTP 响应的状态码是 404 或 500。相反，它会将 Promise 状态标记为 resolve （但是会将 resolve 的返回值的 ok 属性设置为 false ），  仅当网络故障时或请求被阻止时，才会标记为 reject。
默认情况下, fetch 不会从服务端发送或接收任何 cookies, 如果站点依赖于用户 session，则会导致未经认证的请求（要发送 cookies，必须设置 credentials 选项）.

再搭配上async/await将会让我们的异步代码更加优雅：

```
async function test(){
    let response = await fetch(url);
    let data = await response.json();
    console.log(data)
}
```

然而async/await是ES7的API，目前还在试验阶段，还需要我们使用babel进行转译成ES5代码。


fetch是比较底层的API，很多情况下都需要我们再次封装。 比如：
```
//jquery ajax
$.post(url , {name : 'test'})

//fetch
fetch(url , {
    method:'POST',
    body: Object.keys({name :'test' }).map((key)=>{
    return encodeURLComponent(key) + '=' + encodeURLComponent(params[key]);
    }).join('&')
});

```

由于fetch是比较底层的API，所以需要我们手动将参数拼接成'name = test'的格式