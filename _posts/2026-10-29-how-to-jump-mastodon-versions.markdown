---
layout: post
title:  "如何跨版本升级毛象"
date:   2026-10-29 15:00:06 +0100
categories: mastodon 开发
---
小站自从 3.5 版本后就几年没有更新了，最近一口气更新到了 4.7，尤其之前的操作系统还是很老的 Ubuntu 18.04，过程还是挺麻烦的，所以这里记录一下更新过程。

## 备份
‼️最重要的事情就是升级之前一定要备份‼️这样后面搞砸了可以回滚，我就搞砸了两次。我的服务器供应商是 Hetzner，那里可以创建 Backup，搞砸之后从 Backup 里 Rebuild 就好了。
## 升级

### 升级 Ubuntu 系统
新版本的毛象对系统版本也有要求，所以必须更新。为了保险起见，我这里是先更新到 20.04，然后把毛象 4.2 跑起来，确认没事之后再更新到 24.04，接着继续更新毛象。

Ubuntu 的升级过程是这样的
```bash
sudo apt update && sudo apt dist-upgrade -y
sudo apt autoremove -y
sudo do-release-upgrade
```
接着 `reboot` 就可以升级到下一个版本了。后续的升级重复以上步骤就可以了。

### 升级长毛象

官方的[升级指南](https://github.com/mastodon/mastodon/releases/tag/v4.7.4)在列出升级步骤的同时也会强调这是从上一个小版本的升级步骤，如果跨版本的话那么也要参照之前的升级指南。为了保险起见我是一个版本一个版本更新的，具体的过程是这样：

假设现在的版本是 4.2.x，接下来要更新到 4.3，那么就去查看 [4.4 的 release notes](https://github.com/mastodon/mastodon/releases/tag/v4.4.0)，上面写着
> The following instructions are for updating from 4.3.9.

所以我们的下一个更新目标就是 4.3.9。接下来查看从 4.3.0 到 4.3.9 所有的 release notes，看里面是否提到[标准升级过程](#标准升级过程)以外的东西。如果有的话，就先 `checkout` 那个版本，然后按照提示来。如果没有的话，直接 `checkout` 目标版本（比如 4.3.9），然后执行[标准升级过程](#标准升级过程)。确认跑起来之后，再重复以上步骤更新到下一个大版本。
#### 标准升级过程
更新 `ruby`。这个在升级前做一次就可以了。
```bash
su - mastodon
cd ~/.rbenv
git pull
cd ~/.rbenv/plugins/ruby-build
git pull
```
然后就是更新毛象
```bash
cd ~/live
RUBY_CONFIGURE_OPTS=--with-jemalloc rbenv install
bundle install && yarn install --immutable
RAILS_ENV=production bundle exec rails assets:precompile
SKIP_POST_DEPLOYMENT_MIGRATIONS=true RAILS_ENV=production bundle exec rails db:migrate
exit
```
接下来以 root 身份
```bash
systemctl restart mastodon-sidekiq
systemctl reload mastodon-web
```
然后继续
```bash
su - mastodon
cd ~/live
RAILS_ENV=production bundle exec rails db:migrate
```
如果站点开启全文搜索的话，可能还需要重新 config 一遍 elasticsearch，这里官方也有[教程](https://docs.joinmastodon.org/admin/elasticsearch/)。

## 遇到的问题
我在更新过程中踩了两个大坑
- **升级 PostgreSQL 的时候不小心删库了**

  升级 Ubuntu 系统的时候我一口气把 PostgreSQL 从 10 升级到了 18，看见数据库跑起来我以为万事大吉，就把旧版本删了，于是原来的数据库直接没了。正确的升级方式是这样的[^1]（假设从 PostgreSQL 10 更新到 12）:
  
  首先在 terminal 跑
  ```bash
  pg_lsclusters
  ```
  然后看 Ver 12 的 Status 是否是 online[^2]，如果是的话，需要手动关闭：
  ```bash
  pg_dropcluster 12 main --stop
  ```
  再跑一下 `pg_lsclusters`， 确认只有版本 10 是 online 状态，然后就可以开始迁移了
  ```bash
  pg_upgradecluster -m link 10 main
  ```
  这个过程非常快，我自己的小站是几秒钟就好了，然后再检查一下 `pg_lsclusters` 的状态，看到 Ver 12 现在显示的是 online 状态，并且 Port 是 5432（毛象 `.env.production` 里默认的 Port）。
  
  接下来可以直接重启毛象进程，也可以 `sudo -u postgres psql` 然后 `\du` 看看名为 `mastodon` 的用户是否还在。检查完没什么问题之后，就可以删掉旧版本了：
  ```bash
  pg_dropcluster 10 main
  apt remove postgresql-10 postgresql-client-10
  ```
- **原来的默认主题找不到了**

  我的小站添加了很多个性化主题。因为新版本的前端发生了很大的变化，所以我直接把主题全删掉了，准备之后再加上，结果升级完毕后小站显示小象敲键盘，看日志是 Webpack 在抱怨找不到 `flora.css`。这是因为小站的默认主题是 flora，但是现在被我删掉了，网站打不开我也没法更改默认，所以我直接复制了一份毛象默认主题的 `.css` 文件，然后重命名为 `flora.css`。这样小站就跑起来了，我再把默认主题改成毛象自带的。 

[^1]: 参考 [Upgrading Mastodon from PostgreSQL 15 to PostgreSQL 17](https://blog.thms.uk/2025/06/upgrade-postgres-15-17-mastodon/)
[^2]: 升级 Ubuntu 的时候系统会提示有新版本的 PostgreSQL，如果你选择现在创建新的 cluster，那么新版本就会是 online 状态，如果选否，那就不会是 online 状态。选哪个都行。
