---
title: 'GitHub 仓库镜像教学：点样复制一个仓库'
pubDate: 2026-03-03
description: '教你点样用 git clone --bare 同 git push --mirror 镜像一个 GitHub 仓库'
categories: [技術]
tags: ['Git', 'GitHub', '教學', '開發']
---

# GitHub 仓库镜像教学 🪞

如果你想复制一个仓库但唔想 fork，或者想保持一个 mirror 备份，可以用到呢个方法～

> 📌 注意：如果想从其他 Git hosting service (如 GitLab、Bitbucket) 导入项目到 GitHub，可以用 GitHub Importer，详情睇官方文档。

---

## 步骤一：创建新仓库

首先去 GitHub.com 创建一个新既空仓库，记低佢既 URL，例如：

```
https://github.com/EXAMPLE-USER/new-repository.git
```

---

## 步骤二：Bare Clone 原有仓库

打开 Terminal，运行：

```bash
git clone --bare https://github.com/EXAMPLE-USER/old-repository.git
```

呢个命令会创建一个「裸仓库」— 即係得 git 数据，冇工作目录。

---

## 步骤三：Mirror Push 到新仓库

```bash
cd old-repository
git push --mirror https://github.com/EXAMPLE-USER/new-repository.git
```

---

## 步骤四：清理

```bash
cd ..
rm -rf old-repository
```

---

## --mirror 到底做咗啲咩？

`--mirror` 会一次性 push：

- ✅ 所有 branches (main, develop, feature/xxx 等等)
- ✅ 所有 tags (lightweight 同 annotated)
- ✅ 所有 refs (包拪 notes 等)
- ✅ 强制更新 remote (overwrite 原本既嘢)

即係相当于：

```bash
git push origin --all    # 所有 branches
git push origin --tags   # 所有 tags
```

但係一句过，而且会自动处理 deletion (如果你删咗 branch/tag)。

---

## 小提示

- 呢个方法适合做 backup 或者 mirror
- 如果想保持同原仓库同步，可以另外 set remote 同定期 pull
- 记得新仓库要先在 GitHub 创建好啊！

#Git #GitHub #教學
