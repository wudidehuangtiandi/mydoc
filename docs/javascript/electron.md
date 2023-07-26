# electron的应用

## 1.基本构成

> Electron 是一个使用 JavaScript、HTML 和 CSS 构建跨平台的桌面应用程序。它基于 Node.js 和 Chromium，被 Atom 编辑器和许多其他应用程序使用。Electron 兼容 Mac、Windows 和 Linux，可以构建出三个平台的应用程序。

[中文文档地址](https://www.electronjs.org/zh/docs/latest/)

electron需要nodejs环境，它分为主进程和渲染进程，主进程运行在操作系统中，以nodejs为环境，渲染进行负责浏览器相关的工作，Electron 的主进程和渲染进程有着清楚的分工并且不可互换。 这代表着无论是从渲染进程直接访问 Node.js 接口，亦或者是从主进程访问 HTML 文档对象模型 (DOM)，都是不可能的。

解决这一问题的方法是使用进程间通信 (IPC)。可以使用 Electron 的 `ipcMain` 模块和 `ipcRenderer` 模块来进行进程间通信。 为了从你的网页向主进程发送消息，你可以使用 `ipcMain.handle` 设置一个主进程处理程序（handler），然后在预处理脚本中暴露一个被称为 `ipcRenderer.invoke` 的函数来触发该处理程序（handler）。

一般我们的项目中会含有`main.js`其中会维护主进程的操作，包括浏览器的创建，和渲染进程的通信及一些依赖操作系统的逻辑。其中可以创建多个浏览器，及每个浏览器都可以对应控制，这边还会加载`index.html`来实现网页的装载。

一般我们的项目中会含有`preload.js`其中会维护预加载，其中可以处理渲染进程的通信及暴露一些渲染进程需要的主进程函数变量。

最后index.html中导入我们的单页应用，这边就和其它单页前端项目一样了

## 2.调试及页面模拟

利用electron和一些插件配合，不止能做到js开发桌面应用，还能够做到页面模拟鼠标操作，来破解一些需要人工操作的图灵验证

### 2.1chrome-remote-interface

> 这个东西能够打开一个调试端口，默认是9222。它可以做到谷歌浏览器控制台能做到的绝大多数操作，我们主要想利用其获得到页面调用的请求地址及参数。

```javascript
const CDP = require("chrome-remote-interface");
//然后在打开窗口的ipcMain.handle中增加处理,以下代码演示了如何在监听到指定路径时进行隐藏浏览器操作。
   let win = new BrowserWindow({
    width: 1100,
    height: 900,
    webPreferences: {
      nodeIntegration: true,
      contextIsolation: true,
      enableRemoteModule: true,
      devTools: true,
      partition: name,
    },
  });
 CDP().then(async (chrome) => {
    const { Network } = chrome;
    console.log(222);
    await Network.enable();

    // const { targetInfos } = chrome.Target.getTargets()
    //// console.log('1233', targetInfos)

    Network.responseReceived((params) => {
          if (
        params.response.url ==
        "https://mms.pinduoduo.com/janus/api/user/queryUserPermissionByUserId"
      ) {
        
        win.hide();}
    })
 })
```

### 2.2puppeteer-core

> 其主要作用是链接刚刚打开的9222端口并对浏览器进行一切操作，比如UI操作等。需要配合node的node-fetch使用

```javascript
const puppeteer = require("puppeteer-core");
const fetch = require("node-fetch");
//整体页面的模拟方法,在内部区分页面,注意这边不同浏览器使用的是一个端口，使用浏览器ID去区分，这玩意创建的时候会获取到。
async function simulateClick(targetId, snId) {
  // console.log(snId, "sniwd1");
  if (snId) {
    snId = String(snId);
    // console.log(snId, "snid2");

    let res = await fetch(`http://127.0.0.1:9222/json/version`);

    let r = await res.json();

    const browser = await puppeteer.connect({
      browserWSEndpoint: r.webSocketDebuggerUrl,
    });

    let targets = await browser.targets();
    for (const target of targets) {
      if (target.type() === "page") {
        const tid = target._targetId;
        //下面这个要的
        if (tid == targetId) {
          const page = await target.page();
          await page.type(".IPT_input_5-47-0", snId); // 输入文本
        }
      }
    }
  }
}
```

## 3.获取唯一的机器码

> 我们公司做CS的原因就是需要对硬件做出一定的限制，这就需要我们去获取唯一的机器码。

这边我们使用`machine-id2`这个插件来获取,它将获取注册表机器码，一般情况下是唯一的（听说镜像系统有可能重复）

```javascript
const { machineId } = require("machine-id2");
ipcMain.handle("mactionId", () => {
  return machineId();
});
```

## 4.自动更新

> 一般CS架构都需要自动更新，我们给一个简单的自动更新设置

需要用到`electron-updater`

```javascript
const { autoUpdater } = require("electron-updater");
// 部分 API 在 ready 事件触发后才能使用。
app.whenReady().then(() => {
  appIcon = path.join(__dirname, "../src/assets/ybkj-logo.png");
  autoUpdater.setFeedURL("https://自己服务器的代理地址");
  autoUpdater.checkForUpdatesAndNotify();

  autoUpdater.on("update-downloaded", () => {
    autoUpdater.quitAndInstall();
  });
  createWindow();})
```

package.json中需要如下配置

```
"build": {
    "appId": "com.my-website.my-app",
    "productName": "煜博助手",
    "copyright": "Copyright © 2022 yulinZ",
    "publish": [
      {
        "provider": "generic",
        "url": "https://自己服务器的代理地址"
      }
    ]}
```

nginx的server中代理下地址即可

```
   location /autoupdate { 
		root /usr/share/nginx/html/xxx;
   }
```

## 5.另外给一下依赖的配置及贴出我们的部分代码

> package.json

```
{
  "name": "xyzs",
  "private": true,
  "author": "yulinZ",
  "version": "1.2.5",
  "license": "ISC",
  "***": "electron 入口文件",
  "main": "electron/main.js",
  "scripts": {
    "dev": "vite --mode dev",
    "build": "vite build --mode prod",
    "preview": "vite preview --mode prod",
    "electron": "wait-on tcp:3000 && cross-env NODE_ENV=development electron .",
    "electron:serve": "concurrently -k \"pnpm dev\" \"pnpm electron\"",
    "electron:start": "electron .",
    "electron:generate-icons": "electron-icon-builder --input=./src/assets/ybkj-logo.png --output=dist_electron/build --flatten",
    "electron:build-win": "pnpm electron:generate-icons && vite build --mode prod && electron-builder --win --x64",
    "start": "electron-forge start",
    "package": "electron-forge package",
    "make": "electron-forge make"
  },
  "dependencies": {
    "@element-plus/icons-vue": "^2.0.10",
    "@fingerprintjs/fingerprintjs": "^3.4.0",
    "@vitejs/plugin-vue-jsx": "^2.0.1",
    "@vueuse/core": "^9.3.0",
    "@vueuse/shared": "^9.13.0",
    "agent-base": "^6.0.2",
    "any-promise": "^1.3.0",
    "async": "^2.0.0",
    "axios": "^1.1.3",
    "b4a": "^1.6.3",
    "balanced-match": "^2.0.0",
    "bignumber.js": "^9.1.1",
    "bl": "^6.0.1",
    "brace-expansion": "^2.0.1",
    "buffer-crc32": "^0.2.13",
    "builder-util-runtime": "^9.2.0",
    "chownr": "^2.0.0",
    "chrome-remote-interface": "^0.32.1",
    "chromedriver": "^111.0.0",
    "conf": "^11.0.1",
    "debug": "^4.3.4",
    "devtools-protocol": "^0.0.1120367",
    "echarts-wordcloud": "^2.0.0",
    "electron-devtools-installer": "^3.2.0",
    "electron-updater": "^5.3.0",
    "element-plus": "^2.2.18",
    "end-of-stream": "^1.4.4",
    "extract-zip": "^2.0.1",
    "fast-fifo": "^1.1.0",
    "fd-slicer": "^1.1.0",
    "file-saver": "^2.0.5",
    "fs-extra": "^11.1.1",
    "get-stream": "^6.0.1",
    "github-markdown-css": "^5.1.0",
    "glob": "^9.3.2",
    "graceful-fs": "^4.2.11",
    "https-proxy-agent": "^5.0.1",
    "inherits": "^2.0.4",
    "ityped": "^1.0.3",
    "js-file-download": "^0.4.12",
    "js-yaml": "^4.1.0",
    "jsonfile": "^6.1.0",
    "jszip": "^3.10.1",
    "lazy-val": "^1.0.5",
    "lockfile": "^1.0.4",
    "lodash": "^4.17.21",
    "lodash.escaperegexp": "^4.1.2",
    "lodash.isequal": "^4.5.0",
    "lru-cache": "^8.0.4",
    "machine-id2": "^1.0.3",
    "md-editor-v3": "^2.3.0",
    "minimatch": "^7.4.3",
    "minipass": "^4.2.5",
    "mkdirp": "^2.1.6",
    "mkdirp-classic": "^0.5.3",
    "ms": "^2.1.3",
    "native-reg": "^1.1.1",
    "node-fetch": "2.6.2",
    "node-gyp-build": "^4.6.0",
    "node-machine-id": "^1.1.12",
    "once": "^1.4.0",
    "pako": "^2.1.0",
    "path-scurry": "^1.6.3",
    "pend": "^1.2.0",
    "pinia": "^2.0.23",
    "pinia-plugin-persist": "^1.0.0",
    "proxy-from-env": "^1.1.0",
    "pump": "^3.0.0",
    "puppeteer-core": "^19.8.0",
    "qs": "^6.11.0",
    "queue-tick": "^1.0.1",
    "readable-stream": "^3.6.2",
    "rimraf": "^4.4.1",
    "sax": "^1.2.4",
    "selenium-webdriver": "^4.8.1",
    "semver": "^7.3.8",
    "setimmediate": "^1.0.5",
    "streamx": "^2.13.2",
    "tar-fs": "^2.1.1",
    "tar-stream": "^3.0.0",
    "through": "^2.3.8",
    "ts-md5": "^1.3.1",
    "unbzip2-stream": "^1.4.3",
    "universalify": "^2.0.0",
    "unzip-crx-3": "^0.2.0",
    "util-deprecate": "^1.0.2",
    "vue": "^3.2.37",
    "vue-demi": "^0.13.11",
    "vue-i18n": "^9.2.2",
    "vue-router": "^4.0.14",
    "wait-on": "^6.0.1",
    "webdriverio": "^8.6.7",
    "wrappy": "^1.0.2",
    "write-file-atomic": "^2.4.2",
    "ws": "^8.13.0",
    "xlsx": "^0.18.5",
    "yaku": "^1.0.1",
    "yauzl": "^2.10.0"
  },
  "devDependencies": {
    "@electron-forge/cli": "^6.0.5",
    "@iconify-json/ant-design": "^1.1.3",
    "@iconify-json/ep": "^1.1.8",
    "@iconify-json/logos": "^1.1.16",
    "@iconify-json/mdi": "^1.1.34",
    "@intlify/vite-plugin-vue-i18n": "^6.0.3",
    "@vitejs/plugin-vue": "^3.1.0",
    "autoprefixer": "^10.4.12",
    "concurrently": "^6.3.0",
    "cross-env": "^7.0.3",
    "echarts": "^5.4.0",
    "electron": "^21.1.1",
    "electron-builder": "^23.6.0",
    "electron-icon-builder": "^2.0.1",
    "happy-dom": "^7.5.12",
    "nprogress": "^0.2.0",
    "postcss": "^8.4.18",
    "sass": "^1.55.0",
    "tailwindcss": "^3.1.8",
    "typescript": "^4.6.4",
    "unplugin-auto-import": "^0.11.2",
    "unplugin-icons": "^0.14.12",
    "unplugin-vue-components": "^0.22.8",
    "vite": "^3.1.0",
    "vite-plugin-html": "^3.2.0",
    "vite-plugin-vue-markdown": "^0.22.1",
    "vue-tsc": "^0.40.4"
  },
  "build": {
    "appId": "com.my-website.my-app",
    "productName": "xxxx",
    "copyright": "Copyright © 2022 yulinZ",
    "publish": [
      {
        "provider": "generic",
        "url": "https:xxxxxxxxxxxxxxxxxxxxxxxxxxx"
      }
    ],
    "mac": {
      "category": "public.app-category.utilities",
      "icon": "dist_electron/build/icons/icon.ico"
    },
    "nsis": {
      "oneClick": false,
      "language": "2052",
      "perMachine": true,
      "allowToChangeInstallationDirectory": true
    },
    "win": {
      "icon": "dist_electron/build/icons/icon.ico",
      "target": [
        "nsis",
        "msi"
      ]
    },
    "files": [
      "dist/**/*",
      "electron/**/*"
    ],
    "directories": {
      "buildResources": "assets",
      "output": "dist_electron"
    }
  }
}
```

> main.js

```
/*
 * @Author: yulinZ 1973329248@qq.com
 * @Date: 2022-09-22 03:32:55
 * @LastEditors: yulinZ 1973329248@qq.com
 * @LastEditTime: 2022-10-18 13:59:14
 * @FilePath: \vue3vite\electron\main.js
 * @Description:
 *
 * Copyright (c) 2022 by yulinZ 1973329248@qq.com, All Rights Reserved.
 */
// 控制应用生命周期和创建原生浏览器窗口的模组
// const { env } = require('../vite/shared/env')

const { app, BrowserWindow, Menu, ipcMain, session } = require("electron");
const fs = require("fs");

const CDP = require("chrome-remote-interface");
const {
  default: installExtension,
  REACT_DEVELOPER_TOOLS,
} = require("electron-devtools-installer");
// process.env.NODE_TLS_REJECT_UNAUTHORIZED = 0
const NODE_ENV = process.env.NODE_ENV;
process.env["ELECTRON_DISABLE_SECURITY_WARNINGS"] = "true";

const puppeteer = require("puppeteer-core");
const fetch = require("node-fetch");

const path = require("path");
const nodeos = require("os");
const internal = require("stream");
const { machineId } = require("machine-id2");
const { autoUpdater } = require("electron-updater");
const os = require("os");

//记住密码
// const Store = require("electron-store");
// const store = new Store();

// //把从渲染进程拿到的码放起来
const FilePath = path.join(os.homedir(), "password.txt");

ipcMain.handle("remember-passward", (event, passward) => {
  fs.writeFile(FilePath, passward, function (err) {
    if (err) {
      console.error(`Error writing file: ${err}`);
      return;
    }
    console.log(passward, "passward");
  });
});

let isSeam = 2;

//把相同的登陆的id给他关掉get-seam-target-id
ipcMain.handle("get-seam-target-id", (event, id, flag) => {
  console.log(id, "id", flag);
  isSeam = flag;
  // if (flag == 1) {
  //   windows[id].close();
  // }
});

ipcMain.handle("get-passward", () => {
  fs.access(FilePath, fs.constants.F_OK, (err) => {
    if (err) {
      console.error(`Error accessing file: ${err}`);
      mainWindow.webContents.send("getPassward", 1);
      return;
    }
    fs.readFile(FilePath, "utf8", function (err, data) {
      if (err) {
        console.error(`Error reading file: ${err}`);
        mainWindow.webContents.send("getPassward", 1);
        return;
      }
      console.log(data.toString(), "data");
      mainWindow.webContents.send("getPassward", data.toString());
    });
  });
});
app.commandLine.appendSwitch("remote-debugging-port", "9222");

//全局方法
ipcMain.handle("os", () => {
  return nodeos.networkInterfaces();
});

ipcMain.handle("osCpu", () => {
  return nodeos.cpus();
});

ipcMain.handle("hostname", () => {
  return nodeos.hostname();
});
ipcMain.handle("mactionId", () => {
  return machineId();
});

// let macId = await window.versions.macId();
// console.log(macId, "macId");

//整体页面的模拟方法,在内部区分页面
async function simulateClick(targetId, snId) {
  // console.log(snId, "sniwd1");
  if (snId) {
    snId = String(snId);
    // console.log(snId, "snid2");

    let res = await fetch(`http://127.0.0.1:9222/json/version`);

    let r = await res.json();

    const browser = await puppeteer.connect({
      browserWSEndpoint: r.webSocketDebuggerUrl,
    });

    let targets = await browser.targets();
    for (const target of targets) {
      if (target.type() === "page") {
        const tid = target._targetId;
        //下面这个要的
        if (tid == targetId) {
          const page = await target.page();
          await page.type(".IPT_input_5-47-0", snId); // 输入文本
        }
      }
    }
  }
}

//清空搜索框
async function clearSearchInput(targetId) {
  await windows[targetId].webContents.executeJavaScript(`
  document.getElementsByClassName("BTN_outerWrapper_5-47-0 BTN_gray_5-47-0 BTN_medium_5-47-0 BTN_outerWrapperBtn_5-47-0")[0].click()
    `);
}

//暂停,会停止所有的遍历
ipcMain.handle("stop-init", () => {
  //isInit = false
  let objvlaue = Object.values(windows);
  objvlaue.forEach((e) => {
    e.webContents.executeJavaScript(`
    clearTimeout(bl_timer)
    `);
    //停止遍历之后向渲染页面发送信息
  });
  mainWindow.webContents.send("stop-success", "停止成功");
});

//所有窗口的监听
//以下为一个浏览器周期内的方法
//点击开始遍历，跳转页面，获取身份信息
ipcMain.handle("start-traverseData", (event, flag, shopData) => {
  let shopDataS = JSON.parse(shopData);
  //获取所有的浏览器
  shopDataS.forEach((item) => {
    windows[item.webTargetId].loadURL(
      "https://mms.pinduoduo.com/goods/goods_list"
    );

    //   windows[item.webTargetId].webContents.on('did-finish-load', () => {
    setTimeout(() => {
      windows[item.webTargetId].webContents.executeJavaScript(
        `
    if(typeof bl_timer === 'undefined'){
    let bl_timer=null;
    }else{
      clearTimeout(bl_timer)
    }
    bl_timer=setInterval(function(){
      if(document.getElementsByClassName("PGT_next_5-47-0").length>0){
        document.getElementsByClassName("PGT_next_5-47-0")[0].click()
      }else{
        clearTimeout(bl_timer)
      }
    },2000)
   `
      );
    }, 6000);
    isSeam = 1;
    console.log(isSeam, "isSeam2222");
    //})
  });
});

//关闭窗口
// TODO 需要传入关闭的浏览器
ipcMain.handle("close-new-window", () => {
  let objvlaue = Object.values(windows);
  objvlaue.forEach((e) => {
    e.webContents.reloadIgnoringCache();
    e.hide();
  });
});

//判断有没有验证码

//下架某店铺商品
//TODO 需要传入需要下架的页面
//下架某店铺商品
function setTimers(time) {
  return new Promise((resolve) => setTimeout(resolve, time));
}

ipcMain.handle("get-goods", async (event, data) => {
  //下架反馈 把信息传给

  let goodsItem = JSON.parse(data);

  for (const item of goodsItem) {
    // // loadingInstance.close();
    // console.log(item, "item");
    await simulateClick(item.webTargetId, item.id);
    let res = windows[item.webTargetId].webContents.executeJavaScript(
      `
      function setTimer(time) {
        return new Promise((resolve) => setTimeout(resolve, time));
      }
          isSoldDown(); 
        async function isSoldDown() {
            document.getElementsByClassName("BTN_primary_5-47-0")[0].click()
            await setTimer(1000);
            document.getElementsByClassName("BTN_outerWrapper_5-47-0 BTN_textPrimary_5-47-0 BTN_small_5-47-0 BTN_outerWrapperLink_5-47-0")[4].click()
            await setTimer(1000);
            document.getElementsByClassName("BTN_outerWrapper_5-47-0 MDL_okBtn_5-47-0 BTN_primary_5-47-0 BTN_medium_5-47-0 BTN_outerWrapperBtn_5-47-0")[0].click()
            await setTimer(1000);
            return true
          }
     `
    );

    if (res) {
      await setTimers(4000);
      //用户信息和 浏览器id一起传回去
      await clearSearchInput(item.webTargetId);
      item.flag = true;
      // item["webTargetId"] = item.webTargetId;
      mainWindow.webContents.send("sold-dowm", JSON.stringify(item));
      // }, 7000);
    } else {
      await setTimers(4000);
      await clearSearchInput(item.webTargetId);
      windows[item.webTargetId].webContents.executeJavaScript(`
            document.getElementsByClassName("BTN_primary_5-47-0")[0].click()
          },1000)
            `);
      // setTimeout(() => {
      item.flag = false;
      mainWindow.webContents.send("sold-dowm", JSON.stringify(item));
      // }, 7000);
    }
  }
});

//所有窗口
let windows = {};
//创建窗口及其对应局域方法，渲染线程需要记录Num和对应用户信息的绑定关系，以便在渲染线程遍历用户的时候操作对应的窗口
ipcMain.handle("open-new-window", (event, url, num) => {
  const name = "window" + num;
  let win = new BrowserWindow({
    width: 1100,
    height: 900,
    webPreferences: {
      nodeIntegration: true,
      contextIsolation: true,
      enableRemoteModule: true,
      devTools: true,
      partition: name,
    },
  });
  let wc = win.webContents;
  wc.debugger.attach("1.3");

  //浏览器id
  let webTargetId = "";
  wc.debugger.sendCommand("Target.getTargetInfo").then((r) => {
    //获取浏览器的targetId
    webTargetId = r.targetInfo.targetId;
    // console.log(webTargetId, "webTargetId");

    win.loadURL(url);
    windows[webTargetId] = win;
    // 获取页面的document对象
    windows[webTargetId].webContents.on("dom-ready", () => {
      // 在DOM加载完毕后添加元素
      windows[webTargetId].webContents
        .executeJavaScript(
          `
          var oBtn = document.createElement("input");
          oBtn.id = "btn";
          oBtn.type = "button";
          oBtn.value = "扫码成功后点我"
          oBtn.style.position = "absolute";
          oBtn.style.top = "10px";
          oBtn.style.left = "10px";
          oBtn.style.width = "150px";
          oBtn.style.height = "50px";
          oBtn.style.fontSize = "20px";
          oBtn.style.zIndex = "9999999";
          document.body.appendChild(oBtn);
          var promise = new Promise(function(resolve, reject) {
            oBtn.addEventListener('click', () => {
              var new_userinfo = window.localStorage.getItem('new_userinfo');
              if(!new_userinfo) {
                alert('请先进行登陆操作')
              }
              resolve(new_userinfo);
            })
          });
          promise;
          `
        )
        .then((result) => {
          console.log(result, "res000");
          if (!result) {
            return;
          }
          windows[webTargetId].hide();
          if (isSeam != 2) {
            return;
          }
          if (isSeam == 2) {
            //判断是否登陆 然后获取用户信息
            //定义一个是否获取到用户信息的 便于渲染端进行loading
            mainWindow.webContents.send("is-get-shopInfo");
            win.webContents
              .executeJavaScript(
                `
        window.localStorage.getItem('new_userinfo')
      `
              )
              .then((result) => {
                if (result) {
                  //判断登录的店铺是否相同
                  let resultMall = JSON.parse(result);
                  if (resultMall.mall) {
                    resultMall.mall["webTargetId"] = webTargetId; //浏览器id
                  }
                  mainWindow.webContents.send(
                    "update-account",
                    JSON.stringify(resultMall)
                  ); //用户信息和 浏览器id一起传回去
                }
              })
              .catch(() => {
                win.webContents.reloadIgnoringCache();
              });
          }
        });
      isSeam = 2;
    });
  });

  CDP().then(async (chrome) => {
    const { Network } = chrome;
    console.log(222);
    await Network.enable();

    // const { targetInfos } = chrome.Target.getTargets()
    //// console.log('1233', targetInfos)

    Network.responseReceived((params) => {
      if (
        params.response.url ==
        "https://mms.pinduoduo.com/janus/api/user/queryUserPermissionByUserId"
      ) {
        //判断是否登陆 然后获取用户信息
        win.hide();
        if (isSeam != 2) {
          return;
        }
        if (isSeam == 2) {
          //判断是否登陆 然后获取用户信息
          //定义一个是否获取到用户信息的 便于渲染端进行loading
          mainWindow.webContents.send("is-get-shopInfo");
          win.webContents
            .executeJavaScript(
              `
  window.localStorage.getItem('new_userinfo')
`
            )
            .then((result) => {
              console.log(result, "result");
              if (result) {
                //判断登录的店铺是否相同
                let resultMall = JSON.parse(result);
                if (resultMall.mall) {
                  resultMall.mall["webTargetId"] = webTargetId; //浏览器id
                }
                mainWindow.webContents.send(
                  "update-account",
                  JSON.stringify(resultMall)
                ); //用户信息和 浏览器id一起传回去
              }
            })
            .catch(() => {
              win.webContents.reloadIgnoringCache();
            });
        }
      }

      //判断窗口有没有跳验证码 https://apiv2.pinduoduo.net/api/phantom/obtain_captcha

      if (
        params.response.url ==
        "https://apiv2.pinduoduo.net/api/phantom/obtain_captcha"
      ) {
        //打开窗
        windows[webTargetId].show();
        //向渲染进程发送信息 表示需要进行验证码
        mainWindow.webContents.send("send-verification", "请先进行验证码验证");
      }

      //获取商品列表
      if (
        params.response.url ===
        "https://mms.pinduoduo.com/vodka/v2/mms/query/display/mall/goodsList"
      ) {
        //// console.log(document.getElementsByClassName("IPT_input_5-47-0")[0]);
        //// console.log("params", params.response.url);
        Network.getResponseBody({ requestId: params.requestId })
          .then((response) => {
            // 商品列表直接主窗口处理
            //把浏览器id带上
            let responses = JSON.parse(response.body);
            responses["webTargetId"] = webTargetId;
            mainWindow.webContents.send(
              "update-counter",
              JSON.stringify(responses)
            );
          })
          .catch((error) => {
            console.error(error, "ERROR");
          });
      }

      //获取token https://mms.pinduoduo.com/janus/api/subSystem/getAuthToken
      // if (params.response.url === 'https://mms.pinduoduo.com/janus/api/subSystem/getAuthToken') {
      //   Network.getResponseBody({ requestId: params.requestId })
      //     .then(response => {
      //       //// console.log("response", response.body);
      //       win.webContents.send('update-token', response.body)
      //     })
      //     .catch(error => {
      //       console.error(error)
      //     })
      // }
    });
  });
});
let mainWindow = null;
function createWindow() {
  Menu.setApplicationMenu(null);
  // 创建浏览器窗口
  mainWindow = new BrowserWindow({
    icon: appIcon,
    width: 1100,
    height: 800,
    minWidth: 1070,
    resizable: true,
    webPreferences: {
      preload: path.join(__dirname, "preload.js"),
      nodeIntegration: true,
      contextIsolation: true,
      enableRemoteModule: true,
    },
  });
  // mainWindow.webContents.openDevTools();
  // 加载 index.html
  //// console.log( __static,'999999')
  mainWindow.loadURL(
    NODE_ENV === "development"
      ? "http://localhost:3000"
      : `file://${path.join(__dirname, "../dist/index.html")}`
  );
  // mainWindow.loadURL(`http://localhost:${config.VITE_PORT}`)
  // mainWindow.loadFile('dist/index.html')
  // 此处跟electron官网路径不同，需要注意

  // 打开开发工具
  // mainWindow.webContents.openDevTools();
  NODE_ENV === "development" && mainWindow.webContents.openDevTools();
}

//主窗口的创建
let appIcon = "";
// 部分 API 在 ready 事件触发后才能使用。
app.whenReady().then(() => {
  appIcon = path.join(__dirname, "../src/assets/ybkj-logo.png");
  autoUpdater.setFeedURL("https://xxxxxxxxxxxxxxxxxxxxxxxx");
  autoUpdater.checkForUpdatesAndNotify();

  autoUpdater.on("update-downloaded", () => {
    autoUpdater.quitAndInstall();
  });
  createWindow();

  if (process.platform === "darwin") {
    app.dock.setIcon(appIcon);
  }
  app.on("activate", () => {
    // 通常在 macOS 上，当点击 dock 中的应用程序图标时，如果没有其他
    // 打开的窗口，那么程序会重新创建一个窗口。
    if (BrowserWindow.getAllWindows().length === 0) {
      createWindow();
    }
  });
});

// 这段程序将会在 Electron 结束初始化
// 和创建浏览器窗口的时候调用

// 除了 macOS 外，当所有窗口都被关闭的时候退出程序。 因此，通常对程序和它们在
// 任务栏上的图标来说，应当保持活跃状态，直到用户使用 Cmd + Q 退出。
app.on("window-all-closed", () => {
  if (process.platform !== "darwin") {
    app.quit();
  }
});
```

> preload.js

```
// 所有 Node.js API 都可以在预加载过程中使用。
// 它拥有与 Chrome 扩展一样的沙盒。
window.addEventListener("DOMContentLoaded", () => {
  const replaceText = (selector, text) => {
    const element = document.getElementById(selector);
    if (element) element.innerText = text;
  };

  for (const type of ["chrome", "node", "electron"]) {
    replaceText(`${type}-version`, process.versions[type]);
  }
});

const { contextBridge, ipcRenderer } = require("electron");

contextBridge.exposeInMainWorld("versions", {
  node: () => process.versions.node,
  os: () => ipcRenderer.invoke("os"),
  hostname: () => ipcRenderer.invoke("hostname"),
  osCpu: () => ipcRenderer.invoke("osCpu"),
  macId: () => ipcRenderer.invoke("mactionId"),
  // 能暴露的不仅仅是函数，我们还可以暴露变量
});

contextBridge.exposeInMainWorld("myAPI", {
  openNewWindow: (url, num) => ipcRenderer.invoke("open-new-window", url, num),
  sendSeamTargetId: (id, flag) =>
    ipcRenderer.invoke("get-seam-target-id", id, flag),
  savePassward: (passward) => ipcRenderer.invoke("remember-passward", passward),
  getPassward: () => ipcRenderer.invoke("get-passward"),
  getStorePassward: (callback) => ipcRenderer.on("getPassward", callback),
  closeNewWindow: () => ipcRenderer.invoke("close-new-window"),
  clickBtnWindow: (flag, shopData) =>
    ipcRenderer.invoke("start-traverseData", flag, shopData),
  stopInit: () => ipcRenderer.invoke("stop-init"),
  stopSuccess: (callback) => ipcRenderer.on("stop-success", callback),
  handleCounter: (callback) => ipcRenderer.on("update-counter", callback),
  isGetShopInfo: (callback) => ipcRenderer.on("is-get-shopInfo", callback),
  handleAccount: (callback, num) =>
    ipcRenderer.on("update-account", callback, num),
  handleToken: (callback) => ipcRenderer.on("update-token", callback),
  handleTraver: (callback) => ipcRenderer.on("goods-traverse", callback),
  handleSoldGoods: (callback) => ipcRenderer.on("sold-dowm", callback),
  handleGetGoods: (info) => ipcRenderer.invoke("get-goods", info),
  loginSeamShop: (callback) => ipcRenderer.on("login-seam-shop", callback),
  handleVerification: (info) => ipcRenderer.on("send-verification", info),
});

```

> login.vue

```
<!--
 * @Author: yulinZ 1973329248@qq.com
 * @Date: 2022-09-11 17:47:25
 * @LastEditors: yulinZ 1973329248@qq.com
 * @LastEditTime: 2022-10-20 18:03:35
 * @FilePath: \vue3vite\src\pages\login\components\LoginForm.vue
 * @Description: 
 * 
 * Copyright (c) 2022 by yulinZ 1973329248@qq.com, All Rights Reserved. 
-->
<template>
  <el-form
    ref="loginForm"
    :model="loginUser"
    :rules="rules"
    class="loginForm w-full sign-in-form"
  >
    <el-form-item prop="key">

      <el-input
        class="from-input"
        v-model="loginUser.key"
        placeholder="请输入密钥"
      >
        <template #prefix>
          <el-icon class="el-input__icon">
            <Lock />
          </el-icon> </template></el-input>

    </el-form-item>
    <!-- <el-form-item prop="password">
      <el-input class="from-input" v-model="loginUser.password" type="password" placeholder="密码...">
        <template #prefix>
          <el-icon class="el-input__icon">
            
          </el-icon>
        </template>
      </el-input>
    </el-form-item> -->
    <!-- <el-form-item prop="code">
      <el-row>
        <el-col :span="12">
          <el-input v-model="loginUser.imgCode" placeholder="请输入验证码">
            <template #prefix>
              <el-icon class="el-input__icon">
                <IEpUnlock />
              </el-icon>
            </template>
          </el-input>
        </el-col>
        <el-col :span="12">
          <div class="ml-2 mr-2">
            <image-verify ref="verifyRef" />
          </div>
        </el-col>
      </el-row>
    </el-form-item> -->
    <el-form-item>
      <el-button
        :loading="loading"
        @click="handleLogin(loginForm)"
        type="primary"
        class="submit-btn"
      >登录</el-button>
      <el-button
        type="success"
        style="width: 100%;margin-left: 0px; margin-top: 10px;"
        @click="changeBd"
      >
        换绑
      </el-button>
      <el-checkbox
        v-model="rememberPassward"
        label="记住密钥"
        size="large"
        class="remember-pass"
        style="margin-right: 8px;"
      />

    </el-form-item>
    <!-- 找回密码 -->
    <!-- <div class="tiparea">
      <p>
        <span class="text-white dark:text-black">
          忘记密码？ <a>立即找回</a></span
        >
      </p>
    </div> -->
  </el-form>
</template>

<script setup lang="ts">
import { ref } from "vue";
import { debounce } from "@/utils/debounce";

import _ from "lodash";
import { FormInstance, FormRules, ElMessage } from "element-plus";
import { queryChangeBd } from "@/api/api";
import { useUserStore } from "@/stores/useUser";
import pinia from "@/stores/store";
const userStore = useUserStore(pinia);
const dialogTableVisible = ref(false); //弹窗展示
const dateLimit = ref("");
const loading = ref(false);

const router = useRouter();
const route = useRoute();
const rememberPassward = ref(false);
const verifyRef = ref(null);
const loginForm = ref(null);
const props = defineProps({
  loginUser: {
    type: Object,
    required: true,
  },
  rules: {
    type: Object,
    required: true,
  },
});

//节流
const handleLogin = debounce(() => {
  toLogin();
}, 300);

getPassward();
//拿到存储的密码
function getPassward() {
  // let passward = localStorage.getItem('rememberPassward')
  // props.loginUser.key = passward;
  // rememberPassward.value = true;
  window.myAPI.getPassward();
  getStorePassward();
}

function getStorePassward() {
  window.myAPI.getStorePassward((event, value) => {
    if (value !== 1) {
      props.loginUser.key = value;
      rememberPassward.value = true;
    } else { 
      props.loginUser.key = '';
      rememberPassward.value = false;

    }
  });
}

//换绑定
function changeBd() {

  if (!props.loginUser.key) {
    ElMessage({
      message: "请先输入密钥",
      type: "warning",
    });
    return;
  }
  ElMessageBox.confirm(
    "每个密钥仅可匹配一台机器，并且拥有一次换绑机会，换绑的机器必须从未使用任何密钥进行登陆。换绑之后，原机器将无法使用该密钥进行登陆，同一台机器无法使用多个密钥",
    "提示",
    {
      confirmButtonText: "继续换绑",
      cancelButtonText: "取消",
      type: "warning",
    }
  ).then(() => {
    queryChangeBd({
      key: props.loginUser.key,
      machineCode: localStorage.getItem("machineCode"),
    }).then((res) => {
      if (res.code == 200) {
        ElMessage({
          message: "换绑成功，请重新登陆",
          type: "success",
        });
      }
    });
  });
  console.log(111);
}

function toLogin() {
  if (!props.loginUser.key) {
    ElMessage({
      message: "请先输入密钥",
      type: "warning",
    });
    return;
  }
  userStore.login(props.loginUser).then(() => {
    console.log(rememberPassward.value, "rememberPassward");
    
    if (rememberPassward.value) {
      window.myAPI.savePassward(props.loginUser.key);
      // localStorage.setItem("rememberPassward", props.loginUser.key);
    } else { 
      // localStorage.setItem("rememberPassward",'');

    }
    dialogTableVisible.value = true;
    loading.value = true;
    dateLimit.value = localStorage.getItem("dateLimit");
    router.push({ path: route.query.redirect || "/" });
    setTimeout(() => {
      // ElMessage({
      //   message: "登陆成功！",
      //   type: "success",
      //   duration: 5000,
      // });
      ElNotification({
        type: "warning",
        message: "账号到期时间：" + localStorage.getItem("dateLimit"),
        duration: 6000,
        offset: 100,
      });
    }, 800);
  });
}
</script>
<style scoped>
.loginForm {
  padding: 2rem;
  margin-top: 1.25rem;
}

.from-input {
  width: 100%;
}

.submit-btn {
  width: 100%;
}

.tiparea {
  text-align: right;
  font-size: 12px;
  color: #333;
}
.remember-pass {
  text-align: center;
  margin: 0 auto;
}

.tiparea p a {
  color: var(--main-color);
}
::v-deep .el-checkbox__label {
  color: #fff !important;
}
</style>
```

然后贴下我们的项目使用的框架

[地址](https://github.com/xxxxxii/-vue3-vite-element-plus-electron-tailwindcss)

大家可以去星星下
