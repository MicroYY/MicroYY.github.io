---
title: Chirpy Front Matter设置汇总
date: 2026-01-30 20:00:00 +0800
author: Double Young

categories: [技术笔记]
tags: [前端, web, jekyll, chirpy]

pin: false
image:
  #path:
  #alt:
  #lqip:

description:

mermaid: false
math: false
comments: true
toc: true

media_subpath:
permalink:
published: true
render_with_liquid: false
---

整理了一个**全能型模板**，它合并了 Chirpy 主题支持的所有核心 Front Matter 设置。你可以直接将其保存为 Markdown 文件的开头，按需修改或取消注释。

```yaml
---
# [必填] 基础信息
title: "这里输入你的文章标题"
date: 2026-01-30 16:00:00 +0800 # 建议保留时区偏移量
author: Jerry                 # 对应 _data/authors.yml 的 ID 或直接写名字

# [推荐] 分类与标签 (分类建议不超过 2 个，标签不限)
categories: [技术, 后端]
tags: [jekyll, chirpy, markdown]

# [视觉] 首页置顶与封面图
pin: true                     # 置顶显示在首页
image:
  path: /assets/img/cover.jpg # 图片路径
  alt: "封面图描述文字"        # 辅助功能描述
  lqip: "/assets/img/low-res.jpg" # 模糊占位图 (可选)

# [SEO] 摘要描述 (显示在首页预览和搜索结果中)
description: "这是一篇关于 Chirpy 主题所有 Markdown 头部配置的详细总结。"

# [功能开关] 按需开启 (不使用时建议设为 false 或删除，以优化性能)
mermaid: true                 # 启用流程图/时序图支持
math: true                    # 启用数学公式支持
comments: true                # 启用/关闭评论 (默认 true)
toc: true                     # 启用/关闭右侧目录 (默认 true)

# [高级设置]
media_subpath: /assets/img/posts/2026-01-30/ # 简化正文图片引用路径
permalink: /posts/full-config-demo/        # 自定义固定链接
# published: false                           # 设置为 false 则作为草稿不发布
# render_with_liquid: false                  # 提高构建速度 (若正文无 Liquid 标签)
---

# 正文从这里开始

```

---

### 💡 使用小技巧：

1. **关于 `media_subpath**`：设置了这个路径后，你在正文中插入图片只需写 `![图片说明](photo.jpg)`，系统会自动补全为 `/assets/img/posts/2026-01-30/photo.jpg`。
2. **关于 `pin**`：建议全站置顶文章不要超过 3 篇，否则会占据过多的首页首屏空间。
