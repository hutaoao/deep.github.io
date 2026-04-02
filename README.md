# 独行的风

基于 [Jekyll](https://jekyllrb.com/) 主题 [Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) 搭建的个人博客。

## 环境要求

- **Ruby** >= 3.1（推荐使用 [rbenv](https://github.com/rbenv/rbenv) 或 [rvm](https://rvm.io/) 管理版本）
- **Bundler**
- **Git**

## 快速开始（换电脑后重新搭建）

### 1. 安装 Ruby

**macOS（推荐使用 Homebrew）：**

```bash
brew install ruby
```

或使用 rbenv（可管理多版本）：

```bash
brew install rbenv ruby-build
rbenv install 3.3.0        # 安装指定版本
rbenv global 3.3.0         # 设为默认版本
```

> 安装完 rbenv 后，确保 shell 配置文件（`~/.zshrc`）中有初始化代码：
> ```bash
> eval "$(rbenv init - zsh)"
> ```

**Linux（Ubuntu/Debian）：**

```bash
sudo apt install ruby-full build-essential
```

**Windows：** 推荐使用 [RubyInstaller](https://rubyinstaller.org/) 或 WSL。

### 2. 验证环境

```bash
ruby -v    # 应输出 ruby 3.x.x
gem -v     # 应输出 gem 3.x.x
```

### 3. 拉取代码并安装依赖

```bash
git clone https://github.com/hutaoao/hutaoao.github.io.git
cd hutaoao.github.io

# 安装 Bundler（如未安装）
gem install bundler

# 安装项目依赖
bundle install
```

> 如果遇到权限问题，可以使用 `bundle install --path vendor/bundle` 将依赖安装到项目目录下。

### 4. 本地预览

```bash
# 启动开发服务器（默认端口 4000）
bundle exec jekyll s

# 指定端口
bundle exec jekyll s -P 8080
```

启动后访问 `http://localhost:4000` 即可预览。

> `jekyll s` 是 `jekyll serve` 的缩写，两者完全等价。

### 5. 发布文章

项目配置了 [GitHub Actions](.github/workflows/pages-deploy.yml)，推送到 `main` 分支后会自动构建并部署到 GitHub Pages，无需手动构建。

```bash
git add .
git commit -m "新增文章"
git push
```

推送后在 GitHub 仓库的 Actions 页面可查看构建和部署状态。

## 常用命令

| 命令 | 说明 |
|------|------|
| `bundle exec jekyll s` | 启动本地开发服务器 |
| `bundle exec jekyll s --draft` | 开发模式下包含草稿文章 |

## 写文章

在 `_posts/` 目录下新建 Markdown 文件，文件名格式为 `YYYY-MM-DD-文章标题.md`。

文件头部需要包含 Front Matter：

```yaml
---
title: 文章标题
date: 2026-04-01
categories: [分类]
tags: [标签1, 标签2]
---
```

草稿放在 `_drafts/` 目录下，使用 `jekyll s --draft` 预览。

## 项目配置

主要配置文件为 `_config.yml`，包含站点标题、描述、主题、社交链接等设置。

## 参考链接

- [Jekyll 官方文档](https://jekyllrb.com/docs/)
- [Chirpy 主题文档](https://github.com/cotes2020/jekyll-theme-chirpy)
- [Markdown 指南](https://www.markdownguide.org/)
