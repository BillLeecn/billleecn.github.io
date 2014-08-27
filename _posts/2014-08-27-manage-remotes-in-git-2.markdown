---
layout: post
title: 在 Git 中使用多个 Remotes （二） - 部署
tags:
    - git
    - vcs
    - github
comments: true
---

有些服务可以使用 git push  部署，典型的例子就是 github pages. 这里，我就以 github personal pages 为例子，讲讲这种情况。

假设 Jekyll 模板是来自 Alice 的 templete 仓库。在使用这套模板的过程中，难免会需要修复 bug, 添加功能。我希望修复的 bug 能合并到上游，因此，我在 Github 上 fork 了这个仓库。

最后，为了发布 github personal pages, 还需要建立仓库 bob.github.io. 这样，本地的仓库就会有3个 remotes.

开始时，本地仓库的配置和两个 remotes 的情况类似。

{% highlight sh %}
git clone git@github.com:bob/templete.git
cd templete
git remote add upstream https://github.com/alice/templete.git
{% endhighlight %}

然后，我希望每次 push 时，默认 push 到 bob.github.io. 于是我要把 origin 改掉。

{% highlight sh %}
git remote remove origin
git remote add origin git@github.com:bob/bob.github.io.git
{% endhighlight %}

> 注意：这里用的是 github personal pages, 用来生成 pages 的是 master 分支（而不是 gh-pages 分支）。

然后我把新的 post commit 到 master, `git push` 就可以了。

现在，本地仓库只有两个 remote. 一个是 `upstream: alice/templete.git`, 另一个是 `origin: bob/bob.github.io`. 现在考虑这个情况：templete 中有个 bug, 我在使用过程中发现了，然后自己修复了，直接提交到 master 分支。现在，我希望把这个 bugfix 提交到 upstream.

按照 github 的工作流，我应该往在 fork 出的仓库中推送一个新分支，然后发起 pull request. 所以，我首先把 fork 出来的 bob/templete.git 添加到 remotes 中。

{% highlight sh %}
git remote add fork git@github.com:bob/templete.git
{% endhighlight %}

然后，要在 fork/master 上创建一个新分支：

{% highlight sh %}
# 取出 commit fork/master 并将其作为分支 fix-something
git checkout -b fix-something fork/master
{% endhighlight %}

然后，要把 bugfix 应用到这个分支上，并把这个分支推送到 bob/templete.git. 假设 bugfix 对应的 commit id 为 abcdef.

{% highlight sh %}
# 将 commit abcdef 应用到 HEAD 上
git cherry-pick abcdef
git push -u fork fix-something
{% endhighlight %}

现在，就可以在 github 上发起 pull request. 等到 Alice 合并后，再同步 templete 的 master.

{% highlight sh %}
git checkout -b fork-master fork/master
git fetch upstream
git merge upstream/master
git push fork HEAD:master   # 由于 fork-master 和 master 的名称不同，我们无法通过 --set-upstream 来指定默认 push 到哪个 master, 只能显示指定将 HEAD push 到 master
{% endhighlight %}

最后，我希望 bob.github.io 用的模板要跟随上游的更新。考虑到这是个私人性质的 repository, 我可以用 rebase 来修改它的历史。我直接把它 rebase 到新的 upstream/master 上。

{% highlight sh %}
git checkout master
git rebase upstream/master
git push -f
{% endhighlight %}

在 rebase 前，master 分支和 upstream 分支上都有 bugfix 的 commit. 但在 rebase 的时候，master 分支上的 bugfix 应该会被自动丢弃掉。最后的结果是只有 upstream/master 上的那个 bugfix.

最后，由于使用 rebase 改变了历史，在 push 的时候，需要加上参数 `-f` 强制覆盖掉远端 repository 的历史。
