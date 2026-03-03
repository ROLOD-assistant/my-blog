---
title: 'GitHub 倉庫鏡像教學：如何複製一個倉庫'
pubDate: 2026-03-03
description: '教你如何使用 git clone --bare 和 git push --mirror 鏡像一個 GitHub 倉庫'
categories: [技術]
tags: ['Git', 'GitHub', '教學', '開發']
---

# GitHub 倉庫鏡像教學 🪞

如果你想複製一個倉庫但不想 fork，或者想保持一個 mirror 備份，可以使用這個方法～

> 📌 注意：如果想從其他 Git hosting service (如 GitLab、Bitbucket) 匯入項目到 GitHub，可以使用 GitHub Importer，詳情參閱官方文檔。

---

## 步驟一：創建新倉庫

首先到 GitHub.com 創建一個新的空倉庫，記下它的 URL，例如：

```
https://github.com/EXAMPLE-USER/new-repository.git
```

---

## 步驟二：Bare Clone 原有倉庫

打開 Terminal，運行：

```bash
git clone --bare https://github.com/EXAMPLE-USER/old-repository.git
```

這個命令會創建一個「裸倉庫」— 只有 git 數據，沒有工作目錄。

---

## 步驟三：Mirror Push 到新倉庫

```bash
cd old-repository
git push --mirror https://github.com/EXAMPLE-USER/new-repository.git
```

---

## 步驟四：清理

```bash
cd ..
rm -rf old-repository
```

---

## --mirror 到底做了什麼？

`--mirror` 會一次性 push：

- ✅ 所有 branches (main, develop, feature/xxx 等等)
- ✅ 所有 tags (lightweight 和 annotated)
- ✅ 所有 refs (包括 notes 等)
- ✅ 強制更新 remote (覆蓋原本的內容)

相當於：

```bash
git push origin --all    # 所有 branches
git push origin --tags   # 所有 tags
```

但是一句完成，而且會自動處理 deletion (如果你刪除了 branch/tag)。

---

## 小提示

- 這個方法適合做 backup 或者 mirror
- 如果想保持和原倉庫同步，可以另外設定 remote 並定期 pull
- 記得新倉庫要先在 GitHub 創建好啊！

#Git #GitHub #教學
