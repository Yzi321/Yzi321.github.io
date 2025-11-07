---
layout:     post
title:      qt的无边框窗口支持拖拽、Aero Snap、窗口阴影等特性
subtitle:   qt的无边框窗口支持拖拽、Aero Snap、窗口阴影等特性
date:       2024-10-29
author:     Yzi321
header-img: img/post-bg-cook.jpg
catalog: true
tags:
    - qt
---


<font style="color:rgb(0, 0, 0);">环境：Desktop Qt 6.7.2 MSVC2019 64bit</font>

<font style="color:rgb(0, 0, 0);">需要的库：`dwmapi.lib`、`user32.lib`

<font style="color:rgb(0, 0, 0);">需要头文件：`<dwmapi.h>`、`<windowsx.h>`

> <font style="color:rgb(85, 85, 85);">只显示重要代码</font>
>

### 1、去除原边框、<font style="color:rgb(0, 0, 0);">加上阴影、Aero Snap以及其他动画特效</font>
#### （1）头文件
```cpp
#include "Windows.h"
#include "uxtheme.h"
#include "dwmapi.h"
#include "titlebar.h"//自定义类
```

#### （2）去除标题、原边框
初始化，去除边框

```cpp
void MainWindow::init()
{
    setWindowFlags(Qt::Window | Qt::FramelessWindowHint);

#ifdef Q_OS_WIN
    HWND hwnd = reinterpret_cast<HWND>(this->winId());

    const LONG style = ( WS_POPUP | WS_CAPTION | WS_SYSMENU | WS_MINIMIZEBOX | WS_MAXIMIZEBOX | WS_THICKFRAME | WS_CLIPCHILDREN );
    SetWindowLongPtr(hwnd, GWL_STYLE, style);

    const MARGINS shadow = {1, 1, 1, 1};
    DwmExtendFrameIntoClientArea(hwnd, &shadow);

    SetWindowPos(hwnd, 0, 0, 0, 0, 0, SWP_FRAMECHANGED | SWP_NOMOVE | SWP_NOSIZE);
#endif
    // 标题拖动、双击事件
    MyTitleBar *title = new MyTitleBar(this);
    qobject_cast<QBoxLayout *>(ui->centralwidget->layout())->insertWidget(0, title);
}

```

同时在最大化时，增加了边界，可以自行删除  
若使用qt5，重载 `bool nativeEvent(const QByteArray &eventType, void *message, long *result);`

```cpp
bool MainWindow::nativeEvent(const QByteArray& eventType, void* message, qintptr* result)
{
    MSG* msg = (MSG*)message;
    switch (msg->message)
    {
    case WM_NCCALCSIZE:
    {
        // this kills the window frame and title bar we added with WS_THICKFRAME and WS_CAPTION
        *result = 0;
        return true;
    }
    // 若full时边框不合适，可以去除此case
    case WM_GETMINMAXINFO:
    {
        if (::IsZoomed(msg->hwnd)) {
            // 最大化时会超出屏幕，所以填充边框间距
            RECT frame = { 0, 0, 0, 0 };
            AdjustWindowRectEx(&frame, WS_OVERLAPPEDWINDOW, FALSE, 0);
            frame.left = abs(frame.left);
            frame.top = abs(frame.bottom);
            this->setContentsMargins(frame.left, frame.top, frame.right, frame.bottom);
        }
        else {
            this->setContentsMargins(0, 0, 0, 0);
        }

        *result = ::DefWindowProc(msg->hwnd, msg->message, msg->wParam, msg->lParam);
        return true;
    }
    break;
    default:
        return QMainWindow::nativeEvent(eventType, message, result);
    }
}
```

#### （3）支持手动修改窗口
<font style="color:rgb(0, 0, 0);">需要实现一个</font>`QAbstractNativeEventFilter` 类，内容如下：</font>

##### <font style="color:rgb(0, 0, 0);">头文件</font>
```cpp
#include <QAbstractNativeEventFilter>
#include <QWidget>
#include "Windows.h"
#define GET_X_LPARAM(lp) ((int)(short)LOWORD(lp))
#define GET_Y_LPARAM(lp) ((int)(short)HIWORD(lp))

class NativeEventFilter : public QAbstractNativeEventFilter
{
public:
    virtual bool nativeEventFilter(const QByteArray &eventType, void *message, qintptr *result)Q_DECL_OVERRIDE;
};
```

##### <font style="color:rgb(0, 0, 0);">cpp文件</font>
```cpp
bool NativeEventFilter::nativeEventFilter(const QByteArray &eventType, void *message, qintptr *result)

{
#ifdef Q_OS_WIN
    if (eventType != "windows_generic_MSG")
        return false;

    MSG* msg = static_cast<MSG*>(message);
    QWidget* widget = QWidget::find(reinterpret_cast<WId>(msg->hwnd));
    if (!widget)
        return false;

    switch (msg->message) {
    case WM_NCHITTEST: {
        const LONG borderWidth = 9;
        RECT winrect;
        GetWindowRect(msg->hwnd, &winrect);
        long x = GET_X_LPARAM(msg->lParam);
        long y = GET_Y_LPARAM(msg->lParam);

        // bottom left
        if (x >= winrect.left && x < winrect.left + borderWidth &&
            y < winrect.bottom && y >= winrect.bottom - borderWidth)
        {
            *result = HTBOTTOMLEFT;
            return true;
        }

        // bottom right
        if (x < winrect.right && x >= winrect.right - borderWidth &&
            y < winrect.bottom && y >= winrect.bottom - borderWidth)
        {
            *result = HTBOTTOMRIGHT;
            return true;
        }

        // top left
        if (x >= winrect.left && x < winrect.left + borderWidth &&
            y >= winrect.top && y < winrect.top + borderWidth)
        {
            *result = HTTOPLEFT;
            return true;
        }

        // top right
        if (x < winrect.right && x >= winrect.right - borderWidth &&
            y >= winrect.top && y < winrect.top + borderWidth)
        {
            *result = HTTOPRIGHT;
            return true;
        }

        // left
        if (x >= winrect.left && x < winrect.left + borderWidth)
        {
            *result = HTLEFT;
            return true;
        }

        // right
        if (x < winrect.right && x >= winrect.right - borderWidth)
        {
            *result = HTRIGHT;
            return true;
        }

        // bottom
        if (y < winrect.bottom && y >= winrect.bottom - borderWidth)
        {
            *result = HTBOTTOM;
            return true;
        }

        // top
        if (y >= winrect.top && y < winrect.top + borderWidth)
        {
            *result = HTTOP;
            return true;
        }

        return false;
    }
    default:
        break;
    }

    return false;
#else
    return false;
#endif
};

```

##### 应用
<font style="color:rgb(0, 0, 0);">然后在窗口创建之前，使用</font>`QApplication::installNativeEventFilter`<font style="color:rgb(0, 0, 0);"> 方法把监听器注册给主程序。</font>

```cpp
int main(int argc, char *argv[])
{
    NativeEventFilter f;
    QApplication a(argc, argv);
    a.installNativeEventFilter(&f);//支持手动修改窗口大小
    MainWindow w;
    w.show();
    return a.exec();
}
```

<font style="color:rgb(0, 0, 0);"></font>

### <font style="color:rgb(0, 0, 0);">2、自定义标题栏</font>
+ 需要重载`QWidget::mousePressEvent`<font style="color:rgb(0, 0, 0);"> 方法
+ <font style="color:rgb(0, 0, 0);">保存 </font><font style="color:rgb(85, 85, 85);background-color:rgb(238, 238, 238);">Window</font><font style="color:rgb(0, 0, 0);"> 句柄操作原窗口</font>

```cpp
class MyTitleBar : public QFrame  {
    Q_OBJECT
public:
    explicit MyTitleBar(QWidget *parent = nullptr);

protected:
    void mousePressEvent(QMouseEvent* ev);


private:
    QWidget *Window = nullptr; // 保存主窗口的指针

    QVBoxLayout *verticalLayout;
    QHBoxLayout *horizontalLayout;
    QSpacerItem *horizontalSpacer;
    QPushButton *pushButton_min;
    QPushButton *pushButton_normal;
    QPushButton *pushButton_max;
    QPushButton *pushButton_full;
    QSpacerItem *horizontalSpacer_6;
    QPushButton *pushButton_close;
};
```

##### 创建ui
```cpp

MyTitleBar::MyTitleBar(QWidget *parent): QFrame (parent), Window(parent)
{
    // 边缘贴合
    parent->setContentsMargins(0, 0, 0, 0);

    setObjectName("title");
    setMaximumHeight(50);
    this->setStyleSheet("#title{background-color: rgb(255, 255, 255);}");
    verticalLayout = new QVBoxLayout(this);
    verticalLayout->setObjectName("verticalLayout");
    verticalLayout->setContentsMargins(0, 0, 0, 0);
    horizontalLayout = new QHBoxLayout();
    horizontalLayout->setObjectName("horizontalLayout");
    horizontalSpacer = new QSpacerItem(40, 20, QSizePolicy::Policy::Expanding, QSizePolicy::Policy::Minimum);

    horizontalLayout->addItem(horizontalSpacer);

    pushButton_min = new QPushButton(this);
    pushButton_min->setObjectName("pushButton_min");

    horizontalLayout->addWidget(pushButton_min);

    pushButton_normal = new QPushButton(this);
    pushButton_normal->setObjectName("pushButton_normal");

    horizontalLayout->addWidget(pushButton_normal);

    pushButton_max = new QPushButton(this);
    pushButton_max->setObjectName("pushButton_max");

    horizontalLayout->addWidget(pushButton_max);

    pushButton_full = new QPushButton(this);
    pushButton_full->setObjectName("pushButton_full");

    horizontalLayout->addWidget(pushButton_full);

    horizontalSpacer_6 = new QSpacerItem(40, 20, QSizePolicy::Policy::Expanding, QSizePolicy::Policy::Minimum);

    horizontalLayout->addItem(horizontalSpacer_6);

    pushButton_close = new QPushButton(this);
    pushButton_close->setObjectName("pushButton_close");

    horizontalLayout->addWidget(pushButton_close);

    horizontalLayout->setStretch(0, 1);

    verticalLayout->addLayout(horizontalLayout);

    pushButton_min->setText(QCoreApplication::translate("MainWindow", "MIN", nullptr));
    pushButton_normal->setText(QCoreApplication::translate("MainWindow", "NORMAL", nullptr));
    pushButton_max->setText(QCoreApplication::translate("MainWindow", "MAX", nullptr));
    pushButton_full->setText(QCoreApplication::translate("MainWindow", "FULL", nullptr));
    pushButton_close->setText(QCoreApplication::translate("MainWindow", "CLOSE", nullptr));
}
```

##### 连接主窗口的变化
<font style="color:rgb(0, 0, 0);">最大化和关闭按扭，正常调用</font>`QWidget::showMaximized()`<font style="color:rgb(0, 0, 0);">和</font>`QWidget::close()`<font style="color:rgb(0, 0, 0);"> 等Qt自带方法即可。</font>

```cpp
    connect(pushButton_min, &QPushButton::clicked, Window, &QWidget::showMinimized);
    connect(pushButton_close, &QPushButton::clicked, Window, &QWidget::close);
    connect(pushButton_normal, &QPushButton::clicked, Window, &QWidget::showNormal);
    connect(pushButton_full, &QPushButton::clicked, Window, &QWidget::showFullScreen);
    connect(pushButton_max, &QPushButton::clicked, Window, &QWidget::showMaximized);
```

##### <font style="color:rgb(0, 0, 0);">重截</font>`QWidget::mousePressEvent`<font style="color:rgb(0, 0, 0);"> 方法</font>
```cpp
#include "Windows.h"
void MyTitleBar::mousePressEvent(QMouseEvent* ev)
{
    QWidget::mousePressEvent(ev);

    if (Window == nullptr) return;
    if (!ev->isAccepted()) {
        if (ev->type() == QEvent::MouseButtonDblClick) {
            // Toggle maximize/restore on double-click
            if (Window->isMaximized()) {
                Window->showNormal(); // Restore
            }
            else {
                Window->showMaximized(); // Maximize
            }
            return; // Prevent further processing
        }
#ifdef Q_OS_WIN
        ReleaseCapture();
        SendMessage(reinterpret_cast<HWND>(Window->winId()), WM_SYSCOMMAND, SC_MOVE + HTCAPTION, 0);
#endif
    }
}
```

##### 添加
```cpp
    // 标题拖动、双击事件
    MyTitleBar *title = new MyTitleBar(this);
    qobject_cast<QBoxLayout *>(ui->centralwidget->layout())->insertWidget(0, title);
```

### <font style="color:rgb(0, 0, 0);">3、最终实现效果</font>

![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241029022643160-1021096087.gif)
![](https://img2024.cnblogs.com/blog/3546906/202410/3546906-20241029022653032-456573123.gif)

> <font style="color:rgb(0, 0, 0);">参考链接：</font>[https://github.com/deimos1877/BorderlessWindow](https://github.com/deimos1877/BorderlessWindow)