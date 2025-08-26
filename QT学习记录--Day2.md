---
title: QT
date: 2020-06-21 11:00:25
tags:
  - QT
keywords: QT
categories: 理论
---

# QT学习记录--Day2

> 2020/6/21 11点00分
>
> 今天加油希望搞完程序设计

## 1.Lambda表达式

* 见md文件夹下视频，讲解清晰。



## 2.QMainWindow

*  菜单栏  最多有一个

* 工具栏可以有多个

  ~~~C++
  #include "mainwindow.h"
  #include <QMenuBar>
  #include <QToolBar>
  #include <QDebug>
  #include <QPushButton>
  MainWindow::MainWindow(QWidget *parent)
      : QMainWindow(parent)
  {
      //设置窗口大小
      resize(800,600);
  
      //菜单栏   最多只能有一个！！
      //菜单栏创建
      QMenuBar *bar = menuBar();
      //将菜单栏放入窗口中
      setMenuBar(bar);
  
      //创建菜单
      QMenu *FileMenu = bar->addMenu("文件");
      QMenu *EditMenu = bar->addMenu("编辑");
  
      //创建菜单项（二级菜单）
      FileMenu->addAction("新建");
      //添加分割线
      FileMenu->addSeparator();
      FileMenu->addAction("打开");
  
      //工具栏  可以有多个
      //  this  是为了将工具栏添加到对象树中，简化内存管理
      //菜单栏不需要加  this  ，因为其默认自动加入对象树中
      QToolBar * toolbar = new QToolBar(this);
      //将工具栏添加到窗口中，默认在上方
      addToolBar(toolbar);
      //更改工具栏默认窗口位置,位置信息可在官方帮助文档中找到
      //addToolBar(Qt::LeftToolBarArea,toolbar);
  
      //设置工具栏只允许上下停靠
      //注意  区域之间分割用  |
      toolbar->setAllowedAreas(Qt::TopToolBarArea|Qt::BottomToolBarArea);
  
      //设置不允许工具栏浮动
      toolbar->setFloatable(false);
  
      //设置移动（工具栏能否拖动的  总开关 ）
      //toolbar->setMovable(false);
  
      //工具栏中设置内容
      //可以设置直接与菜单栏中内容相关联
      QAction *newAction = FileMenu->addAction("新建");
      toolbar->addAction(newAction);
      //添加分割线
      toolbar->addSeparator();
      //也可以单独设置
      toolbar->addAction("保存");
  
      //工具栏中添加控件（如按钮）
      QPushButton *btn = new QPushButton ("啦啦啦",this);
      toolbar->addWidget(btn);
  }
  
  MainWindow::~MainWindow()
  {
  }
  
  
  ~~~

  最终效果

  ![批注 2020-06-21 125216](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200621130005.png)

* 状态栏  最多只有一个

* 铆接部件  可以有多个

* 核心部件  只能有一个

  ~~~C++
  #include "mainwindow.h"
  #include <QMenuBar>
  #include <QToolBar>
  #include <QDebug>
  #include <QPushButton>
  #include <QStatusBar>
  #include <QLabel>
  #include <QDockWidget>
  #include <QTextEdit>
  MainWindow::MainWindow(QWidget *parent)
      : QMainWindow(parent)
  {
      //设置窗口大小
      resize(800,600);
  
      //菜单栏   最多只能有一个！！
      //菜单栏创建
      QMenuBar *bar = menuBar();
      //将菜单栏放入窗口中
      setMenuBar(bar);
  
      //创建菜单
      QMenu *FileMenu = bar->addMenu("文件");
      QMenu *EditMenu = bar->addMenu("编辑");
  
      //创建菜单项（二级菜单）
      FileMenu->addAction("新建");
      //添加分割线
      FileMenu->addSeparator();
      FileMenu->addAction("打开");
  
      //工具栏  可以有多个
      //  this  是为了将工具栏添加到对象树中，简化内存管理
      //菜单栏不需要加  this  ，因为其默认自动加入对象树中
      QToolBar * toolbar = new QToolBar(this);
      //将工具栏添加到窗口中，默认在上方
      addToolBar(toolbar);
      //更改工具栏默认窗口位置,位置信息可在官方帮助文档中找到
      //addToolBar(Qt::LeftToolBarArea,toolbar);
  
      //设置工具栏只允许上下停靠
      //注意  区域之间分割用  |
      toolbar->setAllowedAreas(Qt::TopToolBarArea|Qt::BottomToolBarArea);
  
      //设置不允许工具栏浮动
      toolbar->setFloatable(false);
  
      //设置移动（工具栏能否拖动的  总开关 ）
      //toolbar->setMovable(false);
  
      //工具栏中设置内容
      //可以设置直接与菜单栏中内容相关联
      QAction *newAction = FileMenu->addAction("新建");
      toolbar->addAction(newAction);
      //添加分割线
      toolbar->addSeparator();
      //也可以单独设置
      toolbar->addAction("保存");
  
      //工具栏中添加控件（如按钮）
      QPushButton *btn = new QPushButton ("啦啦啦",this);
      toolbar->addWidget(btn);
  
  
      //状态栏  最多有一个
      QStatusBar * stBAR = statusBar();
      //设置到窗口中
      setStatusBar(stBAR);
      //放标签控件
      QLabel * label = new QLabel("提示信息",this);
      stBAR->addWidget(label);
      //设置  label位置  在右侧
      QLabel * label2 = new QLabel("右侧提示信息",this);
      stBAR->addPermanentWidget(label2);
  
      //铆接部件  （浮动窗口）
      QDockWidget * dockWidget1 = new QDockWidget("浮动",this);
      //添加到窗口  在窗口下方
      //此下方是指 核心 的下方，如果没有核心内容，则此铆接部件将显示在菜单栏下方！
      addDockWidget(Qt::BottomDockWidgetArea,dockWidget1);
  
      //设置 中心 （核心） 部件  只能有一个
      //此时浮动窗口就会在下方了
      QTextEdit * edit= new QTextEdit(this);
      setCentralWidget(edit);
  
      //设置浮动窗口停靠范围，只允许上下
      dockWidget1->setAllowedAreas(Qt::TopDockWidgetArea|Qt::BottomDockWidgetArea);
  }
  
  MainWindow::~MainWindow()
  {
  }
  
  
  ~~~

  最终效果

  ![批注 2020-06-21 131636](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200621131647.png)

---

## 3.资源文件

* 1.将文件  拷贝  到项目位置下

* 右键项目 -》添加新文件 ——》Qt -》Qt resource Fle -》给资源文件起名

* res生成  res.qrc

* 编辑资源  右键res.qrc ——》选择 open in editor

* 添加前缀 再添加文件

* 使用  “： + 前缀名 + 文件名”

  ~~~C++
  #include "mainwindow.h"
  #include "ui_mainwindow.h"
  
  MainWindow::MainWindow(QWidget *parent)
      : QMainWindow(parent)
      , ui(new Ui::MainWindow)
  {
      ui->setupUi(this);
      //为状态栏等添加图标
      //ui->actionNew->setIcon(QIcon("G:/壁纸/000c2970d3a81fb6190305.jpg"));
      //但是上述形式只能在当前电脑使用，因为图片为 绝对路径
  
      //故需使用Qt资源文件   详情见视频
      //格式  ": + 前缀名 + 文件名 "
      ui->actionNew->setIcon(QIcon(":/image/000c2970d3a81fb6190305.jpg"));
  }
  
  MainWindow::~MainWindow()
  {
      delete ui;
  }
  
  
  ~~~

  显示结果

  ![批注 2020-06-21 135847](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200621135856.png)

---

## 4.对话框

* 分类 (更多详细分类**见官方帮助文档**)

  * 模态对话框   不可以对其他窗口进行操作

    * exec()相当使代码于阻塞在此行

  * 非模态对话框  可以对其他窗口进行操作

    ~~~C++
    #include "mainwindow.h"
    #include "ui_mainwindow.h"
    #include <QDialog>
    #include <QDebug>
    MainWindow::MainWindow(QWidget *parent)
        : QMainWindow(parent)
        , ui(new Ui::MainWindow)
    {
        ui->setupUi(this);
    
        //点击新建按钮，弹出一个对话框
        connect (ui->actionNew,&QAction::triggered,[=](){
            //对话框  分类
            //模态对话框 （不可以对其他窗口进行操作）  非模态对话框 （可以对其他窗口进行操作）
    
            //模态对话框创建
            //QDialog dlg(this);
            //.exec（）实现模块对话框
            //dlg.exec();
            //qDebug()<<"模态对话框弹出了";
    
            //非模态对话框创建
            //一下的方式创建后之后  一闪而过
            //因为以下方式创建的dlg2不在对象树中，show（）运行完之后即被丢弃
            //QDialog dlg2(this);
            //        dlg2.show();
    
            //以下操作可以一直留存
            //此种方式的dlg3创建于对象树之中，只有在主窗体关闭后才会被丢弃
            QDialog *dlg3 = new QDialog(this);
            dlg3->resize(300,180);
            dlg3->show();
            //此句函数中 Qt::WA_DeleteOnClose 确保了关闭对话框时丢弃此对象，保证了内存不会溢出
            //Qt::WA_DeleteOnClose  对应value = 55 （55号属性）
            dlg3->setAttribute(Qt::WA_DeleteOnClose);
            qDebug()<<"非模态对话框创建了";
        });
    }
    
    MainWindow::~MainWindow()
    {
        delete ui;
    }
    
    
    ~~~

* 标准对话框  ------消息对话框

  * 分类 错误、信息、提问、警告、颜色、文件、字体 等对话框

  * QMessageBox 静态成员函数  创建对话框

  * 参数1 父亲 参数2 标题 参数3 显示内容 参数4 按键类型 参数5 默认关联回车按键

  * 返回值  也是  StandardButton  类型 ，利用返回值判断用户的选项

    ~~~C++
    #include "mainwindow.h"
    #include "ui_mainwindow.h"
    #include <QDialog>
    #include <QDebug>
    #include <QMessageBox>
    #include <QColorDialog>
    #include <QFileDialog>
    #include <QFontDialog>
    MainWindow::MainWindow(QWidget *parent)
        : QMainWindow(parent)
        , ui(new Ui::MainWindow)
    {
        ui->setupUi(this);
    
        //点击新建按钮，弹出一个对话框
        connect (ui->actionNew,&QAction::triggered,[=](){
            //对话框  分类
            //模态对话框 （不可以对其他窗口进行操作）  非模态对话框 （可以对其他窗口进行操作）
    
            //模态对话框创建
            //QDialog dlg(this);
            //.exec（）实现模块对话框
            //dlg.exec();
            //qDebug()<<"模态对话框弹出了";
    
            //非模态对话框创建
            //一下的方式创建后之后  一闪而过
            //因为以下方式创建的dlg2不在对象树中，show（）运行完之后即被丢弃
            //QDialog dlg2(this);
            //        dlg2.show();
    
    //        //以下操作可以一直留存
    //        //此种方式的dlg3创建于对象树之中，只有在主窗体关闭后才会被丢弃
    //        QDialog *dlg3 = new QDialog(this);
    //        dlg3->resize(300,180);
    //        dlg3->show();
    //        //此句函数中 Qt::WA_DeleteOnClose 确保了关闭对话框时丢弃此对象，保证了内存不会溢出
    //        //Qt::WA_DeleteOnClose  对应value = 55 （55号属性）
    //        dlg3->setAttribute(Qt::WA_DeleteOnClose);
    //        qDebug()<<"非模态对话框创建了";
    
            //消息对话框
            //（报）错误对话框
            //QMessageBox::critical(this,"critical","错误");
            //(报)ti'shi'xin'xi提示信息对话框
            //QMessageBox::information(this,"info","信息");
            //(报)提示信息对话框  如：确定关闭吗？ y|n  此处显示内容可改 如 QMessageBox::Save(更多见官方文档)
            //更改默认选项，默认选项放到 第五项参数  中
            //参数1 父亲  参数2 标题  参数3 提示内容  参数4 按键类型  参数5 默认关联回车按键
    
            //QMessageBox::question(this,"ques","提问",QMessageBox::Save|QMessageBox::Cancel,QMessageBox::Cancel);
    
            //根据用户不同选择进行不同操作
    //        if (QMessageBox::Save==QMessageBox::question(this,"ques","提问",QMessageBox::Save|QMessageBox::Cancel,QMessageBox::Cancel))
    //            qDebug()<<"用户选择的是保存";
    //        else
    //            qDebug()<<"用户选择的是取消";
            //警告对话框  注意为小写warning
            //QMessageBox::warning(this,"Warn","警告");
    
            //颜色对话框
            //需要单独头文件<QColorDialog>
            //弹出如word选择颜色的选择对话框 其默认颜色为红色 255，0，0
            //QColorDialog::getColor(QColor(255,0,0));
    
            //返回用户选择的颜色信息（r，g，b）
    //        QColor color = QColorDialog::getColor(QColor(255,0,0));
    //        qDebug()<<"r ="<<color.red()<<"g ="<<color.green()<<"b ="<<color.blue();
    
    
            //文件对话框  如想打开D盘下对话框、
            //需要单独头文件<QFileDialog>
            //显示D:\\md笔记中所有md类型的文件
            //参数1 父亲 参数2 标题 参数3 路径 参数4 过滤文件格式
            //QFileDialog::getOpenFileNames(this,"打开文件","D:\\md笔记","(*.md)");
    
            //此函数返回值为 QStringList ，返回的是选择的文件路径
            //QStringList str1 = QFileDialog::getOpenFileNames(this,"打开文件","D:\\md笔记","(*.md)");
            //qDebug()<<str1;
    
            //字体对话框，需要头文件 <QFontDialog>
            //打开word中 选择字体的对话框
            //bool flag;
            //QFontDialog::getFont(&flag,QFont("fira code",36));
    
            //返回用户选择的字体和字号
            bool flag;
            QFont font = QFontDialog::getFont(&flag,QFont("fira code",36));
            qDebug()<<"字体： "<<font.family().toUtf8().data()<<"字号： "<<font.pointSize()
                   <<"是否加粗"<<font.bold()<<"是否倾斜： "<<font.italic();
        });
    }
    
    MainWindow::~MainWindow()
    {
        delete ui;
    }
    
    
    ~~~



## 5.界面布局  详情见视频

* 实现登录窗口 （ui界面拖动调整实现）
* 利用布局方式 给窗口进行美化
* 选取 widget 进行布局  水平布局 、垂直布局、栅格布局
* 给用户名、密码、登录、退出按钮进行布局
* 默认窗口和控件之间  有 9 像素的间隙 ，可以调整 layoutLeftMargin等
* 利用弹簧进行布局



## 控件  详情见视频

*  按钮组
  * QPushButton 常用按钮
  * arutoraise 突起风格