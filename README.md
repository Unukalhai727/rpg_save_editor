# RPG Save Editor

一个基于 Vue 3 的网页版 RPG Maker 存档修改器，用于解码、编辑并重新打包 `.rpgsave` 文件。

RPG Maker 存档内容是经过 LZString + Base64 编码的 JSON 字符串。本项目的所有文件处理均在浏览器本地完成，不会上传存档数据。

## 功能

- 点击选择或拖放 `.rpgsave` 文件
- 自动解码并格式化 JSON 内容
- JSON 语法高亮、行号与格式校验
- 支持 `Ctrl/Cmd + F` 查找定位
- 支持恢复导入时的原始内容
- 重新编码并下载修改后的存档

## 技术栈

- Vue 3
- TypeScript
- Vite
- LZString

## 本地运行

```sh
npm install
npm run dev
```

## 生产构建

```sh
npm run build
```
