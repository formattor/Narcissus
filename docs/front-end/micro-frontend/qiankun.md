# 路由监听获取资源
在single-spa的基础上进行了封装

# 资源加载
将single-spa的js entry替换为html entry

将子应用打包出来HTML作为入口，主框架通过fetch html的方式获取子应用静态资源，将HTML document作为子节点塞到主框架容器中，减少了主应用接入成本，子应用打包不需要调整，解决子应用之间的样式隔离问题。

# 样式隔离

通过主子应用之间的一些默认约定去规避冲突:不智能，旧项目不行

Shadow DOM： 根节点创建一个shadow root

scoped： 为子应用的根节点添加一个特定的随机属性,遍历style节点添加前缀

``` js
registerMicroApps({
    name:app1,
    sandbox:{
        // shadow dom
        strictstyleIsolation:true
        // scoped
        // experimentalStyleIsolation:true
    }
)
```


# JS隔离

::: details SnapshotSandbox

``` js
class Snapshotbox {
    windowSnapshot = {}
    modifyPropsMap = {}
    // 微应用运行状态
    active() {
        // 1.保存window对象上所有属性的状态
        for (const prop in window) {
            this.windowSnapshot[prop] = window[prop]
        }
        // 2.恢复上次运行该应用所修改过的window上的属性
        Object.keys(this.modifyPropsMap).forEach(prop => {
            window[prop] = this.modifyPropsMap[prop];
        })
    }
    // 微应用停止状态
    inactive() {
        for (const prop in window) {
            if (window[prop] !== this.windowSnapshot[prop]) {
                // 3.记录修改了window上的哪些属性
                this.modifyPropsMap[prop] = window[prop]
                // 4.还原window上微应用运行之前的状态
                window[prop] = this.windowSnapshot[prop]
            }
        }
    }
}

// window.city = 'beijing'

// 进入微应用

// window.city = 'shanghai'

// 退出微应用

// window.city = 'beijing'

window.city = 'beijing'

let snapSandbox = new Snapshotbox();

snapSandbox.active();

window.city = 'shanghai'

snapSandbox.inactive();

console.log(window.city); //beijing

snapSandbox.active();

console.log(window.city); // shanghai

// 遍历window所有属性，性能差
// 同一时间只能激活一个微应用
```

:::

::: details LegacySandbox

``` js
class LegacySandbox {
    currentUpdatePropsValueMap = new Map();
    modifiedPropsOriginalValueMap = new Map();
    addedPropsMap = new Map();
    proxyWindow = {}

    constructor() {
        const fakeWindow = Object.create(null);
        this.proxyWindow = new Proxy(fakeWindow, {
            set: (target, prop, value, receiver) => {
                const originalVal = window[prop];
                if (!window.hasOwnProperty(prop)) {
                    this.addedPropsMap.set(prop, value)
                } else if (!this.modifiedPropsOriginalValueMap.has(prop)) {
                    this.modifiedPropsOriginalValueMap.set(prop, originalVal)
                }
                this.currentUpdatePropsValueMap.set(prop, value);
                window[prop] = value;
            },
            get: (target, prop, receiver) => {
                return window[prop]
            }
        })
    }

    setWindowProp(prop, value, isToDelete) {
        if (value === undefined && isToDelete) {
            delete window[prop];
        } else {
            window[prop] = value;
        }
    }
    active() {
        // 回复上一次window运行状态
        this.currentUpdatePropsValueMap.forEach((value, prop) => {
            this.setWindowProp(prop, value);
        })
    }
    inactive() {
        // 还原window原有属性
        this.modifiedPropsOriginalValueMap.forEach((value, prop) => {
            this.setWindowProp(prop, value);
        })
        // 删除新增属性
        this.addedPropsMap.forEach((_, prop) => {
            this.setWindowProp(prop, undefined, true);
        })
    }
}


window.city = 'beijing'

let legacySandbox = new LegacySandbox();

legacySandbox.active();

legacySandbox.proxyWindow.city = 'shanghai'

legacySandbox.inactive();

console.log(window.city); //beijing

legacySandbox.active();

console.log(window.city); // shanghai

// 没有遍历对象的属性
// 也只能一个实例
``` 

:::

::: details ProxySandbox

``` js
class ProxySandbox {
    proxyWindow = {}
    isRunning = false;

    constructor() {
        const fakeWindow = Object.create(null);
        this.proxyWindow = new Proxy(fakeWindow, {
            set: (target, prop, value, receiver) => {
                if (this.isRunning) {
                    target[prop] = value;
                }
            },
            get: (target, prop, receiver) => {
                return prop in target ? target[prop] : window[prop];
            }
        })
    }

    active() {
        this.isRunning = true;
    }
    inactive() {
        this.isRunning = false;
    }
}

window.city = 'beijing'

let proxySandbox1 = new ProxySandbox();
let proxySandbox2 = new ProxySandbox();

proxySandbox1.active();
proxySandbox2.active();

proxySandbox1.proxyWindow.city = 'shanghai'
proxySandbox2.proxyWindow.city = 'nanjing'

proxySandbox1.inactive();

console.log(window.city); //beijing

proxySandbox1.active();

console.log(proxySandbox1.proxyWindow.city); // shanghai

console.log(proxySandbox2.proxyWindow.city); // nanjing

// 不需要遍历window
// 可以激活多个应用
```
:::
# 应用通信

父应用
``` js
import { initGlobalState } from 'qiankun'
const actions = initGlobalState(state);
// 主项目项目监听和修改
actions.onGlobalStateChange((state, prev) => {
  // state: 变更后的状态; prev 变更前的状态
  console.log(state, prev);
});
actions.setGlobalState(state);
```
子应用
``` js
export async function mount(props: any) {
    props.onGlobalStateChange((state, prev) => {
        // state: 变更后的状态; prev 变更前的状态
        console.log(state, prev);
        // 将这个state存储到我们子应用store
    });
    props.setGlobalState({ count: 2 });
    // render(props);
}
```

# 原理

### 父应用使用

``` js
const apps = [
  {
    name: "sub-react", 
    entry: '//localhost:3001',
    activeRule: "/sub-react",
    container: "#sub-app"
  },
  {
    name: "sub-vue",
    entry: '//localhost:3002',
    activeRule: "/sub-vue",
    container: "#sub-app"
  }
]

registerMicroApps(apps)

start()
```

### 注册微应用

``` js
// index.js
let _apps = [];

export const getApps = () => _apps;

// 01.register micro apps
export const registerMicroApps = (apps, lifestyles) => {
    _apps = apps;
}
```

### 启动微应用

1. 启动

``` js
// index.js
// 01.start app
export const start = () => {
    rewriteRouter();
    handleRouter();
}
```

2. 监听路由变化

``` js
// rewrite-router.js
// 02.listen router changed
export const rewriteRouter = () => {
    window.addEventListener('popstate', (e) => {
        handleRouter();
    })

    const rawPushState = window.history.pushState;
    window.history.pushState = (...args) => {
        prevRoute = window.location.pathname;
        rawPushState.apply(window.history, args)
        nextRoute = window.location.pathname;
        handleRouter();
    }

    const rawReplaceState = window.history.replaceState;
    window.history.replaceState = (...args) => {
        prevRoute = window.location.pathname;
        rawReplaceState.apply(window.history, args)
        nextRoute = window.location.pathname;
        handleRouter();
    }
}
```

::: tip 监听路由变化

+ hashChange - hash模式

+ popstate - history模式

    + popstate: history.go history.back history.forward

    + pushState: 添加路由

    + replaceState: 替换路由

:::

3. 获取子应用资源

::: details 获取子应用资源

3.1 路由资源获取

``` js
// handle-router.js
export const handleRouter = async () => {
    let apps = getApps()

    // 01.get prev path
    const prevApp = apps.find(item => getPrevRoute().startsWith(item.activeRule));
    // 02.get current path
    const currentApp = apps.find(item => getNextRoute().startsWith(item.activeRule));

    if (prevApp) {
        // unmout the prev app if exist
        await unmount(prevApp);
    }

    if (!currentApp) {
        return;
    }

    // 03. get micro app DOM,scripts,script function 
    const { template, getExternalScripts, execScripts } = await importHTML(currentApp.entry);
    let container = document.querySelector(currentApp.container);
    console.log(container);
    // 04. append micro app DOM to container
    container.appendChild(template);
}
```

3.2 获取路由资源

``` js
// handle-router.js
export const importHTML = async (url) => {
    // 07.get sub reource by fetch
    let html = await fetchResource(url);
    const template = document.createElement('div');
    template.innerHTML = html;

    const scripts = document.querySelectorAll('script');
    console.log('08.get all sub js resources', scripts);

    function getExternalScripts() {
        return Promise.all(Array.from(scripts).map(script => {
            const src = script.getAttribute('src');
            if (!src) {
                return Promise.resolve(script.innerHTML)
            } else {
                return fetchResource(
                    src.startsWith('http') ? src : `${url}${src}`
                );
            }
        }))
    }

    async function execScripts() {
        const scripts = await getExternalScripts()

        const module = { exports: {} }
        const exports = module.exports;

        scripts.forEach(code => {
            eval(code);
        })

        return module.exports;
    }

    return {
        template,
        getExternalScripts,
        execScripts
    }
}
```

3.3 fetch funtion

``` js
// fetch-resource.js
export const fetchResource = url => fetch(url).then(res=> res.text())
```

:::

4. 执行子应用scripts

``` js
// handle-router.js
export const handleRouter = async () => {

    ...

    // 添加该属性，判断子应用是否被qiankun加载
    window.__POWERED_BY_QIANKUN__ = true;

    const appExports = await execScripts();

    currentApp.bootstrap = appExports.bootstrap;
    currentApp.mount = appExports.mount;
    currentApp.unmount = appExports.unmount;

    await bootstrap(currentApp)
    await mount(currentApp)
}

async function bootstrap(app) {
    app.bootstrap && (await app.bootstrap())
}

async function mount(app) {
    app.mount && (await app.mount({
        container: document.querySelector(app.container)
    }))
}

async function unmount(app) {
    app.unmount && (await app.unmount({
        container: document.querySelector(app.container)
    }))
}

```