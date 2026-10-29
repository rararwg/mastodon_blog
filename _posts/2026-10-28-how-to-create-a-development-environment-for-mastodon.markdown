---
layout: post
title:  "如何在部署长毛象之前预览和测试改动"
date:   2026-10-28 15:28:06 +0100
categories: mastodon 开发
---
如果你想对自己的长毛象实例做个性化的改动，那么最好是能在部署之前测试一下改动。因为这个测试只在每次更新代码的时候需要，所以完全可以在本地的机子上跑，这样也方便改代码和预览效果。我们可以使用 docker 来创造一个开发环境。

> 以下设置参考 [官方教程](https://github.com/mastodon/mastodon/blob/main/docs/DEVELOPMENT.md#docker)， 写这篇文章的时候版本是 `4.7.3`
{: .prompt-info }

首先安装 [docker](https://docs.docker.com/get-started/get-docker/) 以及 docker-compose 。我的操作系统是 MacOS，命令如下
```bash
brew install --cask docker
brew install docker-compose
``````
然后打开在桌面安装好的 docker 应用并登录。

接下来拉自己实例的代码
```bash
gh repo clone yourusername/mastodon
cd mastodon
```
然后跑 docker
```bash
docker compose -f .devcontainer/compose.yaml up -d
docker compose -f .devcontainer/compose.yaml exec app bin/setup
docker compose -f .devcontainer/compose.yaml exec app bin/dev
```
这些完成以后，我们就可以在 <http://localhost:3000> 看到实例跑起来了。自动创建的管理员账号是 `admin@localhost`，密码是 `mastodonadmin`。

然后就可以愉快地改代码了！

如果你只改 `.rb`, `.js` 和 `.jsx` 这些文件的话，只要保存改动就好了，刚刚跑的 `bin/dev` 会自动编译的。
效果满意之后，就可以 commit change 然后按照你熟悉的方式部署代码了。
