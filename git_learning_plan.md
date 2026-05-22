# Git 深入学习路线

---

## 一、前置条件

- 有基本的 Git 使用经验（clone / add / commit / push / pull）
- 理解什么是版本控制，为什么需要它
- 有一个 GitHub / GitLab 账号

---

## 二、学习路线总览

```
┌───────────────┐    ┌───────────────┐    ┌───────────────┐    ┌───────────────┐
│ 第1周         │───▶│ 第2周         │───▶│ 第3周         │───▶│ 第4周         │
│ 内部原理      │    │ 分支与合并    │    │ 进阶技巧      │    │ 协作与工作流  │
└───────────────┘    └───────────────┘    └───────────────┘    └───────────────┘
```

---

## 三、阶段详细规划

### 第1周：内部原理 — Git 是怎么存数据的

**目标：理解 .git 目录结构，摆脱"黑盒"感**

#### Day 1：快照，不是差异

| 主题 | 内容 |
|------|------|
| 其他 VCS 的做法 | 基于差异（delta-based）：存储文件每次修改的 diff |
| Git 的做法 | 基于快照（snapshot-based）：每次提交存完整文件树 |
| 为什么快照更好 | 分支切换快、无需网络、数据完整性高 |
| SHA-1 哈希 | 一切内容（blob/tree/commit）都有唯一哈希 ID |

**练习：**
```bash
git init /tmp/git-demo && cd /tmp/git-demo
echo "v1" > file.txt && git add . && git commit -m "c1"
echo "v2" > file.txt && git add . && git commit -m "c2"
find .git/objects -type f | sort   # 看看有什么对象
```

#### Day 2：三种对象：blob / tree / commit

| 对象 | 说明 |
|------|------|
| **blob** | 文件内容（不含文件名），`git hash-object` |
| **tree** | 目录结构，记录文件名→blob 的映射 |
| **commit** | 一次提交：指向一个 tree + 父 commit + 元信息 |
| **tag** |  annotated tag 也是一个对象，指向 commit |

**练习：**
```bash
git log --oneline                          # 看 commit 列表
git cat-file -p HEAD                       # 看 commit 对象内容
git cat-file -p HEAD^{tree}                # 看 tree 对象内容
git cat-file -p HEAD:file.txt              # 等价于看 blob
git ls-tree HEAD                           # 列出 tree 内容
```

#### Day 3：暂存区（index）的本质

| 主题 | 内容 |
|------|------|
| 三个区域 | 工作目录 → 暂存区（index）→ 版本库 |
| index 文件 | `.git/index`，一个二进制文件，记录暂存状态 |
| `git add` | 将工作区的文件内容写入 blob，更新 index 指向新 blob |
| `git status` | 对比工作区 vs index vs HEAD 三者差异 |
| 分步暂存 | `git add -p` — 同一文件部分暂存 |

**练习：**
```bash
echo "change 1" >> file.txt
git add -p                               # 交互式选择暂存哪些修改
git diff                                # 工作区 vs 暂存区
git diff --cached                       # 暂存区 vs HEAD
```

#### Day 4：引用（refs）— 分支名的真相

| 主题 | 内容 |
|------|------|
| refs 目录 | `.git/refs/heads/` — 只是一个文件，存 commit SHA |
| HEAD | `.git/HEAD` — 指向当前分支的引用，或直接指向 commit（detached） |
| `git branch` | 创建分支 = 在 `refs/heads/` 下新建一个文件 |
| 远程引用 | `.git/refs/remotes/origin/` |

**练习：**
```bash
cat .git/HEAD                            # 看 HEAD 指向哪
cat .git/refs/heads/main                 # 分支就是一个 40 位 SHA
git branch test-branch
cat .git/refs/heads/test-branch          # 指向同一个 commit
git update-ref refs/heads/test-branch HEAD  # 底层创建/更新分支
```

#### Day 5：对象存储与打包

| 主题 | 内容 |
|------|------|
| 松散对象 | `.git/objects/xx/xxxx...` — 每个对象一个文件 |
| 打包文件 | `.git/objects/pack/` — 对象打包 + delta 压缩 |
| `git gc` | 垃圾回收，打包对象去重 |
| `git fsck` | 检查对象库完整性 |
| `git count-objects` | 统计对象数量 |

**练习：**
```bash
git count-objects -vH                     # 查看对象统计
git gc --aggressive                       # 强制打包
find .git/objects -type f | wc -l         # 对比前后文件数
```

#### Day 6：reset / checkout / restore 的底层原理

| 命令 | 行为 |
|------|------|
| `git reset --soft HEAD~1` | 只移动 HEAD，不改暂存区和工作区 |
| `git reset --mixed HEAD~1` | 移动 HEAD + 重置暂存区，保留工作区 |
| `git reset --hard HEAD~1` | 三者全部重置（危险！） |
| `git checkout <branch>` | 切换 HEAD 指向，更新暂存区和工作区 |
| `git restore <file>` | 从暂存区恢复工作区文件 |

**练习：** 创建一个 commit，分别用 `--soft / --mixed / --hard` 回退，观察 `git status` 的变化

#### Day 7：第一周综合练习

在 `/tmp` 创建一个本地仓库，完成以下操作：

1. 创建 5 个 commit，每次修改不同文件
2. 用底层命令（`git cat-file` / `git ls-tree`）追踪每个 commit 的 tree/blob 结构
3. 创建一个分支，观察 `.git/refs/heads/` 的变化
4. 用 `git reset --soft` 回退 2 个 commit，再 `git commit` 重新提交，对比 SHA 变化
5. 用 `git gc` 打包，观察 `.git/objects/` 的变化

---

### 第2周：分支与合并 — Git 的灵魂

**目标：熟练掌握 merge / rebase / cherry-pick，知道各自适用场景**

#### Day 8：merge 的三条路径

| 类型 | 条件 | 结果 |
|------|------|------|
| fast-forward | 目标分支是当前分支的直接后代 | 指针前移，无线程分叉 |
| 三方合并（3-way merge） | 两个分支分叉后各自有提交 | 生成一个合并提交（merge commit） |
| 冲突（conflict） | 同一文件同一位置被不同修改 | 手动解决 |

**练习：**
```bash
# fast-forward 场景
git checkout -b feature
echo "feat" > feat.txt && git add . && git commit -m "feat"
git checkout main
git merge feature                         # fast-forward

# 三方合并场景
git checkout main
echo "main change" >> file.txt && git add . && git commit -m "main work"
git merge feature                         # 生成 merge commit
git log --graph --oneline --all           # 可视化查看
```

#### Day 9：冲突解决

| 主题 | 内容 |
|------|------|
| 冲突标记 | `<<<<<<<` / `=======` / `>>>>>>>` |
| 解决流程 | 编辑文件 → `git add` → `git commit`（或 `git merge --continue`） |
| 取消合并 | `git merge --abort` |
| 冲突工具 | `git mergetool`（vimdiff / VSCode 三路合并） |
| conflict style | `merge.conflictStyle=diff3` 显示共同祖先 |

**练习：** 两个分支修改同一行，制造冲突→解决→提交

#### Day 10：rebase — 重写历史

| 主题 | 内容 |
|------|------|
| 什么是 rebase | 将当前分支的提交"搬到"目标分支的顶端 |
| merge vs rebase | merge 保留历史全貌；rebase 产生线性历史 |
| `git rebase main` | 把当前分支 rebase 到 main 上 |
| 交互式 rebase | `git rebase -i` — squash / fixup / reword / drop |
| 黄金法则 | **不要 rebase 已经 push 到共享仓库的提交** |

**练习：**
```bash
git checkout -b topic
echo "a" >> file.txt && git add . && git commit -m "topic a"
echo "b" >> file.txt && git add . && git commit -m "topic b"
git checkout main
echo "m" >> other.txt && git add . && git commit -m "main work"
git checkout topic
git rebase main                           # 把 topic 的提交搬到 main 之后
git log --graph --oneline --all
```

#### Day 11：交互式 rebase 实战

| 操作 | 说明 |
|------|------|
| `pick` | 保留该提交 |
| `reword` | 修改提交信息 |
| `squash` | 合并到上一个提交 |
| `fixup` | 合并到上一个提交，丢弃当前提交信息 |
| `drop` | 删除该提交 |
| `edit` | 暂停以修改提交内容 |
| `reorder` | 调整提交顺序 |

**练习：** 创建 5 个零散的小 commit，用 `git rebase -i` 整理成 2 个语义清晰的 commit

#### Day 12：cherry-pick & revert

| 命令 | 用途 |
|------|------|
| `git cherry-pick <commit>` | 将某个 commit 的变更"摘取"到当前分支 |
| `git cherry-pick --continue/--abort/--skip` | 遇到冲突时的处理 |
| `git revert <commit>` | 生成一个新 commit 来撤销指定 commit（安全！） |
| revert vs reset | revert 不破坏历史，适合远程分支 |

**练习：** 在一个分支修复了一个 bug（一个 commit），用 cherry-pick 把修复应用到另一个发布分支

#### Day 13：reflog — 你的后悔药

| 主题 | 内容 |
|------|------|
| reflog 是什么 | 记录 HEAD 和分支引用的每一次移动 |
| 保存时间 | 默认 90 天（可配置） |
| 恢复丢失的 commit | `git reflog` → 找到 SHA → `git reset --hard <sha>` |
| 恢复删除的分支 | `git checkout -b <name> <sha>` |

**练习：**
```bash
git commit --allow-empty -m "important"
git reset --hard HEAD~1                   # "丢了"这个 commit
git reflog                                # 找到它的 SHA
git reset --hard <sha>                    # 恢复
```

#### Day 14：第二周综合练习

模拟一个真实开发场景：

```
main ──●────●
        \    \
    feature  hotfix(bug修复)

任务：
1. 将 hotfix cherry-pick 到 feature
2. 将 feature 用 rebase 整理成 2 个干净 commit
3. rebase feature 到 main 最新位置
4. 解决过程中产生的任何冲突
5. 合并（使用 git merge，保留完整历史）
6. 用 git log --graph 确认最终历史线
```

---

### 第3周：进阶技巧

**目标：掌握 stash、bisect、worktree、submodule、hooks 等日常高频工具**

#### Day 15：stash — 暂时保存工作

| 命令 | 说明 |
|------|------|
| `git stash` | 暂存当前工作区的修改（push 到 stash 栈） |
| `git stash list` | 查看 stash 栈 |
| `git stash pop` | 弹出并应用最近一个 stash |
| `git stash apply` | 应用但不弹出 |
| `git stash -u` | 包含 untracked 文件 |
| `git stash -p` | 交互式选择要 stash 的内容 |
| `git stash branch <name>` | 从 stash 创建新分支 |

**练习：** 正在 feature 分支开发时，需要紧急切到 main 修 bug。用 stash 暂存→切分支→修复→切回→恢复

#### Day 16：bisect — 二分查找 bug 引入点

| 主题 | 内容 |
|------|------|
| 适用场景 | 发现一个 bug，知道在哪个版本开始出现 |
| 工作原理 | 二分查找：标记 good/bad → Git 自动 checkout 中间 commit |
| 手动模式 | `git bisect start` / `git bisect good` / `git bisect bad` |
| 自动模式 | `git bisect run <test-script>` — 用脚本自动跑 |

**练习：**
```bash
git bisect start
git bisect bad HEAD                       # 当前版本有 bug
git bisect good v1.0                      # v1.0 确认没问题
# Git 会自动 checkout 中间 commit...
# 测试后告诉它 good 还是 bad
git bisect run python test_bug.py         # 自动化二分查找
git bisect reset                          # 结束后恢复
```

#### Day 17：worktree — 同时操作多个分支

| 主题 | 内容 |
|------|------|
| 为什么需要 worktree | 不用来回切换分支就能在不同分支上工作 |
| `git worktree add` | 在另一个目录 checkout 一个分支 |
| `git worktree list` | 列出所有 worktree |
| `git worktree remove` | 删除 worktree |
| `git worktree prune` | 清理已删除 worktree 的记录 |

**练习：** 在主 worktree 开发 feature，用另一个 worktree 在 hotfix 分支修 bug，两边互不干扰

#### Day 18：blame & log 高级用法

| 命令 | 用途 |
|------|------|
| `git blame <file>` | 查看每行代码是谁在哪个 commit 修改的 |
| `git blame -L 10,20 <file>` | 只看指定行范围 |
| `git log -S "<code>"` | 搜索某段代码第一次出现/最后一次删除的 commit（pickaxe） |
| `git log -G "<regex>"` | 正则匹配代码变更 |
| `git log -- <path>` | 只看某个文件的变更历史 |
| `git log --since/--until/--author` | 按时间/作者过滤 |

**练习：** 在一个老项目中，找到某个函数是什么时候引入的、谁写的，用 `git log -S` 定位

#### Day 19：submodule & subtree

| 主题 | 内容 |
|------|------|
| submodule | 在仓库中引用另一个仓库（存一个指针到特定 commit） |
| `git submodule add` | 添加子模块 |
| `git submodule update --init --recursive` | 克隆后拉取子模块 |
| subtree | 将外部仓库合并为子目录（不需要单独 clone） |
| submodule vs subtree | submodule 更解耦但操作繁琐；subtree 简单但耦合度高 |

**练习：** 创建一个仓库，用 submodule 引入一个公共库（如 shared-config 仓库）

#### Day 20：Git Hooks

| Hook | 触发时机 |
|------|----------|
| `pre-commit` | `git commit` 之前，常用于 lint / format |
| `commit-msg` | 编辑完提交信息后，用于校验格式 |
| `pre-push` | push 之前，用于运行测试 |
| `post-checkout` | checkout 后 |
| `post-merge` | merge 后 |

**练习：** 写一个 pre-commit hook，阻止包含 `TODO` 注释的文件提交

#### Day 21：第三周综合练习

给定一个混乱的仓库：

```
任务：
1. 用 git log -S 找到某 bug 函数首次加入的 commit
2. 用 git bisect 确认 bug 引入点
3. 用 worktree 在另一个分支准备修复
4. 修复后 cherry-pick 到两个发布分支
5. 配置 pre-commit hook 防止类似问题再出现
6. 用 git stash 处理过程中被打断的场景
```

---

### 第4周：协作与工作流

**目标：理解团队协作模式和分支策略，能设计合理的 Git 工作流**

#### Day 22：远程协作核心

| 主题 | 内容 |
|------|------|
| 分布式模型 | 每个 clone 都是完整仓库，无单点依赖 |
| `git fetch` vs `git pull` | fetch 只下载不合并；pull = fetch + merge |
| `git push --force-with-lease` | 安全强制推送（对比远程引用，避免覆盖他人工作） |
| tracking branch | 本地分支跟远程分支的关联 |
| upstream | `git push -u origin <branch>` 设置上游 |

**练习：** 两个人（两个本地目录）模拟远程协作，体验 fetch / pull / push 的完整流程

#### Day 23：Pull Request / Merge Request 流程

| 步骤 | 说明 |
|------|------|
| 1. 创建功能分支 | `git checkout -b feature/xxx` |
| 2. 开发并提交 | 小步提交，提交信息清晰 |
| 3. rebase 整理 | 推送前用 rebase 清理本地历史 |
| 4. 推送分支 | `git push -u origin feature/xxx` |
| 5. 创建 PR | 在 GitHub/GitLab 上发起 |
| 6. Code Review | 根据反馈修改，追加 commit 或 force push |
| 7. 合并 | 由维护者决定 merge / squash merge / rebase merge |

**练习：** 在 GitHub 上创建两个账号的仓库（或用 fork），走一遍完整的 PR 流程

#### Day 24：三种合并策略对比

| 策略 | 效果 | 适用场景 |
|------|------|----------|
| `git merge` | 保留所有 commit + 生成 merge commit | 需要保留完整历史 |
| `git merge --squash` | 所有变更压成一个 commit | feature 分支的 commit 太零碎 |
| `git rebase + merge` | 线性历史，无 merge commit | 追求干净历史的小团队 |

**练习：** 同一个 feature 分支，分别用三种策略合并，对比 `git log --graph` 的结果

#### Day 25：分支策略

| 策略 | 核心思想 | 适用团队 |
|------|----------|----------|
| **GitHub Flow** | main + feature 分支，一切走 PR | 持续部署的小团队 |
| **Git Flow** | main / develop / feature / release / hotfix | 有明确发布周期的团队 |
| **GitLab Flow** | main + 环境分支（staging/production） | 多环境部署 |
| **Trunk-Based** | 只往 main 提交短生命周期分支 + 特性开关 | 成熟 DevOps 团队 |

**练习：** 为同一个项目模拟 Git Flow 流程（新功能 → develop → release → main + hotfix）

#### Day 26：提交信息规范

| 规范 | 格式 |
|------|------|
| Conventional Commits | `type(scope): description` |
| 类型 | feat / fix / docs / refactor / test / chore |
| 正文 | 空行后写 WHY，不需要复述 WHAT |
| 关联 issue | `Closes #123`、`Refs #456` |

**练习：**
```bash
# 为现有仓库的最近 5 个 commit 重写提交信息
git rebase -i HEAD~5    # 用 reword 改信息

# 或配置 commit.template 让每次提交都有规范格式
git config commit.template ~/.gitmessage
```

#### Day 27：Git 配置优化

| 配置项 | 推荐值 |
|--------|--------|
| `pull.ff` | `only`（禁止自动 merge，强制你决定） |
| `merge.conflictstyle` | `diff3`（显示共同祖先，更好解决冲突） |
| `rebase.autosquash` | `true`（fixup commit 自动排到正确位置） |
| `rerere.enabled` | `true`（记录冲突解决方式，下次自动应用） |
| `diff.colorMoved` | `zebra`（高亮代码移动） |
| alias | 自定义缩写（见下方） |

**推荐 alias：**
```bash
git config --global alias.lg "log --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
git config --global alias.unstage "restore --staged ."
git config --global alias.undo "reset --soft HEAD~1"
git config --global alias.amend "commit --amend --no-edit"
git config --global alias.prune-all "fetch --prune && git branch -vv | grep ': gone]' | awk '{print $1}' | xargs git branch -D"
```

#### Day 28：第四周综合练习

设计并实施一个完整的小团队 Git 工作流：

```
场景：一个 3 人团队维护一个 Python Web 项目

任务：
1. 完成仓库设置：
   - 保护 main 分支（不允许直接 push）
   - 配置提交信息模板（Conventional Commits）
   - 添加 pre-commit hook（格式检查）
   - 添加 pre-push hook（运行 pytest）

2. 模拟一个完整的功能开发周期：
   - 从 main 创建 feature 分支
   - 开发过程中多次 commit
   - 用 rebase 整理历史
   - 发起 PR（模拟 code review）
   - 选择合并策略并合并

3. 模拟一个 hotfix：
   - 从 main 创建 hotfix 分支
   - 修复后同时合并到 main 和 develop（Git Flow 模式）
   - 打 tag 发布

4. 编写团队 Git 规范文档（Git convention），包含：
   - 分支命名规范
   - 提交信息格式
   - PR 流程
   - Code Review 检查清单
```

---

## 四、里程碑检查点

```
Week 1 结束：✓ 能解释 git add/commit 在 .git 里实际做了什么
             ✓ 能手动画出 commit → tree → blob 的引用链
Week 2 结束：✓ 能在 merge / rebase / cherry-pick 间选择合适的策略
Week 3 结束：✓ 能独立用 bisect 定位 bug 引入点
Week 4 结束：✓ 能为团队设计合适的 Git 工作流并写出规范文档
```

---

## 五、推荐资源

| 类型 | 资源 |
|------|------|
| 必读书籍 | Pro Git（中文版）— git-scm.com/book/zh/v2 |
| 互动学习 | learngitbranching.js.org — 可视化学习分支操作 |
| 源码阅读 | git 源码 — github.com/git/git |
| 参考 | git 官方文档 — git-scm.com/docs |
| 工具 | tig（终端 Git 浏览器）、lazygit（TUI Git 客户端） |
| 文章 | "Git 内部原理" 系列（Pro Git 第 10 章） |
| 可视化 | "Think Like (a) Git" — think-like-a-git.net |
