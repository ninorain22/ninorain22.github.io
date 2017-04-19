---
layout: post
title: React-Router学习笔记
tags:
- React-Router
date: 2017-04-19 11:00:00.000000000 +09:00
---

## 基本用法
安装
```
$ npm install -S react-router
```

```js
import { Router } from 'react-router';
render(<Router/>, document.getElementById('app'));
```

``Router``组件本身只是一个容器，真正的路由要通过``Route``组件定义

```js
import { Router, Route, hashHistory } from 'react-router';

render((
  <Router history={hashHistory}>
    <Route path="/" component={App}/>
    <Route path="/repos" component={Repos}/>
    <Route path="/about" component={About}/>
  </Router>
), document.getElementById('app'));
```

## 嵌套路由
```js
<Router history={hashHistory}>
  <Route path="/" component={App}>
    <Route path="/repos" component={Repos}/>
    <Route path="/about" component={About}/>
  </Route>
</Router>
```
App要写成下面的样子：
```js
export default React.createClass({
  render() {
    return <div>
      {this.props.children}
    </div>
  }
})
```
App组件的``this.props.children``属性就是子组件。

## Path属性
略

## 通配符
```js
<Route path="/hello/:name">
// 匹配 /hello/michael
// 匹配 /hello/ryan

<Route path="/hello(/:name)">
// 匹配 /hello
// 匹配 /hello/michael
// 匹配 /hello/ryan

<Route path="/files/*.*">
// 匹配 /files/hello.jpg
// 匹配 /files/hello.html

<Route path="/files/*">
// 匹配 /files/ 
// 匹配 /files/a
// 匹配 /files/a/b

<Route path="/**/*.jpg">
// 匹配 /files/hello.jpg
// 匹配 /files/path/to/file.jpg
```

规则如下：
1. :paramName
:paramName匹配URL的一个部分，直到遇到下一个/、?、#为止。这个路径参数可以通过``this.props.params.paramName``取出
2. ()
()表示URL的这个部分是可选的
3. *
*匹配任意字符，直到模式里面的下一个字符为止。匹配方式是非贪婪模式
4. **
** 匹配任意字符，直到下一个/、?、#为止。匹配方式是贪婪模式

path属性也可以使用相对路径（不以/开头），匹配时就会相对于父组件的路径，可以参考上一节的例子。嵌套路由如果想摆脱这个规则，可以使用绝对路由

**由匹配规则是从上到下执行，一旦发现匹配，就不再其余的规则了**

## IndexRoute组件
```js
<Router>
  <Route path="/" component={App}>
    <IndexRoute component={Home}/>
    <Route path="accounts" component={Accounts}/>
    <Route path="statements" component={Statements}/>
  </Route>
</Router>
```

访问``/``的时候，加载的组件结构如下：
```js
<App>
  <Home/>
</App>
```

## Redirect组件
```js
<Route path="inbox" component={Inbox}>
  {/* 从 /inbox/messages/:id 跳转到 /messages/:id */}
  ＜Redirect from="messages/:id" to="/messages/:id" />
</Route>
```
现在访问``/inbox/messages/5``，会自动跳转到``/messages/5``

## IndexRedirect 组件
IndexRedirect组件用于访问根路由的时候，将用户重定向到某个子组件
```js
<Route path="/" component={App}>
  ＜IndexRedirect to="/welcome" />
  <Route path="welcome" component={Welcome} />
  <Route path="about" component={About} />
</Route>
```

## Link
Link组件用于取代``<a>``元素，生成一个链接，允许用户点击后跳转到另一个路由
它基本上就是<a>元素的React 版本，可以接收Router的状态

## IndexLink
如果链接到根路由``/``，不要使用Link组件，而要使用IndexLink组件
是因为对于根路由来说，activeStyle和activeClassName会失效，或者说总是生效，因为/会匹配任何子路由。而IndexLink组件会使用路径的精确匹配

## histroy 属性
Router组件的history属性，用来监听浏览器地址栏的变化，并将URL解析成一个地址对象，供 React Router 匹配。
history属性，一共可以设置三种值
+ browserHistory: 路由将通过URL的hash部分（``#``）切换，URL的形式类似``example.com/#/some/path``
+ hashHistory: 浏览器的路由就不再通过Hash完成了，而显示正常的路径``example.com/some/path``
但是，这种情况需要对服务器改造。否则用户直接向服务器请求某个子路由，会显示网页找不到的404错误
+ createMemoryHistory: 主要用于服务器渲染。它创建一个内存中的``history``对象，不与浏览器URL互动

## 表单处理
```js
<form onSubmit={this.handleSubmit}>
  <input type="text" placeholder="userName"/>
  <input type="text" placeholder="repo"/>
  <button type="submit">Go</button>
</form>
```
**方法一： ``browserHistory.push``**
```js
import { browserHistory } from 'react-router'

// ...
  handleSubmit(event) {
    event.preventDefault()
    const userName = event.target.elements[0].value
    const repo = event.target.elements[1].value
    const path = `/repos/${userName}/${repo}`
    browserHistory.push(path)
  },
```

**方法二： ``context``对象**
略

## 路由的钩子
每个路由都有Enter和Leave钩子，用户进入或离开该路由时触发
```js
<Route path="about" component={About} />
＜Route path="inbox" component={Inbox}>
  ＜Redirect from="messages/:id" to="/messages/:id" />
</Route>
```
使用``onEnter``钩子替代``<Redirect>``组件:
```js
<Route path="inbox" component={Inbox}>
  <Route
    path="messages/:id"
    onEnter={
      ({params}, replace) => replace(`/messages/${params.id}`)
    } 
  />
</Route>
```

``onEnter``钩子还可以用来做**认证**
```js
const requireAuth = (nextState, replace) => {
    if (!auth.isAdmin()) {
        // Redirect to Home page if not an Admin
        replace({ pathname: '/' })
    }
}
export const AdminRoutes = () => {
  return (
     <Route path="/admin" component={Admin} onEnter={requireAuth} />
  )
}
```

当用户离开一个路径的时候，跳出一个提示框，要求用户确认是否离开:
```js
const Home = withRouter(
  React.createClass({
    componentDidMount() {
      this.props.router.setRouteLeaveHook(
        this.props.route, 
        this.routerWillLeave
      )
    },

    routerWillLeave(nextLocation) {
      // 返回 false 会继续停留当前页面，
      // 否则，返回一个字符串，会显示给用户，让其自己决定
      if (!this.state.isSaved)
        return '确认要离开？';
    },
  })
)
```



