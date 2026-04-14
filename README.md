# music163-api

Claude Code 插件，通过网易云音乐内部 Web API 自动化管理歌单。如批量导入歌曲、分类歌单等操作。

## 这是什么

逆向工程自 music.163.com 的内部 weapi 接口，配合 `chrome-devtools-mcp`，让 Claude 能够直接在已登录的 Chrome 浏览器中操作网易云音乐歌单——无需官方 SDK，无需独立登录流程。

## 前置条件

在 music.163.com 登录网易云音乐即可。认证信息（`MUSIC_U` cookie）的获取有三种策略，按优先级自动选择：

1. **自动提取**（推荐）：配置了 `chrome-devtools-mcp` 时，自动从浏览器读取 cookie，无需手动操作
2. **文件/环境变量**：从 `.env` 或环境变量 `MUSIC_U` 读取
3. **手动提供**：以上都没有时，引导你从浏览器 DevTools 复制 cookie

## 安装

```bash
claude --plugin-dir /path/to/music163-api
```

## 已实现的操作

| 操作 | 接口 | 状态 |
|------|------|------|
| 获取账户信息（uid） | POST weapi | 可用 |
| 列出用户所有歌单 | POST weapi | 可用 |
| 获取歌单歌曲列表 | POST weapi | 可用 |
| 向歌单添加歌曲（批量） | POST weapi | 可用 |
| 从歌单删除歌曲 | POST weapi | 可用 |
| 创建歌单 | POST weapi | 可用 |
| 删除歌单 | POST weapi | 可用 |
| 搜索歌曲 | POST weapi | 可用 |
| 红心歌曲（喜欢/取消喜欢） | POST weapi | 可用 |
| 获取红心歌曲列表 | POST weapi | 可用 |
| 获取歌曲详情（批量） | POST weapi | 可用 |
| 文字导入（搜索 + 批量添加） | 组合流程 | 可用 |

## 使用示例

```
帮我导入这些歌到"我喜欢的音乐"歌单：
曼丽 - 晚秋
张子铭 - 月半弯
王韵壹 - 红豆
消愁 - 毛不易
区瑞强 - 飞花
```

```
桌面上有个 songs.txt，每行一首歌，帮我全部导入到"跑步歌单"。
```

```
帮我把"我喜欢的音乐"里所有 Live 版本的歌移到"现场舞台"歌单里
```

## 相关项目

- [qqmusic-api](https://github.com/XHXIAIEIN/qqmusic-api) — QQ 音乐版，同样的使用体验，管理 QQ 音乐歌单

## 免责声明

本插件使用的是网易云音乐**非官方内部 API**，这些接口：

- 随时可能因网易后端变更而失效
- 不保证长期稳定可用
- 使用时需遵守网易云音乐用户协议
- 本项目仅供个人学习和自动化用途，不得用于商业目的

## License

MIT
