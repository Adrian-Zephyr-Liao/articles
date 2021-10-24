# React之派生state

> react在16.3版本以前，想要在props发生变化的时候去更新组件内部的state的唯一方式是使用componentWillReceiveProps，后面增加了一个getDerivedStateFromProps来达到这种目的。

使用派生state会存在以下两种情况：
+ 将父组件传递的props复制一份放到子组件的state中进行管理
+ 在componentWillReceiveProps中接收props的变化，以更新子组件state

## 复制props到state
大多数人会认为，componentWillReceiveProps和getDerivedStateFromProps只会在props发生改变的时候被调用，实则不然。当父组件被重新渲染的时候，这两个生命周期就会被重新调用，这就造成了子组件中状态的丢失。
```jsx
class DefaultInput extends Component {
    constructor(props) {
        super(props);
        this.state = {
            value: props.value,
        };
    }

    handleChange = (event) => {
        this.setState({ value: event.target.value });
    };

    componentWillReceiveProps(nextProps) {
        this.setState({ value: nextProps.value });
    }

    render() {
        return <input value={this.state.value} onChange={this.handleChange} />;
    }
}

class State extends Component {
    constructor(props) {
        super(props);
        this.state = {
            count: 0,
        };
    }
    componentDidMount() {
        this.interval = setInterval(
            () =>
                this.setState((prevState) => ({
                    count: prevState.count + 1,
                })),
            5000,
        );
    }
    componentWillUnmount() {
        clearInterval(this.interval);
    }

    render() {
        return <DefaultInput value="我的名字叫小明" />;
    }
}
```
### 对问题的思考
1. 那如果你单纯的只想要一个默认值呢？不在子组件中做props更新对state产生变化的动作，这样的设计是可以的。
2. 还有一种改进的做法，不去在子组件中进行state的更新，我们只做props的监听，在props改变的时候在子组件的生命周期中更新state。这种情况个人认为是没有必要的，如果通过父组件的props改变去更新state，不在子组件中做，那就根本不需要维护这个state了，直接将组件维护成为一个受控组件（这就是最佳实践的一种）。这种做法会导致组件的复用问题，比如说我有个tabs来切换两个表单，表单的输入框的默认值都是一样的，通过props传递了进去，但是props并未更改，这就会造成组件的复用现象，典型的密码框混用，这是不想被用户看到的。

## 最佳实践
上述问题的关键在于：<strong>任何数据，都要保证只有一个数据来源，而且避免直接复制它</strong>

### 完全受控组件
这个就是将state从子组件中移除，将所有状态的改变通过父组件去更改要传递的props来实现这个功能，这也符合单向数据流的思想。
```jsx
function DefaultInput(props) {
  return <input onChange={props.onChange} value={props.value} />;
}
```
### 完全非受控组件
子组件在接受到父组件传过来的值后，在内部维护一个state，这个state的改变与父组件毫不相关，这样也可以保证数据来源的纯净。但是也会衍生一个问题，就是react会对已存在的组件进行复用，以节省开销。
```jsx
class DefaultInput extends Component {
  state = { value: this.props.value };

  handleChange = event => {
    this.setState({ value: event.target.value });
  };

  render() {
    return <input onChange={this.handleChange} value={this.state.value} />;
  }
}

class State extends Component {
    constructor(props) {
        super(props);
        this.state = {
            show: true,
        };
    }

    handleClick = () => {
        console.log(this.state.show);
        this.setState({
            show: !this.state.show,
        });
    };

    render() {
        return (
            <div>
                {this.state.show ? (
                    <DefaultInput value="我的名字叫小明" key="1" />
                ) : (
                    <DefaultInput value="我的名字叫小芳" key="2" />
                )}
                <button onClick={this.handleClick}>切换</button>
            </div>
        );
    }
}
```
上述代码实现了一个通过按钮来切换输入组件的作用，当用户点击按钮的时候，就会去切换默认值是小明还是小芳的输入框，但是如果不给key值的话，你无论怎么切换还是小明，这就涉及到react的组件更新策略了。我们要手动告知react这是两个不一样的组件，你要在切换的时候销毁原来的组件，并重新创建一个，这并不会带来很大的开销，相比较在组件内部更新逻辑十分复杂的时候，这样做的速度会更快，因为省去了组件的diff成本。
#### 添加key值失效问题
但有的时候因为创建组件的开销太大，给组件添加key值并没有什么用处，在改变某些字段的时候，去观察某些特殊属性是否发生变化，例如，我在使用这个输入框组件的时候，我改变得值仅仅只是value，但是我可以给每个组件赋予独一无二的id，如果组件的id发生改变的时候，我就去重新设置组件内部的state值（这里也就是value）。
```jsx
componentWillReceiveProps = (nextProps) => {
    if (nextProps.id !== this.state.id) {
        this.setState({
            value: nextProps.value,
            id: nextProps.id,
        });
    }
};
```
#### 使用ref来重置组件
这个方法相对于其他方法来说，这是非理想的，这会造成组件二次渲染，在子组件内部定义一个方法来重置组件内部的state值，然后切换子组件的时候通过ref来获取这个方法进行重置。