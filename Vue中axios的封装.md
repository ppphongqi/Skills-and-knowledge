###  axios是Vue官方推荐的一个HTTP库，用axios官方简介来介绍就是：
>Axios 是一个基于 promise 的 HTTP 库，可以用在浏览器和 node.js 中

#### 开始

网上关于 axios 封装的代码不少，但是大部分都是在入口文件（main.js）中进行 axios 全局对象属性定义的形式进行配置，类似于如下代码：

```
axios.defaults.timeout = 10000
```


该方案有两个不足，首先，axios封装代码耦合进入入口文件，不方便后期维护；其次，使用axios全局对象属性定义的方式进行配置，代码过于零散。

针对问题一，我们应该使用 Vue 源码结构中的一大核心思想——将功能拆分为文件，方便后期的维护。单独创建一个 ==http.js 或者 http.ts== 文件，在文件中引入 axios 并对其进行封装配置，最后将其导出并挂载到 Vue 的原型上即可。此时，每次修改 axios 配置，只需要修改对应的文件即可，不会影响到不相关的功能。

针对问题二，采用 axios 官方推荐的，通过配置项创建 axios 实例的方式进行配置封装。代码如下:

```
// http.js
import axios from 'axios'
// 创建 axios 实例
const service = axios.create({
  // 配置项
})
```

### 根据环境设置baseURL

baseURL属性是请求地址前缀，将自动加在url前面，除非url是个绝对地址。正常情况下，在开发环境下和生产环境下有着不同的baseURL，所以，我们需要根据不同的环境切换不同的baseURL。

在开发模式下，由于有着 devServer 的存在，需要根据固定的 url 前缀进行请求地址重写，所以，在开发环境下，将 baseURL 设为某个固定的值，比如：/apis。

在生产模式下，根据 Java 模块的请求前缀的不同，可以设置不同的 baseURL。

```
// 根据 process.env.NODE_ENV 区分状态，切换不同的 baseURL
const service = axios.create({
	baseURL: process.env.NODE_ENV === 'production' ? `/java` : '/apis',
})

```

### 统一设置请求头

什么是封装？在我看来，封装是通过更少的调用代码覆盖更多的调用场景。

由于，大部分情况下，请求头都是固定的，只有少部分情况下，会需要一些特殊的请求头，所以，在这里，我采用的方案是，将普适性的请求头作为基础配置。当需要特殊请求头时，将特殊请求头作为参数传入，覆盖基础配置。


```
const service = axios.create({
    ...
	headers: {
        get: {
          'Content-Type': 'application/x-www-form-urlencoded;charset=utf-8'
          // 在开发中，一般还需要单点登录或者其他功能的通用请求头，可以一并配置进来
        },
        post: {
          'Content-Type': 'application/json;charset=utf-8'
          // 在开发中，一般还需要单点登录或者其他功能的通用请求头，可以一并配置进来
        }
  },
})

```

### 跨域、超时、响应码处理

axios 中，提供是否允许跨域的属性——withCredentials，以及配置超时时间的属性——timeout，通过这两个属性，可以轻松处理跨域和超时的问题。

axios 提供了 validateStatus 属性，用于定义对于给定的HTTP 响应状态码是 resolve 或 reject  promise。所以，正常设置的情况下，我们会将状态码为 2 系列或者 304 的请求设为 resolve 状态，其余为 reject 状态。结果就是，我们可以在业务代码里，使用 catch 统一捕获响应错误的请求，从而进行统一处理。

但是，由于我在代码里面使用了 async-await，而众所周知，async-await 捕获 catch 的方式极为麻烦，所以，在此处，我选择将所有响应都设为 resolve 状态，统一在 then 处理。

```
const service = axios.create({
	// 跨域请求时是否需要使用凭证
	withCredentials: true,
    // 请求 30s 超时
	timeout: 30000,
	validateStatus: function () {
		// 使用async-await，处理reject情况较为繁琐，所以全部返回resolve，在业务代码中处理异常
		return true
	},
})

```

### 请求、响应处理

在不使用axios的情况下，每次请求或者接收响应，都需要将请求或者响应序列化。

而在axios中，transformRequest允许在向服务器发送请求前，修改全球数据；transformResponse在传递给then/catch前，允许修改响应数据。

通过这两个钩子，可以省去大量重复的序列化代码。

代码如下;

```
const service = axios.create({
    // 在向服务器发送请求前，序列化请求数据
    transformRequest: [function (data) {
        data = JSON.stringify(data)
        return data
    }],
    // 在传递给 then/catch 前，修改响应数据
    transformResponse: [function (data) {
        if (typeof data === 'string' && data.startsWith('{')) {
            data = JSON.parse(data)
        }
        return data
    }]
})

```

### 拦截器

拦截器，分为请求拦截器以及响应拦截器，分别在请求或响应被then或catch处理前拦截它们。

之前提到过，由于async-await中catch难以处理的问题，所以将出错的情况也作为resolve状态进行处理。但这也带来一个问题，请求或响应出错的情况下，结果没有数据协议中定义的msg字段。所以，我们需要在出错的时候，手动生成一个符合返回格式的返回数据。

由于，在业务中，没有需要在请求拦截器中做额外处理的需求，所以，请求拦截器的resolve状态，只需直接返回就可以了。

请求拦截器代码如下：

```
// 请求拦截器
service.interceptors.request.use((config) => {
	return config
}, (error) => {
	// 错误抛到业务代码
    error.data = {}
    error.data.msg = '服务器异常，请联系管理员！'
    return Promise.resolve(error)
})

```

再来聊聊响应拦截器，还是之前的那个问题，除了请求或响应错误，还有一种情况也会导致返回的消息体不符合协议规范，那就是状态码不为 2 系列或 304 时。此时，我们还是需要做一样的处理——手动生成一个符合返回格式的返回数据。但是，有一点不一样，我们还需要根据不同的状态码生成不同的提示信息，以方便处理上线后的问题。


响应拦截器代码如下：

```
// 根据不同的状态码，生成不同的提示信息
const showStatus = (status) => {
    let message = ''
    // 这一坨代码可以使用策略模式进行优化
    switch (status) {
        case 400:
            message = '请求错误(400)'
            break
        case 401:
            message = '未授权，请重新登录(401)'
            break
        case 403:
            message = '拒绝访问(403)'
            break
        case 404:
            message = '请求出错(404)'
            break
        case 408:
            message = '请求超时(408)'
            break
        case 500:
            message = '服务器错误(500)'
            break
        case 501:
            message = '服务未实现(501)'
            break
        case 502:
            message = '网络错误(502)'
            break
        case 503:
            message = '服务不可用(503)'
            break
        case 504:
            message = '网络超时(504)'
            break
        case 505:
            message = 'HTTP版本不受支持(505)'
            break
        default:
            message = `连接出错(${status})!`
    }
    return `${message}，请检查网络或联系管理员！`
}

// 响应拦截器
service.interceptors.response.use((response) => {
    const status = response.status
    let msg = ''
    if (status < 200 || status >= 300) {
        // 处理http错误，抛到业务代码
        msg = showStatus(status)
        if (typeof response.data === 'string') {
            response.data = { msg }
        } else {
            response.data.msg = msg
        }
    }
    return response
}, (error) => {
    // 错误抛到业务代码
    error.data = {}
    error.data.msg = '请求超时或服务器异常，请检查网络或联系管理员！'
    return Promise.resolve(error)
})

```








