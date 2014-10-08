---
layout: post
title: 修改 git 历史的各种方法
tags: 
    - git
    - rebase
comments: on
---

1. 取消上一次 commit (保留在 stage):

    ```
    git reset HEAD^
    ```

2. 取消最后两个 commit (完全丢弃):

    ```
    git reset --hard HEAD^^
    ```

3. 修改上一次的 commit:

    ```
    git add something
    git commit --amend
    ```

4. 取消倒数第三、第二个 commit:

    ```
    git rebase -i HEAD~3
    ```

    然后在编辑器中删除第一行和第二行后，保存退出。

5. 修改倒数第三个 commit:

    ```
    git rebase -i HEAD~3
    ```

    然后把第一行的 pick 改成 edit.

6. 合并倒数第三、第二个 commit:

    ```
    git rebase -i HEAD~3
    ```

    然后把第二行 pick 改成 squash 或 fixup.

git rebase --interactive 特别强大，几乎可以实现所有修改历史的需求。
