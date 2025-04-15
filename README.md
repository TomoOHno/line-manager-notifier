# line-manager-notifier
LINE公式アカウント管理画面の未読通知監視アプリ
// ここではElectronを使ったLINE公式アカウント管理画面の未読通知監視アプリの
// 基本的なコード構成を提供します。以下のファイル構成になります。

// package.json - プロジェクト設定ファイル
{
  "name": "line-manager-notifier",
  "version": "1.0.0",
  "description": "LINE公式アカウント管理画面の未読通知監視アプリ",
  "main": "main.js",
  "scripts": {
    "start": "electron .",
    "build": "electron-builder",
    "pack": "electron-builder --dir"
  },
  "author": "",
  "license": "MIT",
  "devDependencies": {
    "electron": "^22.0.0",
    "electron-builder": "^23.6.0"
  },
  "dependencies": {
    "electron-store": "^8.1.0",
    "node-notifier": "^10.0.1"
  },
  "build": {
    "appId": "com.example.linemanagernotifier",
    "productName": "LINE Manager Notifier",
    "mac": {
      "category": "public.app-category.utilities"
    },
    "win": {
      "target": [
        "nsis"
      ]
    },
    "linux": {
      "target": [
        "AppImage"
      ]
    }
  }
}

// main.js - メインプロセスファイル
const { app, BrowserWindow, Tray, Menu, ipcMain, dialog, shell, nativeImage } = require('electron');
const path = require('path');
const Store = require('electron-store');
const notifier = require('node-notifier');
const fs = require('fs');

// 設定を保存するためのストアを作成
const store = new Store();

// グローバル変数
let mainWindow;
let tray;
let isQuitting = false;
let checkInterval;
let notificationInterval;
let lastNotificationTime = 0;

// デフォルト設定
const defaultSettings = {
  checkIntervalSeconds: 10,    // 未読チェック間隔（秒）
  notifyIntervalSeconds: 60,   // 通知間隔（秒）
  startWithWindows: true,      // Windows起動時に起動
  startMinimized: true,        // 最小化状態で起動
  customSoundEnabled: false,   // カスタム通知音の有効化
  customSoundPath: '',         // カスタム通知音のパス
  autoLogin: false,            // 自動ログイン
  lineUsername: '',            // LINEユーザー名（暗号化が必要）
  linePassword: ''             // LINEパスワード（暗号化が必要）
};

// 設定値の初期化
if (!store.has('settings')) {
  store.set('settings', defaultSettings);
}

function createWindow() {
  // ブラウザウィンドウを作成
  mainWindow = new BrowserWindow({
    width: 1280,
    height: 800,
    icon: path.join(__dirname, 'assets/icons/icon.png'),
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload.js')
    }
  });

  // LINE管理画面を読み込み
  mainWindow.loadURL('https://manager.line.biz/');

  // 開発者ツールを開く (開発時のみ)
  // mainWindow.webContents.openDevTools();

  // ウィンドウが閉じられたときの処理
  mainWindow.on('close', (event) => {
    if (!isQuitting) {
      event.preventDefault();
      mainWindow.hide();
      return false;
    }
    return true;
  });

  // 未読通知監視スクリプトを注入
  mainWindow.webContents.on('did-finish-load', () => {
    injectUnreadDetectionScript();
  });

  // システムトレイアイコンを作成
  setupTray();

  // 設定に応じてウィンドウを最小化
  const settings = store.get('settings');
  if (settings.startMinimized) {
    mainWindow.minimize();
  }

  // 通知監視を開始
  startMonitoring();
}

// 監視を開始
function startMonitoring() {
  const settings = store.get('settings');
  
  // 既存のインターバルをクリア
  if (checkInterval) clearInterval(checkInterval);
  
  // チェック間隔を設定
  checkInterval = setInterval(() => {
    if (mainWindow && !mainWindow.isDestroyed()) {
      injectUnreadDetectionScript();
    }
  }, settings.checkIntervalSeconds * 1000);
}

// 未読検出スクリプトを注入
function injectUnreadDetectionScript() {
  if (!mainWindow || mainWindow.isDestroyed()) return;

  mainWindow.webContents.executeJavaScript(`
    (() => {
      // 未読通知を検出
      const unreadBadges = document.querySelectorAll('.badge, .nav-link span:not(:empty), [class*="badge"], [class*="unread"], [class*="notification"]');
      let hasUnread = false;
      let unreadCount = 0;
      let unreadText = '';
      
      unreadBadges.forEach(badge => {
        // 要素が表示されていて内容がある場合
        if (badge.offsetParent !== null && badge.textContent.trim() !== '') {
          hasUnread = true;
          
          // 数字の場合はカウントを増やす
          const badgeText = badge.textContent.trim();
          if (/^[0-9]+$/.test(badgeText)) {
            unreadCount += parseInt(badgeText, 10);
          }
          
          // テキスト内容を保存
          unreadText = badgeText;
        }
      });
      
      return { 
        hasUnread, 
        unreadCount,
        unreadText
      };
    })();
  `)
  .then(result => {
    handleUnreadDetection(result);
  })
  .catch(err => {
    console.error('未読検出スクリプトの実行中にエラーが発生しました:', err);
  });
}

// 未読検出の結果を処理
function handleUnreadDetection(result) {
  if (result.hasUnread) {
    const now = Date.now();
    const settings = store.get('settings');
    
    // 前回の通知から指定時間が経過している場合のみ通知
    if (now - lastNotificationTime >= settings.notifyIntervalSeconds * 1000) {
      showNotification(result.unreadCount);
      lastNotificationTime = now;
    }
  }
}

// 通知を表示
function showNotification(count) {
  const settings = store.get('settings');
  const title = 'LINE公式アカウント';
  const message = count > 0 
    ? `${count}件の未読メッセージがあります` 
    : '未読メッセージがあります';
  
  // システム通知を表示
  notifier.notify({
    title: title,
    message: message,
    icon: path.join(__dirname, 'assets/icons/icon.png'),
    sound: settings.customSoundEnabled && settings.customSoundPath 
      ? settings.customSoundPath 
      : true,
    wait: true
  });
  
  // 通知クリック時の処理
  notifier.on('click', () => {
    if (mainWindow) {
      if (mainWindow.isMinimized()) mainWindow.restore();
      mainWindow.show();
      mainWindow.focus();
    }
  });
}

// システムトレイの設定
function setupTray() {
  const iconPath = path.join(__dirname, 'assets/icons/tray-icon.png');
  tray = new Tray(nativeImage.createFromPath(iconPath));
  
  const contextMenu = Menu.buildFromTemplate([
    { 
      label: '表示', 
      click: () => {
        if (mainWindow) {
          if (mainWindow.isMinimized()) mainWindow.restore();
          mainWindow.show();
          mainWindow.focus();
        }
      } 
    },
    { 
      label: '設定', 
      click: () => {
        openSettingsWindow();
      } 
    },
    { type: 'separator' },
    { 
      label: '再読み込み', 
      click: () => {
        if (mainWindow) {
          mainWindow.reload();
        }
      } 
    },
    { type: 'separator' },
    { 
      label: '終了', 
      click: () => {
        isQuitting = true;
        app.quit();
      } 
    }
  ]);
  
  tray.setToolTip('LINE Manager Notifier');
  tray.setContextMenu(contextMenu);
  
  tray.on('click', () => {
    if (mainWindow) {
      if (mainWindow.isVisible()) {
        mainWindow.hide();
      } else {
        mainWindow.show();
        mainWindow.focus();
      }
    }
  });
}

// 設定ウィンドウを開く
function openSettingsWindow() {
  const settingsWindow = new BrowserWindow({
    width: 500,
    height: 600,
    parent: mainWindow,
    modal: true,
    icon: path.join(__dirname, 'assets/icons/icon.png'),
    webPreferences: {
      nodeIntegration: false,
      contextIsolation: true,
      preload: path.join(__dirname, 'preload-settings.js')
    }
  });
  
  settingsWindow.loadFile('settings.html');
  
  // 開発者ツールを開く (開発時のみ)
  // settingsWindow.webContents.openDevTools();
}

// アプリケーションの初期化完了時
app.whenReady().then(() => {
  createWindow();
  
  app.on('activate', function () {
    if (BrowserWindow.getAllWindows().length === 0) createWindow();
  });
});

// 全てのウィンドウが閉じられたときの処理
app.on('window-all-closed', function () {
  if (process.platform !== 'darwin') app.quit();
});

// 終了前の処理
app.on('before-quit', () => {
  isQuitting = true;
});

// IPCメッセージの受信（設定の保存）
ipcMain.on('save-settings', (event, newSettings) => {
  store.set('settings', newSettings);
  
  // 監視間隔を更新
  startMonitoring();
  
  // Windowsスタートアップ設定を更新
  updateStartupSetting(newSettings.startWithWindows);
  
  dialog.showMessageBox({
    type: 'info',
    title: '設定保存',
    message: '設定が保存されました',
    buttons: ['OK']
  });
});

// Windowsスタートアップ設定を更新
function updateStartupSetting(enableStartup) {
  if (process.platform === 'win32') {
    app.setLoginItemSettings({
      openAtLogin: enableStartup,
      path: process.execPath
    });
  }
}

// preload.js - メインウィンドウのプリロードスクリプト
const { contextBridge, ipcRenderer } = require('electron');

// 安全な通信チャネルを確立
contextBridge.exposeInMainWorld('electronAPI', {
  // 必要に応じてAPIを追加
  notify: (message) => {
    ipcRenderer.send('notify', message);
  }
});

// preload-settings.js - 設定ウィンドウのプリロードスクリプト
const { contextBridge, ipcRenderer } = require('electron');
const Store = require('electron-store');
const store = new Store();

// 安全な通信チャネルを確立
contextBridge.exposeInMainWorld('electronAPI', {
  // 設定を取得
  getSettings: () => {
    return store.get('settings');
  },
  
  // 設定を保存
  saveSettings: (settings) => {
    ipcRenderer.send('save-settings', settings);
  },
  
  // ファイル選択ダイアログを開く
  selectFile: async () => {
    return await ipcRenderer.invoke('select-file');
  }
});

// settings.html - 設定画面のHTML
<!DOCTYPE html>
<html>
<head>
  <meta charset="UTF-8">
  <title>LINE Manager Notifier - 設定</title>
  <style>
    body {
      font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, Helvetica, Arial, sans-serif;
      padding: 20px;
      color: #333;
    }
    
    h1 {
      font-size: 1.5em;
      margin-bottom: 20px;
      color: #06C755; /* LINE Green */
    }
    
    .setting-group {
      margin-bottom: 20px;
      padding: 15px;
      border: 1px solid #e0e0e0;
      border-radius: 5px;
    }
    
    .setting-group h2 {
      font-size: 1.2em;
      margin-top: 0;
      margin-bottom: 15px;
    }
    
    .form-control {
      margin-bottom: 15px;
    }
    
    label {
      display: block;
      margin-bottom: 5px;
      font-weight: bold;
    }
    
    input[type="number"], input[type="text"], input[type="password"] {
      width: 100%;
      padding: 8px;
      border: 1px solid #ddd;
      border-radius: 4px;
      box-sizing: border-box;
    }
    
    input[type="checkbox"] {
      margin-right: 5px;
    }
    
    button {
      background-color: #06C755; /* LINE Green */
      color: white;
      border: none;
      padding: 10px 15px;
      border-radius: 4px;
      cursor: pointer;
      font-size: 14px;
    }
    
    button:hover {
      background-color: #05a649;
    }
    
    .button-group {
      margin-top: 20px;
      text-align: right;
    }
    
    .file-input-group {
      display: flex;
    }
    
    .file-input-group input {
      flex-grow: 1;
      margin-right: 10px;
    }
    
    .version {
      margin-top: 20px;
      color: #999;
      font-size: 0.8em;
      text-align: center;
    }
  </style>
</head>
<body>
  <h1>LINE Manager Notifier 設定</h1>
  
  <div class="setting-group">
    <h2>通知設定</h2>
    <div class="form-control">
      <label for="checkInterval">未読チェック間隔（秒）</label>
      <input type="number" id="checkInterval" min="5" max="60">
    </div>
    <div class="form-control">
      <label for="notifyInterval">通知間隔（秒）</label>
      <input type="number" id="notifyInterval" min="10" max="600">
    </div>
    <div class="form-control">
      <input type="checkbox" id="customSoundEnabled">
      <label for="customSoundEnabled" style="display: inline;">カスタム通知音を使用</label>
    </div>
    <div class="form-control file-input-group">
      <input type="text" id="customSoundPath" placeholder="通知音ファイルのパス" disabled>
      <button id="selectSoundFile">参照...</button>
    </div>
  </div>
  
  <div class="setting-group">
    <h2>起動設定</h2>
    <div class="form-control">
      <input type="checkbox" id="startWithWindows">
      <label for="startWithWindows" style="display: inline;">Windowsと同時に起動</label>
    </div>
    <div class="form-control">
      <input type="checkbox" id="startMinimized">
      <label for="startMinimized" style="display: inline;">最小化状態で起動</label>
    </div>
  </div>
  
  <div class="setting-group">
    <h2>自動ログイン設定 (実験的)</h2>
    <div class="form-control">
      <input type="checkbox" id="autoLogin">
      <label for="autoLogin" style="display: inline;">自動ログインを有効にする</label>
    </div>
    <div class="form-control">
      <label for="lineUsername">LINEユーザー名/メールアドレス</label>
      <input type="text" id="lineUsername">
    </div>
    <div class="form-control">
      <label for="linePassword">LINEパスワード</label>
      <input type="password" id="linePassword">
    </div>
    <p style="color: #888; font-size: 0.9em;">※パスワードはローカルに保存されますが、セキュリティ上の理由により慎重に扱ってください。</p>
  </div>
  
  <div class="button-group">
    <button id="saveSettings">設定を保存</button>
  </div>
  
  <div class="version">LINE Manager Notifier v1.0.0</div>
  
  <script>
    // 設定を読み込み
    const settings = window.electronAPI.getSettings();
    
    // フォームに設定値を反映
    document.getElementById('checkInterval').value = settings.checkIntervalSeconds;
    document.getElementById('notifyInterval').value = settings.notifyIntervalSeconds;
    document.getElementById('customSoundEnabled').checked = settings.customSoundEnabled;
    document.getElementById('customSoundPath').value = settings.customSoundPath;
    document.getElementById('startWithWindows').checked = settings.startWithWindows;
    document.getElementById('startMinimized').checked = settings.startMinimized;
    document.getElementById('autoLogin').checked = settings.autoLogin;
    document.getElementById('lineUsername').value = settings.lineUsername;
    document.getElementById('linePassword').value = settings.linePassword;
    
    // カスタム通知音の有効/無効切り替え
    document.getElementById('customSoundEnabled').addEventListener('change', function() {
      document.getElementById('customSoundPath').disabled = !this.checked;
      document.getElementById('selectSoundFile').disabled = !this.checked;
    });
    
    // サウンドファイル選択ボタン
    document.getElementById('selectSoundFile').addEventListener('click', async function() {
      const filePath = await window.electronAPI.selectFile();
      if (filePath) {
        document.getElementById('customSoundPath').value = filePath;
      }
    });
    
    // 設定保存ボタン
    document.getElementById('saveSettings').addEventListener('click', function() {
      const newSettings = {
        checkIntervalSeconds: parseInt(document.getElementById('checkInterval').value, 10),
        notifyIntervalSeconds: parseInt(document.getElementById('notifyInterval').value, 10),
        customSoundEnabled: document.getElementById('customSoundEnabled').checked,
        customSoundPath: document.getElementById('customSoundPath').value,
        startWithWindows: document.getElementById('startWithWindows').checked,
        startMinimized: document.getElementById('startMinimized').checked,
        autoLogin: document.getElementById('autoLogin').checked,
        lineUsername: document.getElementById('lineUsername').value,
        linePassword: document.getElementById('linePassword').value
      };
      
      window.electronAPI.saveSettings(newSettings);
    });
    
    // 初期状態の反映
    document.getElementById('customSoundPath').disabled = !settings.customSoundEnabled;
    document.getElementById('selectSoundFile').disabled = !settings.customSoundEnabled;
  </script>
</body>
</html>

// README.md ファイル
# LINE Manager Notifier

LINE公式アカウント管理画面の未読通知を監視するデスクトップアプリです。

## 機能

- LINE公式アカウント管理画面の未読通知を定期的に確認
- 未読メッセージがある場合に通知と音でお知らせ
- システムトレイに常駐して軽量動作
- カスタマイズ可能な通知間隔と通知音
- Windows起動時の自動起動オプション

## インストール方法

### リリースからインストール

1. [Releases](https://github.com/yourusername/line-manager-notifier/releases) ページから最新版をダウンロード
2. ダウンロードしたインストーラーを実行
3. 画面の指示に従ってインストール

### ソースコードからビルド

```bash
# リポジトリをクローン
git clone https://github.com/yourusername/line-manager-notifier.git
cd line-manager-notifier

# 依存関係をインストール
npm install

# アプリケーションを起動
npm start

# ビルド
npm run build
```

## 使い方

1. アプリケーションを起動
2. LINEアカウントでログイン（通常のログイン画面が表示されます）
3. ログイン後はシステムトレイに最小化され、未読通知を監視します
4. 未読メッセージがあると、デスクトップ通知と音でお知らせ
5. 通知をクリックするとアプリが前面に表示されます

## 設定項目

- **未読チェック間隔**: 未読メッセージをチェックする間隔（秒）
- **通知間隔**: 未読メッセージがある場合に再通知する間隔（秒）
- **カスタム通知音**: 独自の通知音を設定可能
- **起動設定**: Windows起動時の自動起動と最小化設定
- **自動ログイン**: LINEアカウントの自動ログイン設定（実験的機能）

## ライセンス

MIT

## 注意事項

- このアプリケーションは非公式であり、LINE社とは関係ありません
- LINE公式アカウント管理画面のUIが変更された場合、動作しなくなる可能性があります
- 自動ログイン機能を使用する場合、パスワードはローカルに保存されます。セキュリティ上のリスクを理解した上でご使用ください

## 開発者向け情報

アプリケーションの仕組み：

1. ElectronでLINE公式アカウント管理画面をラップ
2. JavaScriptインジェクションで未読通知バッジを検出
3. 見つかった場合はシステム通知を表示
4. 設定はelectron-storeで保存
