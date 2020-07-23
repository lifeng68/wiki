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
