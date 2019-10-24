

React-redux发布了7.1.0版本的hooks。

这意味着我们可以使用react的最新最佳实践。

**hooks让我们为相同的功能编写更少的代码。我们需要编写的代码越少，我们就可以越快地启动应用程序**

### 简单的Redux组件

```
import React, { Component } from "react";
import { connect } from "react-redux";
import { toggleSwitch } from "./UiReducer";

class Toggle extends Component {
  render() {
    const { ui, toggleSwitch } = this.props;
    return (
      <div>
        <div>{JSON.stringify(ui)}</div>
        <input
          type="checkbox"
          value={ui.toggle}
          onChange={toggleSwitch}
        />
      </div>
    );
  }
}

const mapStateToProps = ({ ui }) => ({
  ui
});

export default connect(
  mapStateToProps,
  { toggleSwitch }
)(Toggle);

```

##### hooks需要在函数组件中使用。不能在React类中使用hooks。

### First step - 将类组件重构为函数组件

```
import React from "react";
import { connect } from "react-redux";
import { toggleSwitch } from "./UiReducer";

const Toggle = ({ ui, toggleSwitch }) => (
  <div>
    <div>{JSON.stringify(ui)}</div>
    <input type="checkbox" value={ui.toggle} onChange={toggleSwitch} />
  </div>
);

const mapStateToProps = ({ ui }) => ({
  ui
});

export default connect(
  mapStateToProps,
  { toggleSwitch }
)(Toggle);

```

##### 可以将类语法替换为更简单的函数语法。从箭头函数参数peops中解构了ui和toggleSwitch属性。
##### Hooks通常用use关键字作为前缀，比如useState或者useSelector。

---

### Second Step - useSlector

从使用hooks读取状态开始。我们需要从react-redux包中导入useSelector。使用useSelector hook，
我们可以读取我们的状态

```
import React from "react";
import { connect, useSelector } from "react-redux";
import { toggleSwitch } from "./UiReducer";

const Toggle = ({connect,useSelector}=>{
    const ui = useSelector(state => state.ui);
    return(
        <div>
            <div>{JSON.stringify(ui}</div>
            <input type="checkbox" value={ui.toggle} onchange={toggleswitch} />
        </div>
    )
})

export default connect(
  null,
  { toggleSwitch }
)(Toggle);

```
#### 注意：我们删除了 ui 参数，并使用了 useSelector hook。useSelector 的第一个参数是存储的状态

### Third Step - useDispatch

useDispatch hook 让我们执行redux操作。我们从react-redux包导入useDispatch hook。

使用useDispatch相对简单，我们将hook实例保存在一个变量下。我们可以在任何时间监听器中调用dispatch函数.

```
import React from 'react';
import {useSelector, useDispatch} from 'react-redux';
import { TOGGLE } from './UiReducer';

const Toggle = ()=>{
    const ui = useSelector(state => state.ui);
    const dispatch = useDispatch();
    
    return (
        <div>
            <div>{ JSON.stringify(ui) } </div>
            <input
                type="checkbox"
                value={ui.toggle}
                onchange={()=>dispatch({ type : TOGGLE })}
            />
        </div>
    );
};

export default Toggle;

```

注意：我们在这里调用 dispatch 函数时使用类型常量 TOGGLE，我们在 Redux 常量中定义了这个类型并将其导入到组件中。

```
export const TOGGLE = "ui/toggle";
```

如果要将 payload 传递给 dispatcher，请像往常一样执行此操作。

```
dispatch({ type: TOGGLE, payload: 'My payload' })
```