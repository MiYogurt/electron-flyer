TypeScript 是微软开发的一款 JavaScript 的超集，提供静态语言类型，这样可以让我们的代码更加符合规范，在多人合作开发中非常有用，目前各大 BAT 公司，都有实践 TypeScript 语言，以下简称 ts。

你最好有一些 TypeScript 基础，没有的话，可以看下我录制的[免费视频](https://nodelover.me/course/ts-basic)，或者[开源书籍](https://github.com/MiYogurt/nodelover-books)，这只是七八本里面的一本。

## 安装依赖

可以参考 [electron-webpack 文档](https://webpack.electron.build/add-ons#typescript) ，它会告诉你更多的信息。

```shell
npm install typescript electron-webpack-ts --dev
```

`electron-webpack-ts` 里面其实没有任何代码，只是会帮你安装一些必要的依赖而已。然后初始化 `tsconfig.json` 配置文件，它规定了你写 ts 的规范，你必须全局安装了 `typescript` 才会有 `tsc` 命令，它可以编译 ts 文件。

```sh
tsc --init
```

修改 `tsconfig.json` 的文件内容，让它继承 `electron-webpack` 的配置。`types` 是为了做类型兼容的。

```json
{
  "extends": "./node_modules/electron-webpack/tsconfig-base.json",
  "compilerOptions": {
    "typeRoots": ["./src/main"]
  }
}
```

在介绍里面就有说明 rxjs，它其实是一个非常强大的工具，但是由于学习曲线的陡峭，只有少部分以 Angular 为开发栈的公司在使用它，比如 Daocloud，现在我们先安装它。

```sh
npm install rxjs --save
```

安装 `electron-util` 的定义文件，这个定义文件跟 API 对应是不全的，所以我们还需要自己魔改一下。

```sh
npm install @types/electron-util --save-dev
```

以插件形式添加类型可以 [参考这里](https://www.tslang.cn/docs/handbook/declaration-files/templates/module-plugin-d-ts.html) , 新建 `src/main/shim.d.ts` 文件

```typescript
import util from 'electron-util'
declare module 'electron-util' {
  export function loadFile(win?: any, path?: string): void
}
```

## 主进程入口

- 导入依赖

```typescript
import { app, BrowserWindow } from 'electron'
import util from 'electron-util'
import { fromEvent } from 'rxjs'
```

[fromEvent](https://rxjs-cn.github.io/learn-rxjs-operators/operators/creation/fromevent.html) 是 `rxjs` 的创建函数，以下两种方式等价，它会自动监听你的事件，通过 `subscribe` 来传递处理回调，类似于 `Promise` 的 `then`

```typescript
fromEvent(<any>win.webContents, 'devtools-opened').subscribe(() => {
  win.focus()
  setImmediate(() => win.focus())
})

win.webContents.on('devtools-opened', () => {
  win.focus()
  setImmediate(() => win.focus())
})
```

- 窗口创建函数

内容基本跟之前的一致，只不过加了 TypeScript 类型，完整代码可以参考源码仓库。

```typescript
let mainWindow: BrowserWindow | null

function createMainWindow(opts?: Electron.BrowserWindowConstructorOptions) {
  const win = new BrowserWindow(opts)

  if (util.is.development) {
    win.webContents.openDevTools()
    win.loadURL(`http://localhost:${process.env.ELECTRON_WEBPACK_WDS_PORT}`)
  } else {
    util.loadFile(win, 'index.html')
  }

  fromEvent(<any>win, 'close').subscribe(() => (mainWindow = null)) // 关闭事件

  fromEvent(<any>win.webContents, 'devtools-opened').subscribe(() => {
    // 打开开发者工具栏后
    win.focus()
    setImmediate(() => win.focus())
  })

  return win
}
```

- 事件处理函数

一个一个的写，不如写一个循环来帮我们做事件绑定。

```typescript
function ready(): void {
  mainWindow = createMainWindow()
}

function windowAllClosed(): void {
  if (util.is.macos) {
    app.quit()
  }
}

function activate(): void {
  if (!mainWindow) {
    mainWindow = createMainWindow()
  }
}

const AppEventMap: any = {
  ready,
  'window-all-closed': windowAllClosed,
  activate
}

Object.keys(AppEventMap).map(eventName =>
  fromEvent(<any>app, eventName).subscribe(AppEventMap[eventName])
)
```

通过一个 map 的映射，来自动监听事件。
