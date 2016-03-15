---
layout: post
category: 杂谈
tags: []
title: Hello Let's Encrypt
---

这一波安利我必须要发！！！么么哒！

刚准备睡觉时，收到 [Kloudsec](https://kloudsec.com/) CEO Steven Goh 发来的一封邮件邀请我试用它们的 [GitHub Pages CDN 服务](https://kloudsec.com/github-pages)，能够让绑定了自定义域名的 GitHub Pages 也能用上 HTTPS。花了十分钟简单配置一下就完成了！

根据他们自己的 [How it works](https://kloudsec.com/#/features)，简单地说是这么回事：

- 你将你域名的 A 记录指向到他们的服务器
- 他们服务器会时刻镜像你原本站点的内容到他们 CDN 上
- 同时还会帮你配置好 [Let's Encrypt](https://letsencrypt.org) 的免费证书，自动续期

相比之下，类似的 CloudFlare 则需要你修改 NS 记录，换言之在 DNS 层面完全接管你的网站。而 Kloudsec 这种做法就只需要改 A 记录，正常多了！

好吧，睡觉了！
