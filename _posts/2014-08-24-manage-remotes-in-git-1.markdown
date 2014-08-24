---
layout: post
title: 在 Git 中使用多个 Remotes （一） - Github 工作流
tags:
    - git
    - vcs
    - github
comments: true
---

Git 的工作流非常灵活。像 linux 之类的开源社区项目一般是使用 clone -> commit -> format-patch -> mailing list -> am 的流程；而 github 上则采用 fork -> clone -> branch -> commit -> push -> pull request -> merge 的流程。Github 的流程就会用到多个 remotes.

举个例子，Alice 有个仓库 example. Bob fork 了这个仓库。然后的流程就大概是：

{% highlight sh %}
git clone git@github.com:bob/example.git
cd example
git remote add upstream https://github.com/alice/example.git
{% endhighlight %}

在 clone 时，git 会自动的加上一个 remote origin. 在做 fetch, pull, push 等操作时，如果不指明 remote, 就默认使用 origin.

添加 upstream 的目的，是为了保持和上游 Alice 的进度的同步。考虑以下情况：

{% highlight sh %}
git branch add-a-new-feature
git checkout add-a-new-feature
# make some changes and commits
{% endhighlight %}

然后，Bob 提交了 pull request, Alice 进行合并。这时，Alice 的 master 分支和 Bob 的 master 分支就不一样了。为了保持 master 分支同步，Bob 应该执行：

{% highlight sh %}
git checkout master
git fetch upstream  # 获得最新的 upstream/master
git merge --ff-only upstream/master
git push
{% endhighlight %}

在 github 流程中，fork 出来的仓库从不在 master 上提交，因此 merge 是应该是可以 fast-forward 的。这里加上 `--ff-only`, 是为了保证，在意外的修改该了 master 分支的情况下，这一步 merge 就会失败。

当然，pull request 被 merge 后，这个 `add-a-new-feature` 分支也就没用了。可以把它删除：

{% highlight sh %}
git branch -d add-a-new-feature     # 删除本地的分支
# 删除 Bob fork 的仓库中的分支
git push origin :add-a-new-feature  # 从语句上理解，就是用一个空白 ref 覆盖掉 origin/add-a-new-feature
{% endhighlight %}

这个是典型的 github 工作流程，在 github 的 help 中已经有介绍。在接下来的文章中，我会讲一个更复杂的情况，也就是这个 blog 的代码仓库的情况。
