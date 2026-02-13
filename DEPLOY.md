# GitHub Pages 部署指南

## 📋 前置准备

已完成：
- ✅ Git 仓库初始化
- ✅ 代码提交完成
- ✅ GitHub Pages 文件结构创建

---

## 🚀 部署步骤

### 步骤 1：在 GitHub 创建新仓库

1. 打开浏览器访问：https://github.com/new
2. 登录你的账号 (CyangZhou)
3. 填写仓库信息：
   - **Repository name**: `horse-year-celebration`
   - **Description**: 甲午马年新春盛典互动网页
   - **Public** (必须公开才能使用免费 GitHub Pages)
   - **不要**勾选 "Add a README file"
4. 点击 **Create repository**

### 步骤 2：推送代码到 GitHub

在 `output` 目录下打开终端，执行以下命令：

```powershell
# 添加远程仓库
git remote add origin https://github.com/CyangZhou/horse-year-celebration.git

# 推送代码
git push -u origin master
```

如果提示输入密码，请使用 **Personal Access Token**（不是 GitHub 密码）：
- 创建 Token: https://github.com/settings/tokens/new
- 勾选 `repo` 权限
- 复制生成的 token 作为密码

### 步骤 3：启用 GitHub Pages

1. 访问仓库设置页面：https://github.com/CyangZhou/horse-year-celebration/settings/pages
2. 在 **Source** 下选择：
   - Branch: `master`
   - Folder: `/ (root)`
3. 点击 **Save**
4. 等待 1-2 分钟部署完成

### 步骤 4：访问网页

部署完成后，网页地址为：

🔗 **https://cyangzhou.github.io/horse-year-celebration/horse-year-celebration.html**

---

## ⚡ 快捷命令（复制粘贴）

```powershell
# 在 output 目录执行
cd "e:\traework\00 ai助手研发\output"

# 添加远程仓库
git remote add origin https://github.com/CyangZhou/horse-year-celebration.git

# 推送代码
git push -u origin master
```

---

## 🔧 可选：重命名 HTML 文件

如果想让网页地址更简洁，可以把 `horse-year-celebration.html` 重命名为 `index.html`：

```powershell
# 重命名文件
Rename-Item horse-year-celebration.html index.html

# 提交更改
git add index.html
git rm --cached horse-year-celebration.html
git commit -m "Rename to index.html for GitHub Pages"

# 推送
git push
```

这样网页地址就变成：
🔗 **https://cyangzhou.github.io/horse-year-celebration/**

---

## ❓ 常见问题

### Q: 推送时提示 "Authentication failed"
A: 需要使用 Personal Access Token 代替密码，详见步骤 2。

### Q: GitHub Pages 显示 404
A: 检查仓库是否为 Public，Pages 设置是否正确。

### Q: 网页样式不显示
A: `.nojekyll` 文件已创建，应该不会有此问题。如果仍有问题，检查文件路径是否正确。
