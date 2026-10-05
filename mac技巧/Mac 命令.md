# Mac 命令

信任任何来源
```
sudo spctl —master-disable
```

修改Host 后 刷新DNS缓存
```
sudo killall -HUP mDNSResponder
```

移动硬盘无法被识别的问题
```
~sudo~ pkill -f fsck 
```

