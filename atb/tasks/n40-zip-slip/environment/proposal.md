# 方案：插件包上传与自动解压

## 背景
作者上传整合包（zip），系统解压后分析其中内容。

## 技术方案

> 需要引入新依赖 `adm-zip`（package.json 里目前没有）。

### 1. 上传
```ts
const buffer = await req.arrayBuffer();
const zip = new AdmZip(Buffer.from(buffer));
zip.extractAllTo(extractDir, true);
```

### 2. 分析
遍历解压目录，列出所有文件与体积。

### 3. 展示
在整合包详情页展示文件清单。

## 预期
- 自动分析包内容
- 作者无需手动填清单
