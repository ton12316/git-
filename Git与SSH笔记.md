# Git 与 SSH 笔记（两类结构）

适用环境：**Windows + Git Bash**（与 Linux/macOS 命令基本一致）。

---

# 第一类：普通 Git 上传（GitHub 等通用流程）

先做 **SSH 密钥**，再做 **本地提交与推送**。下面每条命令下面跟一句「解释」，按顺序做即可。

---

## 1. 密钥和「文件夹」无关（先读这两句）

- 密钥保存在用户目录 **`~/.ssh/`**（例如 `C:\Users\你的用户名\.ssh`），**不是**你项目里的某个子文件夹。
- **一对密钥 + 在网站上传一次公钥**，这个账号下**所有仓库**都能用；换项目只改 **`git remote`**，不用每换一个仓库就 `ssh-keygen`。

---

## 2. 生成密钥（在任意目录打开 Git Bash 执行即可）

```bash
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
```

**解释：** 提示保存路径时直接回车（默认进 `~/.ssh/`）。`Enter passphrase` 若要**完全无口令**，连续两次回车。  
（也可用 RSA：`ssh-keygen -t rsa -b 4096 -C "你的邮箱@example.com"`，则下面文件名里的 `id_ed25519` 改成 `id_rsa`。）

---

## 3. 复制公钥到剪贴板

```bash
clip < ~/.ssh/id_ed25519.pub
```

**解释：** 没有 `clip` 就用 `cat ~/.ssh/id_ed25519.pub` 手动全选复制，**整行一行**，不要断行。

---

## 4. 在 GitHub 网页里登记公钥

打开：**GitHub → 头像 → Settings → SSH and GPG keys → New SSH key**，粘贴保存。

**解释：** 这里登记的是**公钥**；私钥永远不要上传、不要发给别人。

---

## 5. 测试 SSH 是否成功

```bash
ssh -T git@github.com
```

**解释：** 第一次会问 `Are you sure you want to continue connecting`，输入 `yes`。成功会看到 `Hi 用户名! ... authenticated`。

---

## 6.（可选）密钥权限太松时 SSH 拒绝使用

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
chmod 644 ~/.ssh/id_ed25519.pub
```

**解释：** 私钥文件权限必须是「仅本人可读写」，否则在 Git Bash/Linux 下可能报错。

---

## 7.（可选）一台电脑多把密钥：用 config 指定连 GitHub 用哪把

编辑文件 **`~/.ssh/config`**（在用户主目录下，**不是** `C:\` 根目录）：

```text
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

**解释：** `IdentityFile` 写你的私钥路径；`IdentitiesOnly yes` 避免 SSH 乱试别的钥匙导致失败或很慢。

---

## 8. 把本地项目第一次推到 GitHub（推荐顺序）

```bash
git init
git config user.name "你的名字"
git config user.email "你的邮箱@example.com"
git add .
git commit -m "初始提交"
git remote add origin git@github.com:你的用户名/仓库名.git # 给远程仓库起的别名（可自定义，比如 github_repo），origin 是行业通用默认名
git branch -M main    # 强制重命名分支，用于修改本地分支名称。-M代表「强制重命名分支」（即使目标分支已存在，也会覆盖）；若用小写 -m 则是「普通重命名」，目标分支存在时会报错，不覆盖。
```

若远程**已经**有内容（例如创建仓库时勾了 README），需要先拉再推：

```bash
git pull origin main --rebase
git push -u origin main
```

若远程**完全是空的**，可以直接：

```bash
git push -u origin main
```

**解释：**  
- `git remote add origin` **只做一次**；以后改地址用下面的 `set-url`。  
- 想让本机所有仓库默认用同一名字邮箱，把上面 `git config` 改成 `git config --global`（注意拼写是 **global**，不是 gloabl）。  
- 远程默认分支若是 `master`，把命令里的 `main` 改成 `master`。

---

## 9. 远程地址错了，或提示 origin 已存在

先看当前地址：

```bash
git remote -v
```

改地址（**不要**再执行 `git remote add`）：

```bash
git remote set-url origin git@github.com:你的用户名/仓库名.git
```

**解释：** `origin` 只能有一个；要换 URL 就用 `set-url`。

---

## 10.（可选）检查网络

```bash
ping github.com
ssh -T git@github.com
```

**解释：** 有的网络禁止 ping，**ping 不通不代表 Git 一定不能用**；以 `ssh -T` 为准。

---

# 第二类：GitLab（内网 SSH + 推送常见问题）

这一部分：**第一节是「第一次怎么配」**；**第二节是「出错了怎么办」**——每条都先写**问题是什么**，再写**命令**和**解释**。

---

## 2.1 第一次配置 GitLab SSH（按 A→F 顺序做）

### A. 生成只给 GitLab 用的密钥（避免和 GitHub 混用）

```bash
ssh-keygen -t ed25519 -C "你的邮箱@example.com" -f ~/.ssh/id_ed25519_gitlab
```

**解释：** `-f` 指定文件名；passphrase 不要就回车两次。在**任意目录**执行即可，不必先进 `.ssh` 文件夹。

### B. 复制公钥

```bash
clip < ~/.ssh/id_ed25519_gitlab.pub
```

**解释：** 网页打开 **GitLab → 头像 → Preferences → SSH Keys**，粘贴后保存。

### C. 写 `~/.ssh/config`（真实 IP、端口写在这里）

把下面整块放进 **`~/.ssh/config`**（按你机房**实际**改 `HostName` 和 `Port`；端口若是 22 可删掉 `Port` 这一行）：

```text
Host gitlab
  HostName 192.168.0.201
  Port 2222
  User git
  IdentityFile ~/.ssh/id_ed25519_gitlab
  IdentitiesOnly yes
```

**解释：**  
- `Host gitlab` 里的 **`gitlab`** 是**别名**，后面 `git clone`、`remote` 里要写这个名，不要写错。  
- SSH 实际连的是 `HostName` + `Port`，并自动用 `IdentityFile` 这把私钥。

### D. 权限

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/config
chmod 600 ~/.ssh/id_ed25519_gitlab
chmod 644 ~/.ssh/id_ed25519_gitlab.pub
```

**解释：** 和第一类一样，私钥、`config` 太开放会被 SSH 拒绝。

### E. 测试

```bash
ssh -T git@gitlab
```

**解释：** `git@gitlab` 里的 **`gitlab`** 必须和上面 `Host gitlab` 一致。看到 `Welcome to GitLab` 即成功。

### F. 克隆或修改远程（网页 SSH 链接怎么用）

**是的：** 路径来自项目页 **「代码」→「使用 SSH 克隆」** 里那一行，例如：

```text
git@192.168.0.163:hwhk_projects/hangar/driver/planelib_sense.git
```

**但不要整行照抄。** 正确做法是：

1. **只抄冒号 `:` 后面的路径**（从 `hwhk_projects` 一直到 `.git`）：  
   `hwhk_projects/hangar/driver/planelib_sense.git`
2. **把前面的 `git@192.168.0.163` 换成 `git@gitlab`**（`gitlab` 必须和你 `~/.ssh/config` 里 **`Host gitlab`** 一致）。

拼起来应是：

```bash
git clone git@gitlab:hwhk_projects/hangar/driver/planelib_sense.git
```

已有仓库时：

```bash
git remote set-url origin git@gitlab:hwhk_projects/hangar/driver/planelib_sense.git
git remote -v
```

**解释：** 网页上的 **`git@IP`** 往往配错了（例如仍是 163），直连会超时或连错机；**路径**一般是对的。用 **`git@别名`** 才能让 SSH 走你在 config 里写的真实 **`HostName` + `Port`**（见 **问题 1、2**）。若将来管理员把网页上的 IP 也改对了，你仍可以继续用 `git@gitlab:...`，不冲突。

---

## 2.2 问题速查：先认问题，再执行命令

---

### 问题 1：浏览器用 `192.168.0.201` 能打开 GitLab，但 Clone 里 SSH 显示的是 `git@192.168.0.163:...`

**原因：** GitLab 服务器上的配置（例如对外 SSH 主机名）还是旧 IP，网页上的克隆地址按**服务器配置**生成，**不一定**等于你地址栏里输入的 IP。

**解决办法：**  
- **你本机：** 以 **`~/.ssh/config` 里能连上的 `HostName` + `Port`** 为准，远程统一写成 `git@gitlab:路径`，**不要**用网页里那个错的 163。  
- **根治：** 找管理员改 GitLab 服务器配置（如 `gitlab.rb`）后 `gitlab-ctl reconfigure`，让网页显示的克隆地址也变对。

---

### 问题 2：`ssh: connect to host 192.168.0.163 port 22: Connection timed out`

**原因：** `git remote` 仍指向 **163 的 22 端口**，网络连不上（超时），**还没轮到密钥认证**。

**解决办法：**

```bash
git remote set-url origin git@gitlab:你的群组/你的项目.git
git push -u origin main
```

**解释：** `gitlab` 会走你在 config 里配的真实地址（例如 201:2222），不再走错误的 163:22。

---

### 问题 3：`error: remote origin already exists`

**解决办法：**

```bash
git remote set-url origin git@gitlab:你的群组/你的项目.git
git remote -v
```

**解释：** 已经有过 `origin` 时，**不要**再 `git remote add`，只用 `set-url` 改 URL。

---

### 问题 4：`! [rejected] main -> main (fetch first)` 或提示要先 `git pull`

**原因：** 远程 `main` 上已有提交（例如初始化时的 README），你本地没有这些提交。

**解决办法：**

```bash
git pull origin main
git push -u origin main
```

**解释：** 若出现 **拒绝合并不相关历史**，接着看 **问题 6**。

---

### 问题 5：`You are not allowed to force push` / `protected branch` / `pre-receive hook declined`

**原因：** GitLab 把 **`main`（或其它分支）设成受保护分支**，**禁止强推**。

**解决办法：** **不要**再使用 `git push --force`。改用 **pull 合并进历史** 再普通 `push`（见 **问题 6、7、8**）。若必须清空远程历史，只能由项目 **Maintainer** 改保护分支规则（一般不推荐对 `main` 放开强推）。

---

### 问题 6：`fatal: refusing to merge unrelated histories`

**原因：** 本地是 `git init` 后自己提交的历史，远程是 GitLab 建库时生成的历史，两边**没有共同祖先**。

**解决办法：**

```bash
git pull origin main --allow-unrelated-histories
```

若有冲突，解决后：

```bash
git add .
git commit -m "合并远程与本地"
git push -u origin main
```

**解释：** `--allow-unrelated-histories` 表示允许把两段「互不相关」的历史合成一条。  
**注意：** 若终端提示 **`main|MERGING`**，说明合并**还没结束**，必须先 **`git commit`**，**不能**这时就 `push` 成功。

---

### 问题 7：`git push --force-with-lease` 报错里带 `stale info`

**原因：** 本机还没 `fetch`，对「远程现在指到哪一次提交」的信息是旧的，`--force-with-lease` 不敢随便强推。

**解决办法：**

```bash
git fetch origin
git push --force-with-lease origin main
```

**解释：** 若执行后仍提示 **受保护分支**（问题 5），说明**仍然不能强推**，只能走合并（问题 6、8）。

---

### 问题 8：合并时出现 `CONFLICT (add/add)`，同一个文件两套完整内容

**原因：** 本地和远程各自加过同名文件，Git 不知道保留哪一版。

**解决办法（整文件保留「你本机正在写的」）：**

```bash
git checkout --ours 冲突的文件.c
git add 冲突的文件.c
git commit -m "完成合并"
git push -u origin main
```

多个文件可对每个文件执行 `checkout --ours`，或在你确认都要保留本地版时谨慎使用 `git checkout --ours .` 再 `git add -A`。

**解决办法（整文件保留「远程上的」）：** 把 **`--ours` 改成 `--theirs`**，再 `git add`、`git commit`、`git push`。

**解释：** 在「把远程合并进当前分支」这次操作里，**ours = 合并前你当前分支**，**theirs = 远程拉下来的那份**。

---

### 问题 9：出现 `warning: LF will be replaced by CRLF`

**原因：** Windows 下 Git 对换行符的自动转换提示。

**解决办法：** 一般**可以忽略**，不影响正常 `push`。若工作区总显示无关改动，再和团队统一 `.gitattributes` 或 `core.autocrlf` 策略。

---

### 问题 10：合并冲突改完了，`git push` 仍然 `non-fast-forward`

**原因：** 合并流程**没走完**：可能没执行 **`git commit`**，终端分支名旁还有 **`MERGING`**。

**解决办法：** 按顺序执行：

```bash
git status
git add .
git commit -m "完成合并"
git push -u origin main
```

**解释：** 正确顺序是：**解决冲突 → `git add` → `git commit`（结束合并）→ `git push`**。

---

*文件路径：`code_v1.0/GIT/Git与SSH笔记.md`*
