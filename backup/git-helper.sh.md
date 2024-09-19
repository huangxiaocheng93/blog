实现了分支的合并

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