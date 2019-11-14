
### 看看react如何动态添加class

如果不使用第三方模块实现动态添加class是酱紫的：
```
const _classNames = whindow.people ? 'active main' : 'main';

return (
    <div className = {_classNames}></div>
)

```

### 使用classnames动态添加class

npm安装
```
 npm install classnames -S
```
使用

```
const sidebarOpenClass = classNames({
  main: true,
  active: window.people
});
return (
  <div className=${_classNames}></div>
)

```

分析classnames内部实现

```
(function () {
  'use strict';
  // =>  Object.hasOwnProperty   用于判断某个成员是否在对象内
  var hasOwn = {}.hasOwnProperty;
  function classNames () {
     // 存储 className 值
    var classes = [];

    // 循环实参， arguments就是实际调用函数传入的参数，类似数组
    for (var i = 0; i < arguments.length; i++) {
      // 获取实参value
      var arg = arguments[i];

      // 跳过false条件 => false, null, undefined, NaN, 空, ...
      if (!arg) continue;

      // 判断传入参数的类型
      var argType = typeof arg;

      // 如果参数的类型是 string 或者 number
      if (argType === 'string' || argType === 'number') {
        // 直接追加到classes数组后面
        classes.push(arg);

      // 如果参数是数组并且长度大于0
      } else if (Array.isArray(arg) && arg.length) {
        // 调用自身函数，利用apply可以将数组转成字符串
        var inner = classNames.apply(null, arg);

        // 现在是一个字符串，隐士判断布尔值
        if (inner) {
          // 追加到数组后面
          classes.push(inner);
        }
      // 如果传入的参数是对象
      } else if (argType === 'object') {
        // 对object进行遍历
        for (var key in arg) {
          // 判断key是否存在arg对象内并且key的值隐士转换为true
          if (hasOwn.call(arg, key) && arg[key]) {
            // 将值追加到classes数组后面
            classes.push(key);
          }
        }
      }
    }
    // 将数组连接成字符串以空格拼接  => a b c
    return classes.join(' ');
  }

  // 如果是node.js环境运行
  if (typeof module !== 'undefined' && module.exports) {
    classNames.default = classNames;
    module.exports = classNames;

  // 如果用的requirejs模块管理 AMD
  } else if (typeof define === 'function' && typeof define.amd === 'object' && define.amd) {
    define('classnames', [], function () {
    return classNames;
  });

  // 否则运行于浏览器环境
  } else {
    window.classNames = classNames;
  }
}());

```

### classnames使用姿势

classnames使用方式灵活，演示几个demo

#### demox1

```
import classNames from 'classnames';
const _className = classNames('foo'); //=> 'foo'

```
#### demox2

```
const _className = classNames('foo',{
    bar:true,
    active:false,
},['arr-1' , 'arr-2']);
// => foo bar arr-1 arr-2
```

classNames对实参是没有限制的，比较常用的就是在react项目当中