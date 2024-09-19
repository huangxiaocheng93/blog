日常工作中经常要合并分支，例如推送变更到开发或者测试分支。以推送feature分支到test分支为例：
- 推送feature分支到远程；
- 切换到test分支，更新test分支；
- 合并feature分支到当前（test）分支；
- 推送test分支；
- 切换回feature分支；
为了简化这个部分，所以弄了隔脚本，亲测macos下可用，其他平台未测试；
使用方法：
当前在feature分支：
git-help.sh push-to test

```bash
#!/bin/bash
operation=$1
if [ $operation == "push-to" ]; then
	target_branch=$2
	branch_name=`git symbolic-ref --short -q HEAD`
	echo "curr branch name is "$branch_name
	git pull
	git push
	git checkout $target_branch
	git pull
	echo -e "\nmerge "$branch_name" to "$target_branch
	log=`git merge $branch_name`
	echo $log
	git push
	git checkout $branch_name
fi
```