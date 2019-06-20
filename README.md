# projectOptimization
项目的性能优化

by 2019/1/25 10:00
----------------------------------
这几天项目转测，闲了一点，就来研究下前端优化的需求，从之前做的项目和自己搭框架的情况来看的话，一直以来在我写代码的时候，前端性能优化一直是遇到了阻塞或者较长延迟才会想去解决的事情，很惭愧=  =平时能做到的也只是在合适的生命周期中做相应的操作、v-if/v-show使用场景之类、避免js直接操作dom这类很小的优化吧，但当一个SPA或者前端项目越做越大时，合理的网络、缓存以及压缩等方面的优化会让项目在生产环境下表现的更有效率。  

### 1.webpack打包
在我们项目中有这样一个场景，就是我们是基于express的vue项目，在当修改node中的文件后需要重新启动一下整个dev，由于我们的项目很大，每次重启就会浪费很多时间，很痛苦。所以针对这种场景做了以下几种优化方式：
#### 1.配置externals
Webpack可以配置externals来将依赖的库指向全局变量，从而不再打包这个库，比如我们在main.js文件中：
```javascript
import Vue from	'vue' // 引用vue类
```

如果你在Webpack.config.js中配置了externals：
```javascript
module.exports = {
    externals: {
        'vue': 'window.Vue'
    }
    //其它配置忽略...... 
};
```
这段配置等于让Webpack知道，对于vue这个模块就不要打包了，直接指向window.Vue就好。不过别忘了在index.html中加载 vue.min.js，让全局中有Vue这个变量。  

但是这种配置方式从过程上看很明显有个缺陷，对于没有提供生产环境的文件也就是没有vue.min.js这类文件的js库来说是不可以的，特别是某些基于vue的组件库内部会有import Vue from 'vue'这类操作，webpack还是会认真的把vue重新打包一遍，因此这种解决方法需要升级。

### 2.路由懒加载
其实对于Vue前端来说，一共有三种实现懒加载的方式，只要达到异步按需请求vue组件的目的都可以成为路由懒加载。

#### 1.vue异步组件技术
vue-router配置路由，使用vue的[异步组件](https://cn.vuejs.org/v2/guide/components-dynamic-async.html#%E5%BC%82%E6%AD%A5%E7%BB%84%E4%BB%B6?_blank)技术，可以实现按需加载。  
```javascript
{
    path: '/promisedemo',
    name: 'PromiseDemo',
    component: resolve => require(['../components/PromiseDemo'], resolve)
}
```  

#### 2.es6提案的import()
这种异步加载的方式就是我之前做项目用的懒加载方式（需要webpack>2.4），具体配置可以看看我之前整理的[项目记录](https://github.com/Alexandermclean/Security-Cloud-Project/blob/master/Readme.md#1%E8%B7%AF%E7%94%B1router?_blank)，import方式相比于require，import是编译时加载，输出的是对一个模块的引用，效率更高。
* webpack官方文档：[webpack中使用import()](https://webpack.docschina.org/guides/code-splitting/#%E5%8A%A8%E6%80%81%E5%AF%BC%E5%85%A5-dynamic-imports-?_blank)  
* vue官方文档：[路由懒加载使用import()](https://router.vuejs.org/zh/guide/advanced/lazy-loading.html?_blank)  
```javascript
// 下面2行代码，没有指定webpackChunkName，每个组件打包成一个js文件。
const ImportFuncDemo1 = () => import('../components/ImportFuncDemo1')
const ImportFuncDemo2 = () => import('../components/ImportFuncDemo2')

// 下面2行代码，指定了相同的webpackChunkName，会合并打包成一个js文件。
// const ImportFuncDemo = () => import(/* webpackChunkName: 'ImportFuncDemo' */ '../components/ImportFuncDemo')
// const ImportFuncDemo2 = () => import(/* webpackChunkName: 'ImportFuncDemo' */ '../components/ImportFuncDemo2')

export default new VueRouter({
    routes: [
        {
            path: '/importfuncdemo1',
            name: 'ImportFuncDemo1',
            component: ImportFuncDemo1
        },
        {
            path: '/importfuncdemo2',
            name: 'ImportFuncDemo2',
            component: ImportFuncDemo2
        }
    ]
})
```

### 3.webpack提供的require.ensure()
vue-router配置路由，使用webpack的[require.ensure](https://webpack.docschina.org/api/module-methods#require-ensure?_blank)技术，也可以实现按需加载。但不太推荐使用，这里只简单说一哈。  
```javascript
{
    path: '/promisedemo',
    name: 'PromiseDemo',
    component: resolve => require.ensure([], () => resolve(require('../components/PromiseDemo')), 'demo')
},
{
    path: '/hello',
    name: 'Hello',
    // component: Hello
    component: resolve => require.ensure([], () => resolve(require('../components/Hello')), 'demo')
}
```
> 指定chunkName的情况下，上面的都指向了demo这个chunk，和import方式一样，因此会合并打包成一个js文件。