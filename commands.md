# find which package provide specify file

```
dpkg-query -S XXX
```

# install go develop environment by vscode

```
~ » go env -w GO111MODULE=on //打开后，vscode 跳转功能不可用
~ » go env -w GOPROXY=https://goproxy.io,direct
```

```
开启coredump
ulimit -c unlimited
sudo echo "1" > /proc/sys/kernel/core_uses_pid
sudo echo "/corefile/core-%e-%p-%t" > /proc/sys/kernel/core_pattern
sudo mkdir /corefile
sudo chmod -R 777 /corefile
```

How to enable asan version without recompile grpc

```
如果仅仅想测试iSulad的asan而不是grpc，则需要打上社区补丁
https://github.com/grpc/grpc/pull/22325

然后在iSulad的CMakefileList.txt中，定义该宏

```
