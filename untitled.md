---
description: nvm 是一款管理node版本的工具，能很方便地切换电脑上的node版本。本文是我在macbook上使用nvm的笔记。
---

# nvm使用

## 安装nvm

```
brew install nvm
```

{% hint style="info" %}
 Super-powers are granted randomly so please submit an issue if you're not happy with yours.
{% endhint %}

使用`nvm -v`

### 安装node

{% code title="" %}
```bash
# 安装特定版本的node，例如 12.12.0 版本
nvm install 12.12.0
# 安装最新版的node
nvm install node
```
{% endcode %}

### 查看可用版本

```bash
nvm ls-remote
```

