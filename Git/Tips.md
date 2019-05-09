[TOC]

## 1. git pull时出现冲突,放弃本地修改，使远程库内容强制覆盖本地代码

```
git fetch --all //只是下载代码到本地，不进行合并操作

git reset --hard origin/master  //把HEAD指向最新下载的版本
```

## 2. 在 add 以后 commit 前撤销修改:

```
git reset <file> // 撤销提交单独文件

git reset        // unstage all due changes
```

## 3. add以及commit 前撤销对文件的修改:

```
git checkout -- README.md  // 注意, add添加后(同commit提交后)就无法通过这种方式撤销修改
```

## 4. 当前工作区内容已被修改，但是并未完成。而前面的分支上面有一个Bug，需要立即修复。可是又不想提交目前的修改，因为修改没有完成。但是，不提交的话，又没有办法checkout到前面的分支。

```
此时用Git Stash就相当于备份工作区了。

然后在Checkout过去修改，就能够达到保存当前工作区，并及时恢复的作用。
```

## 5. 删除某个远程分支

```
$ git push origin :master
# 等同于
$ git push origin --delete master
```

## 6. git log的通用配置命令

```
$ git config --global alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
```

## 7. 基于某个分支创建分支

```
$ git checkout <remote>/<branch> -b <branch> //设置跟踪关系

$ git pull origin auto-test-v0.9:autotest
```

## 8. 查看某文件修改历史

```
git log -- filename （git log filename）可以看到该文件相关的commit记录

git log -p filename    可以显示该文件每次提交的diff

git show commit-id     根据commit-id查看某个提交

git show commit-id filename    查看某次提交中的某个文件变化
```

## 9. 文件名大小写

只好用 --force了，强制更新掉远程的文件

> git mv --force filename FILENAME

或者实在喜欢简短命令的

> git mv -f filename FILENAME

简单粗暴点的办法就是直接配置git更省事儿

可以通过git config --get --global core.ignorecase 查看默认配置

通过git config core.ignorecase false设置为区分大小写

> git config --global core.ignorecase false

## 10. matching和simple

git push时有俩参数，‘matching’ 参数是 Git 1.x 的默认行为，其意是如果你执行 git push 但没有指定分支，它将 push **所有你本地的分支**到远程仓库中对应匹配的分支。

而 Git 2.x 默认的是 simple，意味着执行 git push 没有指定分支时，只有**当前分支**会被 push 到你使用 git pull 获取的代码

## 11. Windows/Unix换行符

> git config --global core.autocrlf false //* 让Git不要管Windows/Unix换行符转换的事

## 12. 中文乱码

> git config --global gui.encoding utf-8 #//避免git gui中的中文乱码

> git config --global core.quotepath off // 避免git status显示的中文文件名乱码

## 13. 统计代码

### 统计所有的增删行数

> git log --pretty=tformat: --numstat | gawk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2} END { printf "added lines: %s, removed lines : %s, now lines: %s, total lines: %s\n",add,subs,loc,total }'

### 扫描 Log 单独统计每个人的增删行数加强版

> echo -en "\n"; git log --pretty=tformat: --numstat | gawk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2} END { printf "added lines: %s, removed lines : %s, now lines: %s, total lines: %s\n",add,subs,loc,total }';git log --format='%aN' | sort -u | while read name; do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2 } END { printf "added lines: %s, removed lines: %s, now lines: %s, total lines: %s\n", add, subs, loc, total }' -; done

### 单独统计每个人和所有的增删行数

单独统计每个人

> echo -en "\n"; git log --format='%aN' | sort -u | while read name;do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2 } END { printf "added lines: %s, removed lines: %s, now lines: %s, total lines: %s\n", add, subs, loc, total }' -; done;

每个人和所有

> echo -en "\n"; git log --format='%aN' | sort -u | while read name;do echo -en "$name\t"; git log --author="$name" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2 } END { printf "added lines: %s, removed lines: %s, now lines: %s, total lines: %s\n", add, subs, loc, total }' -; done;git log --pretty=tformat: --numstat | gawk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2} END { printf "\nadded lines: %s, removed lines : %s, now lines: %s, total lines: %s\n",add,subs,loc,total }'

### 统计当前用户的增删行数

> git log --author="$(git config --get user.name)" --pretty=tformat: --numstat | gawk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2} END { printf "added lines: %s, removed lines : %s, now lines: %s, total lines: %s\n",add,subs,loc,total }'

### 指定用户名增删行数

> git log --author="your_name_here" --pretty=tformat: --numstat | awk '{ add += $1; subs += $2; loc += $1 - $2; total += $1 + $2 } END { printf "added lines: %s, removed lines: %s, now lines: %s, total lines: %s\n", add, subs, loc, total }'

### 扫描 Log 统计增删行数

> git log --shortstat --pretty="%cE" | sed 's/\(.*\)@.*/\1/' | grep -v "^$" | awk 'BEGIN { line=""; } !/^ / { if (line=="" || !match(line, $0)) {line = $0 "," line }} /^ / { print line " # " $0; line=""}' | sort | sed -E 's/# //;s/ files? changed,//;s/([0-9]+) ([0-9]+ deletion)/\1 0 insertions\(+\), \2/;s/\(\+\)$/\(\+\), 0 deletions\(-\)/;s/insertions?\(\+\), //;s/ deletions?\(-\)//' | awk 'BEGIN {name=""; files=0; insertions=0; deletions=0;} {if ($1 != name && name != "") { print name ": " files " files changed, " insertions " insertions(+), " deletions " deletions(-), " insertions-deletions " net"; files=0; insertions=0; deletions=0; name=$1; } name=$1; files+=$2; insertions+=$3; deletions+=$4} END {print name ": " files " files changed, " insertions " insertions(+), " deletions " deletions(-), " insertions-deletions " net";}'

两个人同时修改，可能是 merge 操作，没有深究

### 仓库提交者排名前 5（如果看全部，去掉 head 管道即可）：

> git log --pretty='%aN' | sort | uniq -c | sort -k1 -n -r | head -n 5

### 仓库提交者（邮箱）排名前5

这个统计可能不会太准，因为很多人有不同的邮箱，但会使用相同的名字

git log --pretty=format:%ae | gawk -- '{ ++c[$0]; } END { for(cc in c) printf "%5d %s\n",c[cc],cc; }' | sort -u -n -r | head -n 5

### 贡献人数统计

git log --pretty='%aN' | sort -u | wc -l

### 提交次数统计

git log --oneline | wc -l 

## 14 http不用输入用户名密码

```
git remote set-url origin git+ssh://git@github.com/Gerry-Lee/Study.git
```

## 15 编辑配置项

1.查看Git所有配置

git config --list

2.删除全局配置项

(1)终端执行命令：

git config --global --unset user.name

(2)编辑配置文件：

git config --global --edit

# 16 