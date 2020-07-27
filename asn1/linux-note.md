# linux

### sysctl
sysctl配置和现实linux运行时的内核参数，参数保存在/proc/sys目录

##### 常用参数

`-w`	临时改变参数，重启系统失效

```shell
sysctl -w net.ipv4.ip_forward=1
echo 1 > /proc/sys/net/ipv4/ip_forward
```

修改`/proc/sys`下对应的参数只能临时生效，重启后失效，但是对应参数文件修改后一经保存马上生效，无需执行额外命令

如需永久修改，需修改`/etc/sysctl.conf`文件，但需执行以下命令，重新加载系统参数才会生效

```shell
sysctl -p
```



`-a`	列出所有可用的内核参数和值