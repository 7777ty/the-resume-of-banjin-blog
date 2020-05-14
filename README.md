## 板筋博客---多人开发共享博客总结

### 开发流程
本次多人共享博客使用Vue.js进行开发，共有以下流程：
1. 使用Vue-cli创建项目
2. 编写基础样式、封装底层方法，设计路由逻辑
3. 编写静态页面
4. 完成各静态页面JS内容
5. 页面完善
6. 项目发布

### 实施细则

#### 1. 项目创建
使用Vue-cli搭建项目，没有太多可赘述内容。


#### 2. 项目准备
项目准备的工作主要有以下三方面内容：编写基础样式，封装底层方法，设计路由逻辑。

2.1 编写基础样式基础样式主要是对页面默认内容的重置以及提取出一些公共样式。主要有以下两个scss文件：

```scss
//reset.scss
* {
  margin: 0;
  padding: 0;
  box-sizing: border-box;
}
a{
  text-decoration: none;
  color: inherit;
}
ul,ol{
  list-style: none;
}
button,input{
  font:inherit;
}
:focus{
  outline: none;
}
h1,h2,h3,h4,h5,h6{
  font-weight: normal;
}

```
创建reset.scss之后，在App.vue引入`@import "~@/assets/styles/reset.scss";`，重置整个项目的页面效果。
```scss
//base.scss
$themeColor: rgb(0, 153, 0);
$themeLighterColor: rgba(0, 153, 51, 0.6);
$bgColor: #149739;
$textLighterColor: #999;
```
--------------------

2.2 封装底层方法 本项目的接口规范由饥人谷提供，如有兴趣，请点击:[共享博客前后端接口规范](https://xiedaimala.com/tasks/0e61bf37-d479-481b-a43e-8d7dd6069f93/text_tutorials/606cfb19-ca16-4fec-8564-75c1979871d6)。
在获得了相关接口后，我选择将其进行封装，使其使用起来更为方便。
首先，我对使用Axios网络请求进行封装，虽然Axios是一个已经封装好的AJAX库，但为了让开发更为便利，我选择再次对Axios进行封装：
```js
import axios from 'axios';
import {Message} from "element-ui";

axios.defaults.headers.post['Content-Type'] = 'application/x-www-form-urlencoded';
axios.defaults.baseURL = 'https://blog-server.hunger-valley.com';
axios.defaults.withCredentials = true; //由于接口的跨域不会带上Cookie，所以在此允许此异步请求带上Cookie

export default function request(url,type='GET',data={}) {
    return new Promise((resolve, reject) => {
        const option={
            url,
            method:type
        };
        if(type.toLowerCase()==='get'){ //根据请求方法的不同，将参数放在不同位置
            option.params=data;
        }else{
            option.data=data;
        }
        axios(option).then(res=>{
            if(res.data.status==='ok'){
                resolve(res.data);
            }else{
                Message.error(res.data.msg);
                reject(res.data);
            }
        }).catch(()=>{
            Message.error('网络异常');
            reject({msg:'网络异常'})
        })
    })

}
```
在封装好之后为了使用是的语义化以及便利性更好，再次根据后端接口文档，将请求分为auth.js、blog.js两部分，分别存放了封装好的用户、博客请求函数。
```js
//auth.js
import request from "@/helper/request";
const URL={
    SIGNUP:'/auth/register',
    LOGIN:'/auth/login',
    LOGOUT:'/auth/logout',
    GET_INFO:'/auth'
};

export default {
    signUp({username,password}){
        return request(URL.SIGNUP,'POST',{username,password});
    },
    login({username,password}){
        return request(URL.LOGIN,'POST',{username,password});
    },
    logout(){
        return request(URL.LOGOUT);
    },
    getInfo(){
        return request(URL.GET_INFO);
    }
}
```
```js
//blog.js
import request from "@/helper/request";

const  URL={
    GET_LIST:'/blog',
    GET_DETAIL:'/blog/:blogId',
    CREATE:'/blog',
    UPDATE:'/blog/:blogId',
    DELETE:'/blog/:blogId'
};

export default {
    getBlogs({page=1,userId,atIndex}={page:1}){
        return request(URL.GET_LIST,'GET',{page,userId,atIndex});
    },
    getIndexBlogs({page=1}={page:1}){
        return this.getBlogs({page,atIndex:true})
    },
    getBlogsByUserId(userId,{page=1,atIndex}={page:1}){
        return this.getBlogs({userId,page,atIndex})
    },
    getDetail(blogId){
        return request(URL.GET_DETAIL.replace(':blogId',blogId));
    },
    updateBlog({blogId},{title,content,description,atIndex}){
        return request(URL.UPDATE.replace(':blogId',blogId),'PATCH',{title,content,description,atIndex})
    },
    deleteBlog({ blogId }) {
        return request(URL.DELETE.replace(':blogId', blogId), 'DELETE')
    },
    createBlog({ title = '', content = '', description = '', atIndex} = { title: '', content: '', description: '', atIndex: false}) {
        return request(URL.CREATE, 'POST', { title, content, description, atIndex })
    }
}
```

------------

2.3 设计页面逻辑 这一步骤主要是用于预设各页面跳转逻辑，以及判断各页面是否需要在用户登陆状态下才可访问。
出于对网页性能的考虑这里对VueRouter引入的页面组件实现了懒加载，接下来将通过懒加载的实现，
以及用户只有在登陆状态下才有权限访问页面的权限判断两部分对VueRouter的index.ts文件进行讲解：

一、懒加载
网上实现懒加载的方法很多，为了方便理解，我采用了其中一种书写方法与VueRouter正常加载写法相似的方法实现懒加载。

首先，在该ts文件内定义一个函数:
```ts
function loadView(view: string) {
  return () => import(/* webpackChunkName: "view-[request]" */ `@/views/${view}.vue`)
}
```
然后在VueRouter配置数组中各项的component进行调用即可:
```ts
const routes: Array<RouteConfig> = [
    {
      path:'/',
      redirect:'/home'
    },
    {
      path:'/home',
      name:'Home',
      component: loadView('Home')
    },
    {
      path: '/details',
      name:'Details',
      component:loadView('Details')
    }
]
```
二、权限判断
这里通过查阅VueRouter文档的路由元信息和路由守卫就可以实现。
首先在定义路由时配置meta信息，一个路由匹配到的所有路由记录会暴露为 $route 对象 (还有在导航守卫中的路由对象) 的 $route.matched 数组。
这样我们就可以通过在路由守卫中遍历$route.matched 来检查路由记录中的 meta 字段。即可实现权限判断
```ts
function loadView(view: string) {
  return () => import(/* webpackChunkName: "view-[request]" */ `@/views/${view}.vue`)
}
const routes: Array<RouteConfig> = [
    {
      path:'/',
      redirect:'/home'
    },
    {
      path:'/home',
      name:'Home',
      component: loadView('Home')
    },
    {
      path: '/details',
      name:'Details',
      meta:{requiresAuth:true}
      component:loadView('Details')
    }
];

//路由守卫对其进行requiresAuth判断
router.beforeEach((to, from, next) => {
  if (to.matched.some(record => record.meta.requiresAuth)) {
    store.dispatch('checkLogin').then(isLogin => {
      if (!isLogin) {
        next({
          path: '/login',
          query: {redirect: to.fullPath}
        })
      } else {
        next()
      }
    })
  }else {
  next() // 确保一定要调用 next()
}
});
```
#### 3.编写静态页面
静态页面的编写就不再赘述，按设计稿来就好，较为简单，使用flex布局。
值得一说的是，本次尝试使用了elementUI，真是十分方便。

#### 4.完成JS内容
在完成静态页面后，就可以通过调用之前封装好的请求方法进行网络请求，获取响应数据填充在页面上，
对于一些公共数据我们可以通过VueX进行状态管理。在博客详情展示页尝试使用了marked.js，以实现通过markdown语法编写的博客内容以对应样式展示。
此部分还添加了一个util.js文件用于在主页上更改用户发布的时间为？分钟前||？小时前||？天前。
```js
function friendlyDate(datsStr) {
    const dateObj = typeof datsStr === 'object' ? datsStr : new Date(datsStr);
    const time = dateObj.getTime();
    const now = Date.now();
    const space = now - time;
    let str = '';

    switch (true) {
        case space < 60000:
            str = '刚刚';
            break;
        case space < 1000*3600:
            str = Math.floor(space/60000) + '分钟前';
            break;
        case space < 1000*3600*24:
            str = Math.floor(space/(1000*3600)) + '小时前';
            break;
        default:
            str = Math.floor(space/(1000*3600*24)) + '天前';
    }
    return str
}

export default {
    install(Vue, options) {
        Vue.prototype.friendlyDate = friendlyDate
    }
}

```

#### 5.完善页面内容
在这一方面，主要是对样式的些许调整以及部分bug的修复。

##### 6.项目发布
最后，将项目发布到github上。
