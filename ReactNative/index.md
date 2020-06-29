# mobx 在 Next.js 中使用涉及三个插件：

mobx
mobx-react
mobx-react-lite
如果对Mobx不熟悉，建议先看文档 MobX 入门。

如果对React Context不熟悉，建议先看文档 React Context



项目中需要使用Mobx实现的功能
store分为全局store、页面store和组件store：

语言、货币数据保存在全局store，所有js代码（包括非组件的js插件）共享；
页面store为单个页面自有store模块，仅供页面自身使用；
组件store为组件使用的store模块，如多个页面共用的组件或拆分层级较多的组件；


所有组件中访问全局变量
跨组件传递需要借助Context，所有组件都需要读取这个Context变量，因此新建一个文件保存变量，方便各组件引入

新建 utils/store.ts 文件，创建 storeContext变量
```
import { createContext, useContext } from 'react';
​
export const storeContext = createContext(Object.create(null));
```

在最外层组件（pages/_app.tsx）中通过Provider 将需要共享的变量注入
```
import { storeContext } from '@/utils/store';
import { useLocalStore } from 'mobx-react-lite';
import App from 'next/app';
​
function MyApp(props) {
   const { Component, pageProps, lang } = props;
   const store = useLocalStore(() => ({
       rootStore: { lang }
  }));
   return (
       <storeContext.Provider value={store}>
           <Component {...pageProps} />
       </storeContext.Provider>
  )
}

MyApp.getInitialProps = async (appContext) => {
   const appProps = await App.getInitialProps(appContext);
   // 服务端获取 lang 的值，通过 props 传递给 MyApp 组件
   const lang = appContext.ctx.req.query.lng;
   return { ...appProps, lang }
};
​
export default MyApp;
```

页面组件通过 useContext hook 读取变量
```
import { storeContext } from '@/utils/store';
​
function Page() {
   const store = React.useContext(storeContext);
   return <div>Current Lang: {store.rootStore.lang}</div>
}
​
export default Page;
```

封装 React.useContext 方法，utils/store.ts 中添加接口 useStore
```
import { useContext } from 'react';
​
export const useStore = () => {
   const store = useContext(storeContext);
   if (!store) {
       // this is especially useful in TypeScript so you don't need to be checking for null all the time
       throw new Error('useStore must be used within a StoreProvider.');
  }
   return store;
};
```
组件中使用
```
import { useStore } from '@/utils/store';
​
function Page() {
   const { rootStore } = useStore();
   return <div>lang in store: {rootStore.lang}</div>
}
```


非组件js读取全局store
以 ajax请求插件api.js 为例，插件需要同时支持服务端和客户端发起ajax请求，并且url格式为 http(s)://api.com/en/getXXX，当前语言必须写在域名后面。

在上面的组件示例中，读取全局变量用到的 React.useContext 只能用在 hook 中，api.js 不是组件，没办法通过 Context 读取。

因此要实现共享，只能是共享一个同时能被组件和非组件使用的对象。

在 MobX 入门 中提到了：

useLocalStore 是基于observable的封装，下面两种写法作用是一样的
```
const [person] = useState(() => observable({ name: 'John' }));
const person = useLocalStore(() => ({ name: 'John' }));
```
下面尝试共享 observable 对象来实现：

utils/store.ts 中增加用于共享的 observableStore
```
import { observable } from 'mobx';
​
// 所有组件共享的 store对象
export const observableStore = observable({
   rootStore: { lang: 'en' }
});
```

修改pages/_app.tsx
```
import { storeContext, observableStore } from '@/utils/store';
import { useState } from 'react';
​
function MyApp(props) {
   const { Component, pageProps, lang } = props;
   const [store] = useState(() => observableStore);
   store.rootStore.lang = lang;
​
   return (
       <storeContext.Provider value={store}>
           <Component {...pageProps} />
       </storeContext.Provider>
  )
}
```

新建 utils/api.js
```
import { observableStore } from '@/utils/store';
​
export default {
   getLang() {
       return observableStore.rootStore.lang
  }
}
```

修改页面组件
```
import { useStore } from '@/utils/store';
import api from '@/utils/api';
​
function Page(props) {
   const {rootStore} = useStore();
   return (
       <div>
          lang in store: {rootStore.lang}<br/>
          lang in component api: {api.getLang()}<br/>
          lang in getServerSideProps api: {props.langInServer}
       </div>
  )
}
​
export function getServerSideProps() {
   return {
       props: {
           langInServer: api.getLang()
      }
  }
}
​
export default Page;
```

getServerSideProps 传递给组件的 langInServer出现了问题：

url中的语种由en改为de，显示：lang in getServerSideProps api: en

刷新后变成了 lang in getServerSideProps api: de

根据现象得出结论：

服务端 rootState 变量值被储存了，刷新页面后没有初始化；

Page组件中服务端和客户端都获取到了更新后的rootStore数据，但是getServerSideProps 中只能获取更新前的数据；



刷新页面时初始化 rootState
修改utils/store.ts 文件，添加 initStore 方法
```
// 创建一个新的 store 对象
const createStore = (rootStoreData) => observable({
    rootStore: Object.assign({
        lang: 'en'
    }, rootStoreData)
});
​
// 所有组件共享的 store对象
export let observableStore = null;
​
// store 初始化
export const initStore = (rootStoreData) => {
    observableStore = createStore(rootStoreData);
}
```

修改 pages/_app.tsx
```
import { storeContext, observableStore, initStore } from '@/utils/store';
import { useState } from 'react';
import App from 'next/app';
​
function MyApp(props) {
    const { Component, pageProps, lang } = props;
​
   initStore({ lang });
​
    const [store] = useState(() => observableStore);
​
    // return ...
}
​
MyApp.getInitialProps = async (appContext) => {
    const lang = appContext.ctx.req.query.lng;
    // 服务端初始化store
    // MyApp.getInitialProps 会在页面组件的 getServerSideProps 执行，可以解决页面组件服务端取不到最新值的问题
    initStore({ lang });
    //...
};
​
export default MyApp;
```

store模块动态注册
网上关于 store 的分享，最常见的是将所有 store 文件放在同一目录中，在初始化时注册所有模块。

我想要的使用方式是，初始化时仅注册 rootStore，各页面或组件的 store 模块动态注册。

mobx 提供了一个 extendObservable 方法，可以实现该功能。

utils/store.ts 中添加 addStore 方法
```
// 动态增加 store 模块
export const addStore = (moduleName, moduleData) => {
   const store = useStore();
   if (!store[moduleName]) {
       const module = Object.create(null);
       module[moduleName] = moduleData;
       extendObservable(store, module);
  }
   return store;
};
```
组件示例
```
import { addStore } from '@/utils/store';
import { useObserver } from 'mobx-react-lite';
import { useEffect } from 'react';
​
function createStore(defaultPoints) {
   return {
       points: defaultPoints,
       get pointsTip() {
           return 'Total Points: ' + this.points;
      },
       setPoints(num) {
           this.points = num;
      },
       addPoints() {
           this.points += 1;
      }
  }
}
​
function Page(props) {
   // 动态添加页面 store 模块
   const { thePage } = addStore('thePage', createStore(props.defaultPoints));
​
   return useObserver(() => (
           <div onClick={() => thePage.addPoints()}>
              {thePage.pointsTip}
           </div>
      )
  )
}
​
// 服务端返回初始值
export function getServerSideProps() {
   return {
       props: {
           defaultPoints: 50
      }
  }
}
​
export default Page;
```

进一步优化封装
变量私有化

store.ts 中 export 了两个对象 storeContext 和 observableStore，改为 export hook组件
```
export const StoreProvider = ({ children }) => {
   const [store] = useState(() => observableStore);
   return <storeContext.Provider value={store}>{children}</storeContext.Provider>;
};
```
_app.tsx 使用方式修改
```
import { StoreProvider, initStore } from '@/utils/store';
​
function MyApp(props) {
   const { Component, pageProps, lang } = props;
   
   initStore({ lang });
​
   return <StoreProvider><Component {...pageProps} /></StoreProvider>
}
```

改写 useStore 接口，支持非组件调用
```
export const useStore = () => {
   let store;
   try {
       store = useContext(storeContext);
  } catch (e) {
       // 非 hook 组件中会不能使用 useContext，直接返回 observableStore
       store = observableStore;
  }
   if (!store) {
       throw new Error('Store has not been initialized.');
  }
   return store;
};
```

api.js 使用方法
```
import { useStore } from '@/utils/store';
​
export default {
   getLang() {
       const { rootStore } = useStore();
       return rootStore.lang
  }
}
```

至此，已满足基本开发所需功能。
