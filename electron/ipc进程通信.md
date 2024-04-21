# ipc进程通信

首先根据官方文档总结整理了几种进程通信的方式，简单明了，方便记忆。

### 1、渲染进程 → 主进程（单向）

方式一：ipcRenderer.send -> ipcMain.on（染进程发送消息至主进程，主进程监听）

```纯文本
ipcRenderer.send('my_channel', 'my_data');

ipcMain.on('my_channel', (event, message) => {
  console.log(`receive message from render: ${message}`) 
})

```

### 2、渲染进程 <-> 主进程（双向）

方式一：await ipcRenderer.invoke发送消息等待ipcMain.handle返回结果

```纯文本
// 渲染进程
const replyMessage = await ipcRenderer.invoke('my_channel', 'my_data');

// 主进程
ipcMain.handle('my_channel', async (event, message) => {
  console.log(`receive message from render: ${message}`);
  return 'replay';
});

```

方式二：ipcRenderer.send和ipcMain.on配合。这也是Electron 7没有 ipcRenderer.invoke 之前通过 IPC 进行异步双向通信的推荐方式。

```纯文本
渲染进程发送和监听返回事件
ipcRenderer.on('asynchronous-reply', (_event, arg) => {
  console.log(arg) // 在 DevTools 控制台中打印“pong”
})
ipcRenderer.send('asynchronous-message', 'ping')


主进程监听
ipcMain.on('asynchronous-message', (event, arg) => {
  console.log(arg) // 在 Node 控制台中打印“ping”
  // 作用如同 `send`，但返回一个消息
  // 到发送原始消息的渲染器
  event.reply('asynchronous-reply', 'pong')
})

```

方式三：ipcRenderer.sendSync 向主进程发送消息，并同步等待响。同步特性意味着它将阻塞渲染器进程

```纯文本
渲染进程发送
const result = ipcRenderer.sendSync('synchronous-message', 'ping')

主进程监听
ipcMain.on('synchronous-message', (event, arg) => {
  console.log(arg) // 在 Node 控制台中打印“ping”
  event.returnValue = 'pong'
})

```

### 3、主进程 → 渲染进程

方式一：webContents.send，主进程使用BrowserWindow\.webContents.send 向渲染进程发送消息

```纯文本
const mainWindow = new BrowserWindow();
mainWindow.webContents.send('messageToRenderer', 'Hello from Main!');

```

方式二：ipcMain 模块监听来自渲染进程事件，通过event.sender.send方法向渲染进程发送消息

```纯文本
ipcMain.on('messageFromMain', (event, arg) => {
  event.sender.send('messageToRenderer', 'Hello from Main!');
});
```

### 4、渲染进程 → 渲染进程

方式一：将主进程作为渲染器之间的消息代理

```纯文本
主进程监听渲染进程A消息，将消息发送给另一个渲染进程B
ipcMain.on('win1-msg', (event, arg) => {
    // 这条消息来自 window 1
    console.log("name inside main process is: ", arg); 
    // 发送给 window 2 的消息.
    window2.webContents.send( 'forWin2', arg );
  });
  
渲染进程A发送消息
ipcRenderer.send('win1-msg', 'msg from win1');


在另一个渲染进程B中注册监听事件
ipcRenderer.on('forWin2', function (event, arg){
  console.log(arg);
});
```

方式二：从主进程将一个 MessagePort 传递到两个渲染器。 允许在初始设置后渲染器之间直接进行通信。

[ MessagePort - Web API 接口参考 | MDN Channel Messaging API 的 MessagePort 接口代表 MessageChannel 的两个端口之一，它可以让你从一个端口发送消息，并在消息到达的另一个端口监听它们。 https://developer.mozilla.org/zh-CN/docs/Web/API/MessagePort](https://developer.mozilla.org/zh-CN/docs/Web/API/MessagePort " MessagePort - Web API 接口参考 | MDN Channel Messaging API 的 MessagePort 接口代表 MessageChannel 的两个端口之一，它可以让你从一个端口发送消息，并在消息到达的另一个端口监听它们。 https://developer.mozilla.org/zh-CN/docs/Web/API/MessagePort")

在各自的preload.js文件中注入port

```javascript
const { ipcRenderer } = require('electron')

ipcRenderer.on('port', e => {
  // port received, make it globally available.
  window.electronMessagePort = e.ports[0]

  window.electronMessagePort.onmessage = messageEvent => {
    // handle message
  }
})
```

窗口准备就绪下发port给每个渲染进程

```javascript
const { BrowserWindow, app, MessageChannelMain } = require('electron')

app.whenReady().then(async () => {
  // create the windows.
  const mainWindow = new BrowserWindow({
    show: false,
    webPreferences: {
      contextIsolation: false,
      preload: 'preloadMain.js'
    }
  })

  const secondaryWindow = new BrowserWindow({
    show: false,
    webPreferences: {
      contextIsolation: false,
      preload: 'preloadSecondary.js'
    }
  })

  // set up the channel.
  const { port1, port2 } = new MessageChannelMain()

  // once the webContents are ready, send a port to each webContents with postMessage.
  mainWindow.once('ready-to-show', () => {
    mainWindow.webContents.postMessage('port', null, [port1])
  })

  secondaryWindow.once('ready-to-show', () => {
    secondaryWindow.webContents.postMessage('port', null, [port2])
  })
})
```

渲染进程发送至渲染进程

```javascript
// elsewhere in your code to send a message to the other renderers message handler
window.electronMessagePort.postMessage('ping')
```

如何封装一个通用的进程通信模块？

整体思路

为了在渲染进程和主进程中进行统一的通信调用和简化事件通道管理，需要导出相同的api来使用。采用方案：主进程使用ipcManager.ts来导出on、send、invoke、handle等方法给主进程调用，渲染进程使用ipcService.ts导出类似的api，但是需要通过preload的contextBridge.exposeInMainWorld将api暴露给渲染进程进行调用。

这里需要注意

1、不能在主进程中访问window对象，否则会报错window is not defined。因为window 是渲染进程中定义的。

2、不能在渲染进程中调用主进程的对象，否则会报错 \_\_dirname is not defined
