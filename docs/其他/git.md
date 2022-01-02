## tag

1. 添加

```shell
# 本地添加
git tag v1.0.0
# 推送tag
git push --tags
```

2. 删除

```shell
# 删除本地
git tag -d v1.0.0
# 删除远程
git push origin :refs/tags/v1.0.0
```
