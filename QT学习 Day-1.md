---
title: QT
date: 2020-06-20 18:25:22
tags:
  - QT
keywords: QT
categories: 理论
---

# QT学习 Day-1

> 前言：没想到学习QT的最大动力竟然来自大作业DDL
>
> 从今天开始吧 2020/6/20 18点25分

## 1.QT简介

###     1.1QT优点

* 跨平台

* 接口简单，易于上手
* 一定程度上简化了内存回收

###     1.2成功案列

* Lnux桌面系统KDE
* 谷歌地图
* VLC多功能化媒体播放器

----

##  2.创建第一个QT项目

    ### 2.1 创建过程

* 点击创建项目后，Location选择项目位置和项目名称
* Bulid System选择编译系统，qmake，cmake等
* **qmanke cmake差异：**
* Details选择默认窗口类，QWidget为空白窗体，QMainWindow带有菜单栏、工具栏等，QDialog带有对话框

* Translation选择none即可
* Kits选择使用的QT版本
* Summary为选择版本控制软件

## 3.命名规范即快捷键

* 命名规范：
  1. 类名 首字母大写，单词和单词之间首字母大写
  2. 函数名、变量名称 首字母小写，单词和单词之间首字母大写
* 快捷键：
  1. 注释  ctrl + /
  2. 运行  ctrl + r
  3. 编译  ctrl + b
  4. 字体缩放  ctrl + 鼠标滚轮
  5. 查找关键字  ctlr + f
  6. 整行移动  ctrl + shift + ⬆或⬇
  7. 帮助文档  F1 或打开Assistant
  8. 自动对齐    ctrl + i
  9. 同名之间的.h和.cpp文件切换  F4

## 4.按钮常用API

```c++
//创建按钮btn1
    QPushButton * btn1=new QPushButton;
        btn1->show();//show()一以顶层方式弹出
        //使btn1对象 依赖在myWidget窗口中（即在当前窗口中显示）
        btn1->setParent(this);
        //为按钮设置内容
        btn1->setText("第一个按钮");
    QPushButton *btn2=new QPushButton("第二个按钮",this);
        //移动按钮位置
        btn2->move(250,250);
        //按钮重置大小
        btn2->resize(100,50);
    //设置窗口大小
    resize (800,640);
    //设置窗口标题
    setWindowTitle("宾馆客房管理系统");
    //设定固定窗口大小，用户无法拖动
    setFixedSize (800,640);
```

## 5.对象树

* 当创建的对象在堆区时候，如果是从QObjict派生而来的子类，可以不用管理内存释放的操作，此对象会自动放入对象树中，在程序结束时自动释放。
* 其一定程度上简化了内存回收机制

## 坐标体系

![QQ截图20200620213437](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200620213550.png)

## 6.信号与槽

* 连接函数：connect（）
* 参数
  1. 参数1  信号的发送者
  2. 参数2  发送的信号
  3. 参数3  信号的接收者
  4. 参数4  处理的槽函数（当接收到信号时做什么）
* 松散耦合
* 实现 点击按钮 关闭窗口的案例

```c++
//此conneconnect函数的作用是点击btn2关闭窗口
    connect (btn2,&QPushButton::clicked,this,&MyWidget::close);
```

* 自定义信号和槽函数

  * 自定义信号

    1. 返回值为void
    2. 需要声明，不需要实现
    3. 可以有参数，可以重载
    4. 写到 signals 下

  * 自定义槽函数

    1. 返回void
    2. 需要声明，也需要实现
    3. 可以有参数，可以重载
    4. 写道 public slots 下（推荐）（可被识别为槽函数）或者全局函数（不推荐，且我的版本无法被识别为槽函数） 

  * 触发自定义信号

    * emit 自定义信号

    ```c++
    void Widget::classover()
    {
        //触发老师饿了的信号
        //emit关键字用以触发信号
        emit ls->hungery();
    }
    ```

  * 案例：下课后，老师触发饿了信号，学生回应

    * **pro(qmake)文件**

      ~~~c++
      QT       += core gui
      
      greaterThan(QT_MAJOR_VERSION, 4): QT += widgets
      
      CONFIG += c++11
      
      # The following define makes your compiler emit warnings if you use
      # any Qt feature that has been marked deprecated (the exact warnings
      # depend on your compiler). Please consult the documentation of the
      # deprecated API in order to know how to port your code away from it.
      DEFINES += QT_DEPRECATED_WARNINGS
      
      # You can also make your code fail to compile if it uses deprecated APIs.
      # In order to do so, uncomment the following line.
      # You can also select to disable deprecated APIs only up to a certain version of Qt.
      #DEFINES += QT_DISABLE_DEPRECATED_BEFORE=0x060000    # disables all the APIs deprecated before Qt 6.0.0
      
      SOURCES += \
          main.cpp \
          student.cpp \
          teacher.cpp \
          widget.cpp
      
      HEADERS += \
          student.h \
          teacher.h \
          widget.h
      
      # Default rules for deployment.
      qnx: target.path = /tmp/$${TARGET}/bin
      else: unix:!android: target.path = /opt/$${TARGET}/bin
      !isEmpty(target.path): INSTALLS += target
      
      ~~~
      
    * **student头文件**
    
      ~~~C++
      #ifndef STUDENT_H
      #define STUDENT_H
      
      #include <QObject>
      
      class Student : public QObject
      {
          Q_OBJECT
      public:
          explicit Student(QObject *parent = nullptr);
      public slots:
          void treat();
          //treat（）函数重载，对应hungery（）函数的重载
          void treat(QString FoodName);
      signals:
      
      };
      //早期Qt版本，槽函数必须写道public slots下，高级版本可以写到public或者全局下
      //但是我的只有写在slots下时，才能被识别为槽函数！！
      //槽函数返回值为void ， 需要声明，也需要实现
      //可以有参数，也可以重载
      #endif // STUDENT_H
      
      ~~~
    
    * **teacher头文件**
    
      ~~~C++
      #ifndef TEACHER_H
      #define TEACHER_H
      
      #include <QObject>
      
      class Teacher : public QObject
      {
          Q_OBJECT
      public:
          explicit Teacher(QObject *parent = nullptr);
      signals:
          //signals  自定义信号写入此定义域
          //返回值为void ， 只需要声明，不需要实现
          //可以有参数，可以重载
          void hungery();
          //hungery（）函数重载,老师指定学生带什么饭
          void hungery(QString FoodName);
      
      };
      
      #endif // TEACHER_H
      
      ~~~
    
    * **widget头文件**
    
      ~~~C++
      #ifndef WIDGET_H
      #define WIDGET_H
      
      #include <QWidget>
      #include "teacher.h"
      #include "student.h"
      class Widget : public QWidget
      {
          Q_OBJECT
      
      public:
          Widget(QWidget *parent = nullptr);
          ~Widget();
      private:
          Teacher *ls;
          Student *xs;
          void classover();
      };
      #endif // WIDGET_H
      
      ~~~
    
      
    
    
    
    * **student cpp文件**
    
      ~~~c++
      #include "student.h"
      #include <QDebug>
      Student::Student(QObject *parent) : QObject(parent)
      {
      
      }
      void Student::treat()
      {
          qDebug()<<"吃你大爷！";
      }
      void Student::treat(QString FoodName)
      {
          //这样输出结果为：就你还想吃 "宫保鸡丁" ???
          //FoodName带有“”
          //qDebug()<<"就你还想吃"<<FoodName<<"???";
          //这样输出结果FoodName将不带有“”（见下图）
      //原理为将
          //QString -> char * 先转成QByteArray （ .toUtf8() ) ,再转 char * ( .data() )
      qDebug()<<"就你还想吃"<<FoodName.toUtf8().data();
      }
      
      ~~~
    
      ![批注 2020-06-21 020627](https://lalalademaxiya01.oss-cn-beijing.aliyuncs.com/img20200621020709.png)
    
    * **teacher cpp文件**
    
      ~~~c++
      #include "teacher.h"
      
      Teacher::Teacher(QObject *parent) : QObject(parent)
      {
      
      }
      
      ~~~
      
    * **widget cpp文件**
    
      ~~~C++
      #include "widget.h"
      #include <QPushButton>
      Widget::Widget(QWidget *parent)
          : QWidget(parent)
      {
          //创建一个Teacher对象
          this->ls = new Teacher(this);
          //创建一个Student对象
          this->xs = new Student(this);
          //老师饿了，学生回应
          //connect(ls,&Teacher::hungery,xs,&Student::treat);
          //连接带参数的 信号和曹
          //指针-》地址
          //故函数指针-》函数地址
          //声明并实现函数指针
          //函数指针格式
          //类型说明符 (*函数名)(参数)
          //如 bool length_compare(const string &, const string &);
          //的函数类型为:bool (const string &, const string &);
          //的函数指针形式为
          //bool (*pf)(const string &, const string &);
          //也可以在定义是赋值bool (*pf)(const string &, const string &)=length_compare
          void(Teacher:: *teacherSignal)(QString) = &Teacher::hungery;
          void(Student:: *StudentSlot)(QString) = &Student::treat;
          connect(ls,teacherSignal,xs,StudentSlot);
          //调用下课函数
          //classover();
          QPushButton *btn1 = new QPushButton ("下课",this);
          //点击按钮，触发下课
          //connect (btn1,&QPushButton::clicked,this,&Widget::classover);
      
          //无参数信号和槽连接
          //注意为无参数函数
          void(Teacher:: *teacherSignal2)(void) = &Teacher::hungery;
          void(Student:: *StudentSlot2)(void) = &Student::treat;
          connect (ls,teacherSignal2,xs,StudentSlot2);
      
          //信号连接信号
          connect (btn1,&QPushButton::clicked,ls,teacherSignal2);
          //断开信号
          //disconnect(ls,teacherSignal2,xs,StudentSlot2);
      
          //扩展
          //1.信号可以连接信号
          //2.一个信号可以连接多个槽函数
          //3.多个信号，可以连接同一个槽函数
          //4.信号和槽函数的参数  必须类型一一对应！！
          //5.信号和槽函数的参数个数  信号的参数个数  可以多于  槽函数的参数个数
          //                      槽函数的参数个数  必须小于等于  信号的参数个数
      
      
          //Qt4版本及之前的信号和槽的连接方式
          //connect(ls,SIGNAL(hungery()),xs,SLOT(treat()));
          //优点： 参数直观  缺点： 类型不做检测，能通过编译，仅报WARN
          //Qt5以上版本：  支持Qt4的版本写法，但Qt4不支持Qt5的写法（这不废话吗）
      
      }
      void Widget::classover()
    {
          //触发老师饿了的信号
        //emit关键字用以触发信号
          emit ls->hungery("宫保鸡丁");
    }
      
      Widget::~Widget()
      {
      }
      
      
      ~~~
    
    * **mian cpp文件**
    
      ~~~~C+++
      #include "widget.h"
      
      #include <QApplication>
      
      int main(int argc, char *argv[])
      {
        QApplication a(argc, argv);
          Widget w;
          w.show();
          return a.exec();
      }
      
      ~~~~
  
  
  
  ## 7.当自定义信号和槽出现重载
  
  * 需要用函数指针 明确指向函数的地址
  
    ~~~C++
    //指针-》地址
        //故函数指针-》函数地址
        //声明并实现函数指针
        //函数指针格式
        //类型说明符 (*函数名)(参数)
        //如 bool length_compare(const string &, const string &);
        //的函数类型为:bool (const string &, const string &);
        //的函数指针形式为
        //bool (*pf)(const string &, const string &);
        //也可以在定义是赋值bool (*pf)(const string &, const string &)=length_compare
        void(Teacher:: *teacherSignal)(QString) = &Teacher::hungery;
        void(Student:: *StudentSlot)(QString) = &Student::treat;
      connect(ls,teacherSignal,xs,StudentSlot);
    ~~~
  
  * QString 转成 QByteArray
  
  * .toUtf-8 转为  QByteArray
    
    * .data()  转为 char *
    
  * 信号可以连接信号
  
  * 断开信号  disconnect
  
  > 其实已经到第二天了，但为了知识点的体系化，故仍在此编辑
  
  ## 8.扩展
  
  ```c++
      //1.信号可以连接信号
      //2.一个信号可以连接多个槽函数
      //3.多个信号，可以连接同一个槽函数
      //4.信号和槽函数的参数  必须类型一一对应！！
      //5.信号和槽函数的参数个数  信号的参数个数  可以多于  槽函数的参数个数
      //                      槽函数的参数个数  必须小于等于  信号的参数个数
  ```
  
  
  
  ## 9.Qt4 版本的写法
  
  ~~~C++
  //connect （信号的发送者，  发送的信号  SIGNAL（信号），  信号接收者  ，槽函数  SLOT(槽函数)）
  //Qt4版本及之前的信号和槽的连接方式
      connect(ls,SIGNAL(hungery()),xs,SLOT(treat()));
      //优点： 参数直观  缺点： 类型不做检测，能通过编译，仅报WARN
      //Qt5以上版本：  支持Qt4的版本写法，但Qt4不支持Qt5的写法（这不废话吗）
  ~~~
  
  
  
  

