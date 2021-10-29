# 一般原则
- 分支类型分为临时分支和持久分支。临时分支包括：功能开发分支`feature/<xxx>`、热修复分支`hotfix/<xxx>`；持久分支包括：主干分支`master`、对外分发分支（例如APP）`release/<xxx>`、持续部署分支（例如后台服务）`prod`。
- 代码的变化必须由“上游”向“下游”发展：功能分支、热修复分支是master的上游，master是对外分发分支、持续部署分支的上游。
- 持久分支不会删除，功能分支合并到master后立即删除，热修复分支合并到缺陷所在分支后删除。
- 功能分支在开发人员的开发环境中部署测试；master分支在测试环境中部署测试；分发分支和持续部署分支在验收环境中部署测试，验收通过可立即发布生产。
- master分支是随时可部署的，允许有未发现的缺陷，但软件整体功能是正常的；功能分支必须有对应的需求；热修复分支必须有对应的缺陷。禁止创建其它类型分支。
- 对外分发分支每一次分布就创建一个分支，分支命名格式为: `release/<xxx>`
- 功能分支的建立必须从master分支创建，参与该功能开发的人员的代码都提交至该功能分支，功能开发完成后（开发人员在自己环境中测试通过），应该使用`git rebase`把多个 commit 合并成一个，然后合并至master。
- 热修复分支的建立从发现缺陷的分支创建，修复完成后（开发人员在自己环境中测试通过），使用`git cherry-pick`合并到master，在测试环境中进一步测试。如果发现缺陷的分支还在下游，master测试通过后需要继续`git cherry-pick`到目标下游分支并进行测试。若在向下游分支`cherry-pick`的过程中，在某一分支测试不通过，需要对不通过的分支创建新的热修复分支，如此重复直至可上线，这会创建多个热修复分支。
- 分发分支和持续部署分支的建立必须从master分支创建，通过`git cherry-pick`从master中选择该次发布的功能进入。
- 开发人员负责开发环境和测试环境中的测试，质量保证团队人员负责验收环境的测试和生产环境的最终验收。

# 特殊情况的处理
### 紧急需求或生产环境热修复的处理
经过批准，可以从生产环境对应的分支创建临时分支，开发完成后直接提交至生产环境对应的分支，然后用生产环境对应的分支依次部署测试环境和验收环境进行测试，此时测试环境和验收环境必须此让路。测试流程与正常情况相同。临时分支代码发布生产环境后，按正常流程合并至master分支及master下游分支。

### 未列举情况的处理
原则上经过批准可以按**紧急需求或生产环境热修复的处理**的流程处理。

# 示例
###  初始化仓库
 - 创建主仓库：
	```sh
	git init -b master
	```

 ### 开发功能
 - 从master分支切出一个新的功能分支：
	```sh
	git checkout -b feature/a-sample-feature master
	```
 - 开发过程中，把变更提交至功能分支：
	```sh
	git add .
	git commit -m 'a good commit message.'
	git push <remote> feature/a-sample-feature  
	```
 - 当功能完成并且自测通过后，将功能分支合并到master：
	 ```sh
	 git rebase -i <从master分叉处的commitId>
	 git checkout master
	 git pull <remote> master
	 git merge --no-ff feature/a-sample-feature
	 ```
 - 删除当前分支：
	```sh
	git branch -d feature/a-sample-feature
	```

### 持续部署
- 从master选择需要发布的功能合并到prod: 
	```sh
	git checkout prod # 初次创建prod时执行 git checkout -b prod <master第一个commitId>
	git pull <remote> prod
	git cherry-pick <master上的commitId>  # 若有多个提交需要发布时，则多次执行该命令
	```
	
### 对外发布
- 从master选择需要发布的功能合并到`release/<ver>`: 
	```sh
	git checkout <上一次发布的分支> # 初次创建release时执行 git checkout <master第一个commitId>
	git pull <remote> <上一次发布的分支> # 初次创建时无须执行
	git checkout -b release/<ver>
	git cherry-pick <master上的commitId>  # 若有多个提交需要发布时，则多次执行该命令
	```
	
### 普通热修复
- 从发现缺陷的目标分支创建热修复分支：
	```sh 
	git checkout <target-branch>
	git pull <remote> <target-branch>
	git checkout -b hotfix/<bugId或者日期date>
	# 修复代码完成后提交
	git add .
	git commit -m 'a good message'
	```
- 修复完成并测试通过后cherry-pick到其它需要应用的分支，先从上游分支开始：
	```sh
	git checkout master
	git pull <remote> master
	git cherry-pick <commitId> 
	# master测试通过后，才继续应用到下游分支。
	# 在任意一分支不通过则继续按**普通热修复**流程创建新的热修复分支。
	git checkout <target-branch>
	git pull <remote> <target-branch>
	git cherry-pick <commit>
	```
- 修复代码应用完成后，删除热修复分支：
	```sh
	git branch -d hotfix/<bugId或者日期date>
	```

### 紧急需求或生产环境热修复 
- 从生产环境对应的分支创建临时分支：
	```sh
	git checkout <prod或者release/<ver>>
	git pull <remote> <prod或者release/<ver>>
	git checkout -b hotfix/<bugId或者日期date>
	# 修复代码完成后提交
	git add .
	git commit -m 'a good message'
	```
- 应用到生产环境对应的分支
	```sh
	git checkout <prod或者release/<ver>>
	git pull <remote> <prod或者release/<ver>>
	git cherry-pick <commitId> 
	```
- 修复代码上线后应用到master及master下游分支
	```sh
	git checkout master
	git pull <remote> master
	git cherry-pick <commitId> 
	# master测试通过后，才继续应用到下游分支。
	# 在任意一分支不通过则继续按**普通热修复**流程创建新的热修复分支。
	git checkout <target-branch>
	git pull <remote> <target-branch>
	git cherry-pick <commit>
	```
- 修复代码应用完成后，删除热修复分支：
	```sh
	git branch -d hotfix/<bugId或者日期date>
	```

# 参考

[Introduction to GitLab Flow _ GitLab (8_9_2021 4_55_01 PM).html](https://docs.gitlab.com/ee/topics/gitlab_flow.html)


[基于GitLab Flow 的开发_测试_运维统一上线流程 _ FinTx (8_18_2021 10_45_39 AM).html](http://www.fintx.org/20170705-dev-qa-ops-unified-flow-base-on-gitlab-flow.html)





本文以《署名-非商业性使用-相同方式共享 4.0 协议 (CC BY-NC-SA 4.0)》授权。
