# 第5章 引用与 HEAD：四十位散列怎么变成 master

**一句话**：`引用`是保存 `SHA-1` 值、有个简单名字的文件；分支只是 `refs/heads` 下一个可移动的指针，`HEAD` 则通常是一个指向该指针的`符号引用`。

**为什么需要它**：第 4 章造出了完整历史，却仍要靠 `git log 1a410e` 这样的写法记住起点。原书给的方案很朴素：如果有一个文件来保存 `SHA-1` 值、而该文件有一个简单的名字，用这个名字指针替代原始散列值就方便多了。在 Git 中，这种简单的名字叫`引用`（references，简写 refs），可在 `.git/refs` 目录下找到。这一层把「内容寻址」与「人类可读的分支名」接了起来，也是第 6—8 章所有分支、标签、远程操作共同的地基。

**核心结构**：三类引用——`分支引用`（`refs/heads`）、`标签引用`（`refs/tags`）、`远程引用`（`refs/remotes`）。两个命令——`git update-ref`（安全地更新引用，原书不提倡直接编辑引用文件）与 `git symbolic-ref`（读写 `HEAD` 的值）。一个特殊文件——`HEAD` 文件，通常是一个`符号引用`，内容形如 `ref: refs/heads/master`；执行 `git checkout test` 会把它改成 `ref: refs/heads/test`。第四种对象——`标签对象`，包含标签创建者信息、日期、注释与一个指针，通常指向`提交对象`而非`树对象`。两种标签——`轻量标签`只是一个固定的引用；`附注标签`会额外创建`标签对象`并让引用指向它。

**关键设计**：**名字与内容分离，且名字可以移动**。原书说「这基本就是 Git 分支的本质：一个指向某一系列提交之首的指针或引用」，并补上一句：`git branch <branch>` 实际上会运行 `git update-ref`，取得当前所在分支最新提交的 `SHA-1` 值，加入新引用中。于是创建分支的成本只是写一个 41 字节的文件；而提交时 `git commit` 会创建一个`提交对象`，并用 `HEAD` 文件中那个引用所指向的 `SHA-1` 值设置其父提交字段——**分支「自动向前移动」不是复制历史，只是把引用文件里的键换掉**。名字可写、对象不可变，这个不对称是理解第 7 章`变基`风险的钥匙。

**拆解**：

### `refs` 目录与 `git update-ref`

一个新仓库的 `.git/refs` 下先有 `heads` 与 `tags` 两个空目录。原书先演示「技术上」的做法：`echo 1a410efbd13591db07496601ebc7a059dd55cfe9 > .git/refs/heads/master`，随后 `git log --pretty=oneline master` 就能列出历史；紧接着强调不提倡直接编辑引用文件，应该用 `git update-ref refs/heads/master` 加上 `1a410efbd13591db07496601ebc7a059dd55cfe9`。再用 `git update-ref refs/heads/test cac0ca` 在第二个提交上建一个分支，`git log --pretty=oneline test` 就只显示从第二个提交往前的记录。第 12 章会说明为什么这个「更安全」很重要：`引用日志`由 `git update-ref` 更新，直接写文件则不会留下痕迹。

### `HEAD` 文件与`符号引用`

原书用两个 `cat .git/HEAD` 的输出讲清切换：`ref: refs/heads/master` 与 `ref: refs/heads/test`。所谓`符号引用`，表示它是一个指向其他引用的指针；`git symbolic-ref HEAD` 可查看，`git symbolic-ref HEAD refs/heads/test` 可设置，但不能把它设成不符合引用规范的值（原书给出 `fatal: Refusing to point HEAD outside of refs/` 的报错）。而在检出标签、提交或远程分支时，`HEAD` 文件可能直接包含一个 Git 对象的 `SHA-1` 值，仓库进入「`分离 HEAD`」状态——`分离头指针`这个词指的就是这件事。提交时谁来当父提交，取决于 `HEAD` 里这根指针此刻指到哪。

### `标签对象`、`附注标签`与`轻量标签`

原书说标签对象「像是一个永不移动的分支引用」，但实现上多一层：`git update-ref refs/tags/v1.0 cac0cab538b970a37ea1e769cbbde608743bc96d` 就是`轻量标签`的全部内容；而 `git tag -a v1.1 1a410efbd13591db07496601ebc7a059dd55cfe9 -m 'test tag'` 会创建一个`标签对象`，引用指向该对象而不是直接指向提交。用 `git cat-file -p` 打出来，里面是 `object`、`type commit`、`tag`、`tagger` 和注释。原书还特别说明标签对象并非必须指向提交：Git 源码项目把维护者的 GPG 公钥作为`数据对象`存进去再打标签，Linux 内核版本库首个标签对象指向的是一份树对象。第 9 章 `packed-refs` 里以 `^` 开头的行，正是为了记录附注标签被「剥离」后指向的那个提交。

### `远程引用`为什么只读

第三类引用是`远程引用`：添加远程版本库并推送后，Git 把最近一次通信时每个分支对应的值记在 `refs/remotes` 下，例如 `.git/refs/remotes/origin/master`。原书点明它与 `refs/heads` 下分支引用最主要的区别——**远程引用是只读的**。虽然可以 `git checkout` 到某个远程引用，但 Git 不会把 `HEAD` 指向它，因此永远不能通过 `commit` 更新它；Git 把这些引用当作「记录远程服务器上各分支最后已知位置状态的书签」来管理。第 8 章会接着讲：正因如此，`git fetch` 更新书签、`git merge` 才把内容并进本地分支，`git pull` 只是把两步合起来。

**今天怎么做**：想弄清「我到底在哪个分支、它指向哪个提交」，直接读 .git/HEAD 与对应 refs/heads 文件，比记命令更不容易被别名和 IDE 提示误导。**延伸，非语料原文**：packed-refs 之外的 reftable 存储、git symbolic-ref 在 main 默认分支下的写法、git switch 分离 HEAD 的新入口都不改变本章机制；这些不在本书语料里，只作衔接说明。

<div class="chapend" markdown="1">

**容易误解的地方**：把分支理解成「一条独立的提交序列」——它只是一个会随提交前移的引用，两个分支可以长时间指向同一个提交。另一个误区是把 `HEAD` 当成「最新提交」本身：它通常指向的是**分支引用**而不是提交，只有`分离 HEAD` 时才直接装散列值。还有人以为标签和分支差不多：分支引用会移动，`轻量标签`是固定引用，`附注标签`还多一个`标签对象`。

**记住这一句**：`引用`给 40 位键起名字，分支是会移动的名字，`HEAD` 指出当前用哪个名字；`git commit` 用它指向的键当父提交。

</div>
