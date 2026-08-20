---
title: '{{ replace .File.ContentBaseName "-" " " | title }}'
description: 
date: {{ .Date | time.Format "2006-01-02 15:04:05" }}
# lastmod 无需手填：hugo.toml 已启用 enableGitInfo，构建时自动取 git 最近提交时间
image: cover.jpg
categories:
  - 
tags:
  - 
---
