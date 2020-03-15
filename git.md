#关于git：
##	Git：分布式版本控制仓库。写在最前面的话
###		1、	Git是基于修改的版本控制,最小修改单元是修改,精确到行,而不是文件。
###		2、	一切都是基于指针操作的。
####			Does not ignore HEAD forever:
			头指针head指向版本和提交。Checkout基于head操作,表示移动head位置。操作head在很多情况下可等价于操作指向的分支和提交。head指向branch和commit,branch基于commit管理。不同分支的管理依赖于不同的commit。
			Git中不仅仅head有引用标识符的概念,类似于指针或者引用。Head、head~n、branch都可以当作引用在命令checkout/log/show/reset等价于指向的commitID：
				$ git log origin/master -3
				$ git reset -hard origin/taizhou
				$ git checkout head~1
			Head-n 和head~n的区别：
				Head~n会移动head指针,等价于指向的commit引用
				Head-n目前只在log中遇到过,表示显示最近的几个log。
##	初始化
		git config
###		设置缓存大小
			git config --global https.postBuffer 524288000
			git config --global http.postBuffer 524288000

###		查看用户名和邮箱地址：
			$ git config user.name
			$ git config user.email
###		修改用户名和邮箱地址
			$git config --global user.name  "name"
			$git config --global user.email  "…com"
###		Git remote
			Git init 初始化本地仓库
####		查看所有remote
				git remote -v
					origin /Users/lcjun/git/repo (fetch)
					origin /Users/lcjun/git/repo (push)
####		添加remote
				git remote add origin /Users/lcjun/git/repo.git
####		删除remote
				git remote rm origin
				git remote remove origin
##	git 三部曲
###	Git add 
		Git add将修改加入暂存区
		注意：一个文件可以分块加入暂存区
###	Git commit 
		git commit -m "line2" -a = Git add * & git commit -m "line2"
		check/fetch/reset/rebase
###	Checkout
		检出->
####	Checkout 三种用法：
		1、切换分支
			Git checkout branchName
			git checkout -b branchName
			check -b ->
				创建新分支：			git branch branchName
				切换到新分支：		git checkout branchName

		2、切换commit版本
			Git checkout commitID
			$ git checkout head~1
				Previous HEAD position was 908acf5a fix rowkey
				HEAD is now at 93cd4366 add log debug

		3、放弃暂存区修订(无法撤销,你的修改将丢失)
			Git checkout -- file1
			Git checkout .  撤销所有暂存区修改

	Checkout指向head操作,切换head到本地repository版本。Checkout可以切换本地分支和版本(版本是默认指向最新版本,因为分支是基于版本的,所以commitID区分分支版本)。
##	Git Fetch
	获取origin develop引用到本地
	$git fetch origin develop
## 在本地新建一个temp分支,并将远程origin仓库的master分支代码下载到本地temp分支；
	$ git fetch origin master:temp
## 比较本地代码与刚刚从远程下载下来的代码的区别；
	$ git diff temp
## 合并temp分支到本地的master分支;
	$ git merge temp
###		Git pull = git fetch  &&  git merge
Fetch获取引用线上remote origin引用,只获取,不会切换状态。git fetch命用来查看其他人的进程,因为它取回的代码对你本地的开发代码没有影响。采用此种方法建立的本地分支不会和远程分支建立映射关系。
##	Git reset
###Git reset 的三种用法
	1、回退指定版本(可选择是否清空暂存区)
		git reset –-soft commitID
			回退版本,不清空暂存区,将已提交的内容恢复到暂存区,不影响原来本地的文件(未提交的也不受影响) 
		git reset --hard commitID 
			回退一个版本,清空暂存区,将已提交的内容的版本恢复到本地,本地的文件也将被恢复的版本替换

	2、放弃提交到暂存区的修改(将修改移出暂存区)
		修改不会发生改变,状态变更到add之前(绿色变为红色)
			现在暂存区有三个文件
				A  file1
				A  file2
				A  file3
			将file1移出暂存区
 				git reset file1 
				git status -s 
				A  file2
				A  file3
				?? file1
			File1再次进入未跟踪状态
			不指定参数Git reset 会将暂存区所有文件置入未跟踪状态
				git reset
				git status -s
				?? file1 ?? file2 ?? file3 

			3、可以撤销git rebase操作
				可以配合git reflog使用,git reflog会列出所有操作,所有操作均可被reset撤销！
####			Git reflog
					会显示出你所有的操作(不一定是commit,多个操作可能会对应一个commitID,比如来回切换branch, head会在commitID之间移动)
					$ git reflog head -2
						908acf5a (HEAD -> taizhou, tag: v1.0, origin/taizhou) head@{0}: reset: moving to origin/taizhou
						c3544660 head@{1}: reset: moving to head~2
					git reset --hard HEAD@{4} 

##	Git rebase
	1、合并分支
		可以说一下和merge的区别。使用merge合并分支的时候会将merge记录显示在log之中,污染了本身的 commit 记录(如果开发者有代码洁癖是不能忍受的),想要保持一份干净的 commit,可以使用git rebase命令:
###		Git rebase branchName
			这里建议一下操作需谨慎,如果分支版本相差过大的话会引起很多复杂的问题。

	2、合并commit
		$git rebase -i HEAD~n 表示想要合并head往前的几个分支(也可以输入commitID)
			p--pick = use commit
			r--reword = use commit, but edit the commit message
			e--edit = use commit, but stop for amending
			s--squash = use commit, but meld into previous commit
			f--fixup = like “squash”, but discard this commit’s log message
			x--exec = run command (the rest of the line) using shell
			d--drop = remove commit
		我们平时用到的基本只有s(merge commit)和p(use commit)
		
		git rebase --edit-todo 如果vi窗口异常退出,继续编辑操作
		git rebase –abort 在任何时候,可以使用abort操作来放弃rebase操作
		git rebase –continue 解决完冲突之后执行continue完成rebase

##	Git log/show/diff
###		git log
			Git log 有一些很有趣的命令,能看到不同的信息,当然最全面的还是无参数命令
			git log --abbrev-commit --pretty=oneline
			git log fileName
			git log head -2 fileName. head 可以省略

###	Git show branchName/commitID/head~n比较当前版本和指定版本的不同

###Git diff
	$git diff head~1 head filename
	git rev-parse --abbrev-ref HEAD
	查看head指向的引用----这个命令不一定指向分支,会指向head指针的引用,如果此时你执行checkout命令将head移动到指定commitID,则此时的引用不会面向某个分支。如下图所示
###	Git 强制origin覆盖本地
			Git origin覆盖本地仓库操作:
			Git fetch –-all 				获取所有origin引用到本地
			Git reset –-hard origin/master 	回退到origin/master所在版本
			Git pull 						获取origin master代码到本地仓库

		其实这种操作有很多种思路:比如可以先将本地仓库回退到线上之前的版本,然后再pull origin仓库(两种思路异曲同工):
			Git reset –hard head~1
			Git pull origin barnchName
		Git 强制提交到远程仓库
			当本地版本和origin仓库版本发生冲突,可以在origin解决冲突问题,也可以强行覆盖origin(权限允许)
			git push -u origin master -f
##	将线上分支与本地仓库分支建立映射
		git branch --set-upstream-to=origin/branch branchName
##	.gitignore
	不起作用的话
		git rm -r --cached .
		git add .
		git commit -m 'update .gitignore'   

