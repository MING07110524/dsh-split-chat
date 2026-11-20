# 多对话分屏（Split board）

DSH Web 客户端插件：在主区并排显示多个会话（2、3、4…），每格是完整对话（消息流、输入框、流式更新），可随时增删。不改 DSH 源码，纯客户端扩展。

## 入口

- **打开面板**：左侧栏面板列表的「多对话」图标。
- **拖进来**（主要方式）：
  - 直接从左侧会话列表**拖动会话行**，松手放到分屏里 → 加入一列；
  - 拖到左栏的「多对话」图标上 → 直接打开分屏，并把拖的会话和你刚才所在的对话一起放进去；
  - 打开空分屏时会自动放入你刚才所在的对话，所以第一次进来就有一个。
- **加对话**（不用拖时的备用）：分屏头部「添加对话」→ 搜索/选择会话；或会话行「…」→「加入多对话」；或当前对话头部「加入多对话」。
- **每格**：顶部窄栏显示会话标题（点击 = 回到该会话的普通对话）+ 关闭 ×。
- **返回**：分屏头部「返回对话」；或左栏点任意会话（DSH 原生行为）。

## 关于右侧栏与文件预览

分屏占用了主区，DSH 认为「当前没有对话在屏」，所以右侧栏相关的动作（格子里点文件链接、「在侧边栏预览」）在分屏里没有可直接挂载的面板。

插件做了兜底：**检测到这种点击时，自动切回对话面板并在右侧栏打开预览**（不会再弹「无法打开文件」）。代价是这次点击会离开分屏，点「多对话」图标即可回来。

## 行为与限制

- 单行等宽列，列宽自动均分整屏（6 格也能全看见），最小 220px；窗口实在太窄时才会横向滚动。
- 上限 6 格（每格一条活会话流），到上限「添加对话」禁用、拖动不再加入。
- 拖放用的是 DSH 会话行自带的拖拽（原生用于列表内重排，payload 为 `text/plain` 的会话 id）；拖到分屏外不会触发重排。
- 成员列表存在 `localStorage['dsh.split-board.v1']`，重开 DSH 自动恢复；已归档/不存在的会话在列表就绪后自动清理。
- 每格是真实会话：在格子里输入会真的发给该会话。同一会话若同时开在普通对话与格子里，两侧共享同一 draft/composer 状态。
- **文件拖放**：把文件拖到**某一格**里 → 只加进该格会话的输入框（附件/上传，格子高亮提示）；拖到分屏头部、空白处或左栏 → 忽略（不导航、不卡死）。分屏内不接管从 DSH 文件面板发起的应用内文件拖拽（请用该格输入框的附件按钮）。
- 分屏打开期间，整个窗口的文件拖放都由本插件接管：不会再出现「一个文件同时进了所有格子」或「窗口跳到文件」的情况。
- 分屏不随主区会话切换而变化；点左栏会话会回到普通对话（DSH 原生行为）。
- 不做：拖拽调整列宽/顺序、快捷键、把格子拖进右栏。

## 安装 / 卸载

安装包名 `dsh-split-chat`：

```
plugin_manager install_bundle  target = dsh-split-chat
```

- 要求 DSH `>=0.2.2-rc.1 <0.3.0`。
- 本地开发装源码目录也行：`target = E:\Project\dsh-split-chat`（包名 `@local/dsh-split-chat`，链接安装）。
- 关闭：`plugin_manager set_bundle`（选择 `dsh-split-chat`，enabled=false）
- 移除：`plugin_manager remove_bundle`
- 改完 `client.js` 后需要刷新窗口或重启桌面应用，浏览器端不会自动热更。

## 文件

| 文件 | 作用 |
|---|---|
| `package.json` | bundle 声明：`dsh.bundle.patch` + `dsh.client`（web、immediately） |
| `cordis.patch.yml` | 插入 Loader 行 `split-board` |
| `index.js` | 宿主半边空实现（客户端插件的占位行） |
| `client.js` | 全部实现：资源协议、主面板、单元格、入口、样式 |
| `locale/zh.json`、`locale/en.json` | 插件管理页的标题/描述 |
| `icon.svg` | 插件图标 |

## 实现要点

- 资源协议 `dsh-resource://sessionchat/session/<id>`：`ctx.sessions.retain(id, {source:'splitBoard', signal})`，组件用 `useResource(address)` 挂载即持有、卸载即释放。
- 主面板：`main` 槽注册 `key: 'split-board'`，声明子槽 `split-board.cell`（`kind:'single'`, `scope:'session'`）以获得 `SessionProvider` / `renderSlot` / `renderFactorySlot`。
- 每格：`<SessionProvider session={reference}>` 包住 `split-board.cell` 槽，槽组件调用 `conversation.content` 工厂（`variant:'embedded'`，Chat 视图固定），复用 DSH 原生对话内容与 composer。
- 状态：一个 apply 级 observable（`useSyncExternalStore` 订阅），所有入口共享。
