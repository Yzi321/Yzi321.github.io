---
layout:     post
title:      vs+qt修改本地ico（已有一份ico）
subtitle:   vs+qt修改生成exe的ico图标
date:       2024-10-28
author:     Yzi321
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - qt
---


（此方法只修改文件夹里exe的ico，运行时的ico需要从ui文件中修改）

#### 1、有ICO文件

#### 2、右键项目，添加ico资源

![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028233753308-618990586.png)

#### 3、选择ico文件，此时项目列表中会出现ico文件

![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028233824788-1273774922.png)

#### 4、清楚项目，重新生成项目

#### 5、右键rc文件，查看代码

![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028233852113-1074103776.png)
![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028233904461-335355465.png)

#### 6、此时会出现两个ico

![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028233939681-779409767.png)

#### 7、关闭rc文件，找到resource.h文件，打开

#### 8、找到IDI_ICON1和IDI_ICON2

![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028233925774-1597514222.png)

#### 9、选择需要改为的ico文件，将后面的数字修改全局最小，即<font style="color:rgb(68, 68, 68);">比其他IDI_***的号码都小就即可</font>

#### <font style="color:rgb(68, 68, 68);">10、清理项目</font>

#### <font style="color:rgb(68, 68, 68);">11、重启电脑</font>

（此方法只修改文件的ico，运行时的ico需要从ui文件中修改）



#### 相关链接：[https://www.cnblogs.com/zzzsj/p/15721858.html](https://www.cnblogs.com/zzzsj/p/15721858.html)
