# Vue 房间示例

该项目基于 **Vue 3 + Vite** 与 **LiveKit**，提供一个简单的房间示例：本地可以发布音视频，远程成员加入后会自动展示他们的视频流。支持在本地切换音频/视频、连接与断开房间等操作。

## 环境要求

- Node.js 20.19+（或 22.12+）
- 已注册的 LiveKit 云端或自建服务，能够获取房间的 `roomServer`（WebSocket 地址）与 `roomToken`

## 安装依赖

```bash
bun install
```

### 安装 LiveKit SDK

项目依赖 `livekit-client`。如果你是从模板或空项目迁移代码，请确保先安装该依赖：

```bash
bun add livekit-client
```

## 本地开发

```bash
bun run dev
```

启动后浏览器打开命令行输出的地址。默认会使用 `src/App.vue` 中内置的示例 LiveKit 服务地址，但推荐在 URL 查询参数中传入你自己的配置：

```
http://localhost:5173/?roomServer=wss://xxx.livekit.cloud&roomToken=your_token
```

| 参数名      | 说明                                     |
| ----------- | ---------------------------------------- |
| roomServer  | LiveKit 房间信令地址（以 `wss://` 开头） |
| roomToken   | 用于认证的 Access Token                  |

## 生产构建

```bash
bun run build
```

构建后的文件位于 `dist/` 目录，可通过任意静态服务托管。

## 使用说明

1. 浏览器会请求摄像头与麦克风权限，允许后即可在页面最上方看到本地视频。
2. 点击 “连接房间” 将使用当前 URL 参数中的 `roomServer` 与 `roomToken` 连接 LiveKit。
3. 上方按钮可切换本地音频、视频或断开连接；下方 “远程参与者” 区域会列出房间内的其他用户。
4. 当只有两人时，切换本地音视频不会重复渲染视频容器；重新开启视频会立即恢复本地预览。

若需更复杂的功能（屏幕共享、录制、数据通道等），可在本示例基础上扩展。建议查阅 [LiveKit 官方文档](https://docs.livekit.io) 了解更多能力。*** End Patch
