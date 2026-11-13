# 第8章 远端与引用规范：一次 fetch 到底搬了什么

**一句话**：`引用规范`用 `<src>:<dst>` 说清「把远端的哪些引用写到本地哪里」，`git fetch` 只更新`远程引用`这份只读书签，`git pull` 才是抓取之后再合并。

**为什么需要它**：第 5 章留下一个悬问：`refs/remotes` 下的`远程引用`是谁写进去的、凭什么写到那个路径。原书「引用规范」开头说，全书一直在用「远程分支到本地引用」这种简单映射，但映射其实可以更复杂；`git remote add origin git@gitserver:/srv/git/project.git` 会在 `.git/config` 里加一个小节，指定远程版本库名称、`URL` 与一个用于获取操作的`引用规范`。这一章把「和远端打交道」从一串命令还原成两条规则：**先按引用规范改名字，再按对象缺口搬数据**。后一半是第 9 章的事。

**核心结构**：一段配置——`[remote "origin"]` 下的 `url = https://github.com/libgit2/libgit2` 与 `fetch = +refs/heads/*:refs/remotes/origin/*`。一个格式——`引用规范`由可选的 `+` 号与紧随其后的 `<src>:<dst>` 组成，`<src>` 是代表远程版本库中引用的`模式`，`<dst>` 是本地跟踪的远程引用位置；`+` 告诉 Git 即使在不能`快进`的情况下也要强制更新引用。一组等价写法——`git log origin/master`、`git log remotes/origin/master`、`git log refs/remotes/origin/master` 都会被扩展成 `refs/remotes/origin/master`。一组动作——`git fetch`、`git push`、`git pull`、`引用规范推送`与`删除引用`。

**关键设计**：**引用先映射，对象后传输**。`git remote add origin` 生成的默认规范把服务器上 `refs/heads/` 下的所有引用写进本地 `refs/remotes/origin/`，所以 `git fetch` 之后你得到的是一批更新过的书签，而不是工作目录里被覆盖的代码——这解释了第 5 章「`远程引用`是只读的书签」为什么是必要的：若抓取直接改本地分支，本地未提交的提交就会被无声覆盖。原书还说明这条默认规范**只针对 `git fetch`**，命令行可以临时指定别的规范，配置文件里也能写多行 `fetch = +refs/heads/*:refs/remotes/origin/*` 分别取不同分支；`push = refs/heads/master:refs/heads/qa/master` 则决定 `git push origin` 的默认去向。

**拆解**：

### `<src>:<dst>` 与 `+` 号

原书拆得很细：可选的 `+` 号之后是 `<src>:<dst>`，`<src>` 是远端引用的模式，`<dst>` 是本地要写入的位置。若只想每次取 master 一支，把配置改成 `+refs/heads/master:refs/remotes/origin/master` 即可；若想临时取一次并放到别处，可写 `git fetch origin master:refs/remotes/origin/mymaster`。一次拉取多个分支时，原书给出 `[rejected]` 的例子：master 因不能`快进`被拒绝、topic 作为新分支被接受，解决办法是在规范前加 `+` 覆盖该规则。也就是说，`+` 不是「更强力地推」，而是放宽对目标引用能否快进的检查。

### 命名空间与部分通配符的限制

原书明确说「不能在模式中使用部分通配符」，`+refs/heads/qa*:refs/remotes/origin/qa*` 这类写法不合法；但可以用命名空间（目录）达到类似目的：写两行 `fetch = +refs/heads/master:refs/remotes/origin/master` 与 `fetch = +refs/heads/qa/*:refs/remotes/origin/qa/*`，就只取 master 和 QA 团队推上来的整组分支。原书举的场景是 QA 团队、开发人员、集成团队各推各的分支，在本地为它们建各自的命名空间。这也是「引用规范比一条命令复杂」的实际价值：它把协作约定写进了配置，而不是靠每个人记命令。

### 引用规范推送与删除引用

QA 团队要把分支放进远端的 `qa/` 命名空间，用的是 `git push origin master:refs/heads/qa/master`；若希望每次 `git push origin` 都这样，就在配置里加 `push = refs/heads/master:refs/heads/qa/master`。删除引用则利用 `<src>:<dst>` 的另一面：`git push origin :topic` 把 `<src>` 留空，等于把远端 topic 分支定义为空值，也就是删掉它；原书同时给出 Git v1.7.0 之后可用的 `git push origin --delete topic`。这一节还提醒：无法通过引用规范「从一个仓库获取并推送到另一个仓库」，那是另一类操作。

### 远程引用、跟踪分支与 `git pull`

第 5 章说远程引用是只读书签，这里补上它的更新时机：添加远程版本库并推送后，Git 会把最近一次通信时每个分支对应的值记在 `refs/remotes` 下。`跟踪分支`（也叫`远程跟踪分支`）是与某个远程分支建立关联的本地分支，`git pull` 的主体就是 `git fetch` 之后再 `git merge`。所以「`git fetch` 之后代码没变」不是故障，而是设计：**抓取改的是名字，合并才改内容**。把这条界线画清楚，才能理解第 6 章的`快进`与第 7 章的`强制推送`分别发生在哪一步。

**今天怎么做**：排查「远端有、本地看不到」时，先读 .git/config 里的 fetch = ... 确认命名空间有没有被过滤，再看 refs/remotes 下的书签值；这比反复重跑 git fetch 更快。延伸，非语料原文：Git 托管平台上的拉取请求、保护分支与 fork 工作流，以及 sparse-checkout 之类的按需取文件机制，都属于协作流程与后来特性，不在本书语料里。

<div class="chapend" markdown="1">

**容易误解的地方**：把 `git fetch` 当成「把远端代码拉到本地」——它只按`引用规范`更新`远程引用`，工作目录与本地分支都不动；`git pull` 才含合并。另一个误区是把 `+` 理解成「强制推送」：它作用在引用规范上，含义是允许非`快进`的引用更新，与命令行上的强制推送既相关又不是一回事。还有人以为 `origin` 是 Git 的特殊名字，其实它只是 `git remote add` 时给远程仓库起的默认名称，配置里完全可以叫别的。

**记住这一句**：`引用规范`先决定「哪些引用写到哪」，`git fetch` 据此更新只读的`远程引用`；代码变不变，取决于之后有没有合并。

</div>
