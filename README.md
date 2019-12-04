# react-router-source-code
react-router 和 history 和其中的依赖都是都是 Michael Jackson 老哥一个人写的太猛了

[mjackson](https://www.npmjs.com/~mjackson 'mjackson')
## history
react-router 所有对url的处理都是基于 [history](https://github.com/ReactTraining/history 'history') 的，所以如果不知道用法，看 react-router 的源码就会看得很难受，所以我们先来研究 history 源码，详见 [history-source-code](https://github.com/ChunchunIsMe/history-source-code 'history-source-code')

## BrowserRouter
如果对 history 了解之后我们就通过 BrowserRouter 这条路来对 react-router 进行探查

### BrowserRouter.js
package/react-router-dom/modules/BrowserRouter.js

我们先来看导入的模块
1. react -> React
2. react-router -> Router
3. history -> createBrowserHistroy 
4. prop-types -> PropTypes
5. tiny-warning -> warning

比较常见所以没什么好说的

组件实现：
这里就是使用 createBrowserHistory 创建了一个 history 对象然后将 histroy 和 this.props.children 传入了 Router 组件

这里的 createBrowserHistory 需要传入一个
```
{
  forceRefresh = false,
  getUserConfirmation = getConfirmation,
  keyLength = 6,
  basename = '',
}
```
然后会返回一个
```
{
  length: globalHistory.length,
  action: 'POP',
  location: initialLocation,
  createHref,
  push,
  replace,
  go,
  goBack,
  goForward,
  block,
  listen,
  setState
}
```
具有这些属性和方法的对象

这里如果是开发环境则会判断 propType 并且会生成一个 componentDidMount 周期函数会判断 props 是否有 history 如果有则会告诉你传入没有用

### RouterContext.js
因为 Router 和 route 都是通过 Context 来进行通信的并且它创建了一个 Context 来让大家导入，所以我们先来看这个 package/react-router/modules/RouterContext.js

先看导入的模块
1. create-react-context -> createContext

这个是 React v16 之前的 context 的用法，这里应该是为了做兼容

他这里就是创建了一个 context 然后导入了，他给 context 的 Provider/Consumer 都设置了 displayName 属性，这个就是用来在 devtool 上用的属性
### Router.js
package/react-router/modules/Router.js

先看导入的模块：
1. react -> React
2. prop-types -> PropTypes
3. tiny-warning -> warning
4. RouterContext.js -> RouterContext

这些也都没什么好说的比较常见

组件实现：

1. 定义了一个静态方法 computeRootMatch 这个静态方法只是返回了一个对象
2. 在 constructor 中将 props.histroy.location 赋值给 this.state.location
3. 定义 this._isMounted/this._pendingLocation 来做当组件还没有渲染的时候跳转
4. 如果 props.staticContext 不存在
5. 将在 history 的监听数组中加入一个函数，每次 url 操作都会调用这个函数，当调用这个函数时，如果组件还没渲染完成则会将新的 location 赋值给 this._pendingLocation，如果组件渲染完成了，则直接调用 this.setState 更新 this.state.location
6. 这里还将监听函数去除的方法赋值给了 this.unlisten
7. componentDidMount：当组件渲染完成之后 标记状态将 this._isMounted 设置为 true，如果组件还没渲染成功就发生了 url 变化，则 this._pendingLocation 存在，那么更新 this.state.location
8. componentWillUnmount：当这个组件销毁的时候当然要将注册的监听函数取消
9. render：返回 RouterContext.Provider 组件传入 
```
childre = this.props.children
value = {
  history: this.prop.history,
  location:this.state.location,
  match: Router.computeRootMatch(this.state.location.pathname),
  staticContext: this.props.staticContext
}
```
10. 如果是开发环境则会加上 propTypes 验证，并且每次更新都检查是否更新 history 如果更新的 history 和上次的不同则抛出警告说不能更改 history
### matchPath.js
因为接下来要说 Route 而 Route 涉及到路由匹配，而 React-router 的路由匹配就是通过这个来做的

先来看导入的包和缓存
1. path-to-regexp -> pathToRegexp 这是一个传入路由生成正则匹配的工具，具体可以看[path-to-regexp](https://www.npmjs.com/package/path-to-regexp 'path-to-regexp')
2. 定义了一个缓存对象 cache，将匹配过的路由缓存，如果下次再次访问这个路由则不用进行生成直接返回缓存中的数据
3. 定义了 cacheLimit 缓存最大限制
4. 定义了 cacheCount 用来记录缓存的数量
#### compliePath
这个函数就是生成路由正则和缓存正则的函数 参数是 path/options
1. 创建一个字符串 cacheKey 由传入的 option 生成
2. 将`cache[cachekey]`这个值赋值给 pathCache 如果`cache[cachekey]`不存在则 `pathCache = cache[cachekey] = {}`
3. 如果`pathCache[path]`存在则直接返回
4. 创建一个 keys 空数组，然后通过 pathToRegexp 传入path，key，options创建一个 regexp
5. 将结果储存在 {regexp,keys} 赋值给 result
6. 如果缓存还没有超过限制则加入缓存中并且缓存计数+1
7. 将 result 返回
#### matchPath
这个函数就是用来匹配路由的，参数是 pathname(匹配的路径)、options(配置参数)

我们把步骤分为 return 外和 return 内

首先 return 外
1. 如果 options 是字符串则将 options 设置为 {path: options}
2. 从 options 中结构赋值出(括号内代表默认值) path/exact(false)/strict(false)/sensitive(false)
3. 创建一个 paths 数组值 为 [].concat(path)

> 说明：options 下参数代表的意思 path(路由)、exact(是否完全匹配)、strict(如果为true，则regexp允许可选的尾部定界符匹配)、sensitive(如果为true，则正则表达式将区分大小写。)

return 内

matchPath 最终返回 paths.reduce 的结果，匹配到了的话则会返回一个对象，没有匹配到则会返回 null 我们直接来看循环内

1. 首先 reduce 的初始值设置为 null
2. 看循环内，如果上一次的结果不为 null 则直接返回 matched
3. 将 循环到的路由path/配置 传入 compilePath 生成 regexp 和 路由参数 keys 比如 '/arr/:id' 会生成 匹配 /arr 的路由正则 和 keys [{name: 'id' ....}]
4. 将传入的 pathname 进行匹配正则如果 Boolean(match) 为 false 则返回 null
5. 从 match 中解构出 url、values 数组
6. 判断 匹配的 url 是否和 pathname 全等
7. 如果参数设置了全等，url 和 pathname 不是全等则返回 false
8. 匹配成功则最终返回路径对象

这里需要注意的就是 paths = [].concat(path);

concat 比较神奇 如果传入的参数不是数组则会直接在数组中加入这个参数，所以这里用结构赋值不行
```
[...'123'] // ['1', '2', '3']
[].concat('123') // ['123']
[].concat(() => {}) // [()=>{}]
[...(()=>{})] // error
```
### Route.js
首先他定义了一个函数 isEmptyChildren 用来判断传入的参数是否没有一个组件

React.Children.count(children) 返回 children 中组件总数量

因为最终显示的都是 Route 所以我们接下来来看 Route，Route 直接调用render返回个了一个 DOM，这个 DOM 用 consumer 直接拿到 context 内容，然后我们来看 Consumer 中都做了什么

1. 判断 context 是否存在不存在直接报错
2. 如果 Route 没有传入 location 则取 context.location 赋值给 location
3. 定义 match 存放是否匹配到这个路由 首先确认有没有传入 computedMatch 如果存在则赋值为这个因为 Switch 会传入这个值，如果不存在则判断 props.path 是否存在，存在则使用 matchPath 匹配结果并且获取，不存在则直接使用 context.match
4. 将 context 结构赋值给 props 并将 location 和 match 属性覆盖
5. 从 props 属性中获取 children Component render
6. 如果 children 是数组并且长度为 0 则直接把 children 赋值为 null
7. 如果 children 是 function 则传入 props 之后调用结果再赋值给children，此时如果 children 为 undefined 则会将 children 赋值为 null 如果是开发环境还会抛出警告
8. 最后的 return 中 又创建了一个 Provider 并将 props 传入
9. 最后这个判断还是比较明显的 就是判断这个那个最后显示,我们再整理一下就比较清晰了
```
if (children && !isEmptyChildren(children)) {
  return chidren
} else {
  if (props.match) {
    if(component) {
      return React.createElement(component, props);
    } else {
      if (render) {
        return render(props);
      } else {
        return null;
      }
    }
  } else {
    return null
  }
}
```
最后如果是开发环境则还做了 props 的 type 检测，并且在渲染完成和更新的时候加入一些警告

我们可以看到这里用到的 React API
```
React.createElement(
  type,
  [props],
  [...chidren]
);

// 创建并返回指定类型的新 React 元素。其中类型参数既可以是标签名字符串('div')，也可以是 React组件类型 (class/function) 或是 React.fragment(<></>) 类型
```