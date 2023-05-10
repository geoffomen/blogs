# 一般原则
- 分支类型分为临时分支和持久分支。临时分支包括：功能开发分支`feature/<xxx>`、热修复分支`hotfix/<xxx>`；持久分支包括：主干分支`master`、对外分发分支（例如APP）`release/<xxx>`、持续部署分支（例如后台服务）`prod`。
- 代码的变化必须由“上游”向“下游”发展：feature、hotfix是master的上游，master是release、prod的上游。
- 持久分支不会删除，feature分支合并到master后立即删除，hotfix分支合并到缺陷所在分支后删除。
- feature分支在开发人员的开发环境中部署测试；master分支在测试环境中部署测试；release分支和prod分支在验收环境中部署测试，验收通过可立即发布。
- master分支是随时可部署的，允许有未发现的缺陷，但软件整体功能是正常的；feature分支必须有对应的需求；hotfix分支必须有对应的缺陷。禁止创建其它类型分支。
- 对外的每一次应用分发就创建一个release分支，分支命名格式为: `release/<xxx>`
- feature分支的建立必须从master分支创建，参与该功能开发的人员的代码都提交至该功能分支，功能开发完成后（开发人员在自己环境中测试通过），应该使用`git rebase`把多个 commit 合并成一个，然后合并至master。
- hotfix分支的建立从发现缺陷的分支创建，修复完成后（开发人员在自己环境中测试通过），合并到master在测试环境中进一步测试。如果发现缺陷的分支还在下游，master测试通过后需要继续应用到目标下游分支并进行测试。
- release分支和prod分支的建立必须从master分支创建，通过`git cherry-pick`从master中选择该次发布的commit进入。
- 开发人员负责开发环境和测试环境中的测试，质量保证团队人员负责验收环境的测试和生产环境的最终验收。
- 多人同时开发时，为避免过多的代码合并冲突，建议本地开发分支尽可能与远程分支保持同步：
    ```
	git fetch <remote> <master or feature>
	git rebase <master or feature>
    ```

# 特殊情况的处理
### 紧急需求或生产环境紧急热修复的处理
经过批准，可以从生产环境对应的分支创建临时分支，开发完成后直接提交至生产环境对应的分支，然后用生产环境对应的分支依次部署测试环境和验收环境进行测试，此时测试环境和验收环境必须为此让路。测试流程与正常情况相同。临时分支代码发布生产环境后，按正常流程合并至master分支。

### 未列举情况的处理
原则上经过批准可以按**紧急需求或生产环境紧急热修复的处理**的流程处理。

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
    git fetch <remote> feature/a-sample-feature
	git rebase <remote>/feature/a-sample-feature
    # 若有冲突，则须要解决冲突
	git push <remote> feature/a-sample-feature  
	```
 - 当功能完成并且自测通过后，将功能分支合并到master：
	 ```sh
     git checkout master
	 git pull <remote> master
     git checkout feature/a-sample-feature
	 git rebase -i master
     git checkout master
	 git merge --no-ff feature/a-sample-feature
	 ```
 - 删除当前分支：
	```sh
    git checkout master
	git branch -d feature/a-sample-feature
	```

### 持续部署
- 从master选择需要发布的功能合并到prod: 
    非初次创建prod
	```sh
	git checkout prod 
	git pull <remote> prod
	git cherry-pick <master上的commitId>  # 若有多个提交需要发布时，则多次执行该命令
	```
    初次创建prod
    ```sh
    git checkout --orphan=prod
    git cherry-pick <master上的commitId>  # 若有多个提交需要发布时，则多次执行该命令
    ```
	
### 对外发布
- 从master选择需要发布的功能合并到`release/<ver>`: 
    非初次发布时：
	```sh
	git checkout <上一次发布的分支>
	git pull <remote> <上一次发布的分支>
	git checkout -b release/<new-ver>
	git cherry-pick <master上的commitId>  # 若有多个提交需要发布时，则多次执行该命令
	```
    初次发布时：
    ```sh
    git checkout --orphan=release/<new-ver>
    git cherry-pick <master上的commitId> # 若有多个提交需要发布时，则多次执行该命令
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
- 修复代码上线后应用到master
	```sh
	git checkout master
	git pull <remote> master
	git cherry-pick <commitId> 
	```
- 修复代码应用完成后，删除热修复分支：
	```sh
	git branch -d hotfix/<bugId或者日期date>
	```

# 参考

[Introduction to GitLab Flow _ GitLab (8_9_2021 4_55_01 PM).html](https://docs.gitlab.com/ee/topics/gitlab_flow.html)
