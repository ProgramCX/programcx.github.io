---
title: Qt 5.15 connect() 函数的一个大坑
description: Qt 5.15 connect() 函数的一个大坑
date: 2024-09-16
image: qt.webp
categories:
    - C++
    - Qt
---

# Qt 5.15 connect() 函数的一个大坑

有以下代码：

```cpp
 connect(ui->styleComboBox, &QComboBox::currentIndexChanged, this, [this](int index) {
        //TODO: change theme
        spdlog::info("User change default theme to {}", index);
        AppConfig::instance().setBasic("style", index);
    });
```
以上代码将`ui->styleComboBox`的`QComboBox::currentTextChanged`信号连接至lambda表达式里的匿名函数。看似没有什么问题，但是在Qt 5.15上报错：
```error
/Users/runner/work/FlowD/FlowD/src/SettingsBasicWidget.cpp:21:5: error: no matching member function for call to 'connect'
    connect(ui->styleComboBox, &QComboBox::currentIndexChanged, this, [this](int index) {
    ^~~~~~~
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:222:36: note: candidate function not viable: no overload of 'currentIndexChanged' matching 'const char *' for 2nd argument
    static QMetaObject::Connection connect(const QObject *sender, const char *signal,
                                   ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:225:36: note: candidate function not viable: no overload of 'currentIndexChanged' matching 'const QMetaMethod' for 2nd argument
    static QMetaObject::Connection connect(const QObject *sender, const QMetaMethod &signal,
                                   ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:481:41: note: candidate function not viable: no overload of 'currentIndexChanged' matching 'const char *' for 2nd argument
inline QMetaObject::Connection QObject::connect(const QObject *asender, const char *asignal,
                                        ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:242:43: note: candidate template ignored: couldn't infer template argument 'Func1'
    static inline QMetaObject::Connection connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal,
                                          ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:283:13: note: candidate template ignored: couldn't infer template argument 'Func1'
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, const QObject *context, Func2 slot,
            ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:322:13: note: candidate template ignored: couldn't infer template argument 'Func1'
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, const QObject *context, Func2 slot,
            ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:274:13: note: candidate function template not viable: requires 3 arguments, but 4 were provided
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, Func2 slot)
            ^
/Users/runner/work/FlowD/Qt/5.15.2/clang_64/lib/QtCore.framework/Headers/qobject.h:314:13: note: candidate function template not viable: requires 3 arguments, but 4 were provided
            connect(const typename QtPrivate::FunctionPointer<Func1>::Object *sender, Func1 signal, Func2 slot)
            ^
1 error generated.
```
为什么会出现这种情况呢？
因为QComboBox 有两个重载的信号，原型分别是`void	currentTextChanged(const QString &text)`和`void	currentIndexChanged(int index)`，上面的代码我们明显是想要连接`void	currentIndexChanged(int index)`这个信号，但是编译器当成了`void	currentTextChanged(const QString &text)`这个信号，所以会报出参数不匹配的错误:`matching member function for call to 'connect'
    connect(ui->styleComboBox, &QComboBox::currentIndexChanged, this, [this](int index) `。
    上面的代码在Qt 6上不会报错。
    
   我们该如何解决呢？只需使用预定义宏，将比Qt 6.0.0版本低的部分的代码改成`QOverload<int>::of(&QComboBox::currentIndexChanged)`即可：
   

```cpp
#if QT_VERSION <= QT_VERSION_CHECK(6, 0, 0)
    connect(ui->styleComboBox, QOverload<int>::of(&QComboBox::currentIndexChanged), this, [this](int index) {
        //TODO: change theme
        spdlog::info("User change default theme to {}", index);
        AppConfig::instance().setBasic("style", index);
    });
#else
    connect(ui->styleComboBox, &QComboBox::currentIndexChanged, this, [this](int index) {
        //TODO: change theme
        spdlog::info("User change default theme to {}", index);
        AppConfig::instance().setBasic("style", index);
    });
#endif
```