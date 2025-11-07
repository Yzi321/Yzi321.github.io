---
layout:     post
title:      qt标题，解决title的png图片scaled后显示有明显锯齿
subtitle:   优化qt下自定义TitleBar的左上角ICO的显示效果
date:       2024-10-28
author:     Yzi321
header-img: img/post-bg-keybord.jpg
catalog: true
tags:
    - qt
---

### 一、通用方法（使用Qlabel）
```cpp
// 添加窗口图标
iconLabel = new QLabel(this);
QPixmap iconPixmap(":/ico.png"); // 替换成你的图标文件路径
iconLabel->setPixmap(iconPixmap.scaled(125, 35, Qt::KeepAspectRatio, Qt::SmoothTransformation));
iconLayout->addWidget(iconLabel);

```

此时图片会有锯齿感

###### 原因：
```cpp
iconPixmap.scaled(125, 35, Qt::KeepAspectRatio, Qt::SmoothTransformation)
```

此代码的图片是正常的，但是

```cpp
iconLabel->setPixmap();
```

调用此函数后，显示在QLabel中的实际大小会大于原来125*35的大小，所以会有锯齿感

### 二、解决办法
##### 1、使用QToolButton
```cpp

QImage iconImage(PNG_PATH); // 替换成你的图标文件路径

// 缩放
float radio = 0.3;
float radioButton = 0.6;
int width = iconImage.width() * radio;
int height = iconImage.height() * radio;

toolButton = new QToolButton(this);
toolButton->setMinimumSize(width * radioButton, height * radioButton);
toolButton->setStyleSheet(".QToolButton{padding-left: -20px;padding-top: 7px;background: transparent;border: 0px;}.QToolButton:hover{background-color: transparent;}.QToolButton:pressed{background-color: transparent;}");

QIcon icon(std::move(QImage2QPixmap(iconImage.scaled(width, height, Qt::KeepAspectRatio, Qt::SmoothTransformation))));
toolButton->setIcon(icon);
toolButton->setIconSize(QSize(width * radioButton, height * radioButton));

// 设置按钮的其他属性（可选）
toolButton->setText("");

// 设置按钮自动提升，使其在不可点击时呈现为灰显
toolButton->setAutoRaise(true);

iconLayout->insertWidget(0, toolButton);

```



1、通过调整 **radio** 的调整本地png的缩放大小

2、通过调整 **radioButton** 的值来调整toolButton的大小

3、通过调整setStyleSheet中的 **padding-left: -20px;padding-top: 7px**; 调整位置

### 效果对比
![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241028230258290-662004227.png)
