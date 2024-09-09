---
title: blog reborn
author: Double Young
date: 2024-07-30 20:12:00 +0800
categories: [计算机科学]
tags: [GitHub Pages, jekyll, chirpy, blog]
render_with_liquid: false
---

# 得与失
上个月的某一天打开邮箱，发现订阅了好些年的服务器供应商早在上上个月就发邮件提醒续费。由于自己没有备份的意识，服务器到期直接导致在上面建立的博客数据全部丢失，几年的记录和积攒彻底变成了回忆。懊悔之余开始反思，大概有一小一大两点需要自己以此为鉴：

- 数据备份意识的缺失。脑海中的记忆会模糊，所以我们喜欢通过博客、照片、视频来记录。对于这些数字化的信息，本地备份结合云备份，我们可以一定程度上让其成为永恒。但这种方式也并不是万无一失，存储介质寿命犹有竟时，网络云存储指不定也有宕机的一天，就像《三体》中人类文明即将被二向箔吞噬，一切的科技都无能为力，文明的痕迹只能通过石头来记录。

- 工作流可以改进。这次的问题完全是个人主观上的大意，但很明显可以借助一些客观的工具来弥补，在设计上进行防呆。比如通过类似ToDo的工具定期提醒进行备份，或者通过脚本自动备份，免去任何人为的失误。这是很有意义的一点，完全可以抽象并推广到生活的各个细节。

# new blog

现在的博客方案，我选择了[GitHub Pages](https://docs.github.com/zh/pages)。这是GitHub官方给出的静态站点托管服务，它直接从用户的代码仓库中构建并发布网站，不需要云服务器，也完全免费，只是有一定的仓库空间和站点带宽的限制，也足够应付咱们这个小破站了。静态站点生成器我用的是GitHub官方推荐的[Jekyll](https://jekyllrb.com/)，主题用的是GitHub高赞的[chirpy](https://github.com/cotes2020/jekyll-theme-chirpy)，chirpy在它们的demo里给出了非常实用的[指南](https://chirpy.cotes.page/)，基本上一路傻瓜式部署。最后按照GitHub Pages的[教程](https://docs.github.com/zh/pages/configuring-a-custom-domain-for-your-github-pages-site/about-custom-domains-and-github-pages)绑定了域名。

# 新的体验

这套方案通过代码仓库构建静态网站，不需要购买云服务器，不需要折腾最好的语言、sql、wordpress/typecho，部署简单无比，可以说是我这个菜鸟加懒人的福音。目前觉得体验欠佳的地方就是写文章时没有编辑器，在vscode里手写markdown还是感觉效率略低，尤其是需要填写一些诸如标题，分类，tag等格式化的信息，十分繁琐。后面会继续看看有没有可以增强体验或者好玩的东西。