---
title: 删除启动台 (Launchpad) 中无法去除的图标
date: 2019-10-13 15:22:42
tags:
- mac
- tips
categories:
- mac
- tips
---

应用删除后图标还在启动台？打开终端！

1. 找到 Launchpad 存放数据的路径
``` bash
find 2>/dev/null /private/var/folders -name com.apple.dock.launchpad
```

2. 进入输出目录的db子目录
``` bash
cd /private/var/folders/**/com.apple.dock.launchpad/db
```

3. 删除数据库中title为@@@的数据，将@@@替换为应用名称（区分大小写）&&重启Dock
``` bash
sqlite3 db "delete from apps where title='@@@';"&&killall Dock
```
