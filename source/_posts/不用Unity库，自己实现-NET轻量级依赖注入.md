---
title: 不用Unity库，自己实现.NET轻量级依赖注入
date: 2016-11-20 22:30:37
tags:
---
&#160; &#160; &#160; &#160;在面向对象的设计中，依赖注入（**IoC**）作为一种重要的设计模式，主要用于削减计算机程序的**耦合问题**，相对于Java中的Spring框架来说，微软企业库中的Unity框架是目前.NET平台中运用比较广泛的依赖注入框架之一（还有的Spring.NET等）。但是对于这些“官方版本”的强大依赖注入框架，通常使用和配置都比较复杂，我个人更希望实现一种“约定胜于配置”轻量级IoC框架。

&#160; &#160; &#160; &#160;实现依赖注入主要是运用C#中的反射技术，通过配置文件，把代码的实现注入到接口中。用户只是访问接口，对于接口的实现一概不知，可以有效地**对接口和实现进行解耦**，在介绍代码之前，先来看一下我们的配置文件：

```c#
<appSettings>
        <add key="Domain.Entity.IRepository" value="Infrastructure.Repository"/>
</appSettings>
```

 &#160; &#160; &#160; &#160;Domain.Entity.IRepository与Infrastructure.Repository都是.NET中的程序集（dll），其中Domain.Entity.IRepository中定义了领域模型中所需要的仓储接口，Infrastructure.Repository中定义了仓储接口的实现。领域模型需要访问数据的时候，并不是直接访问ADO或者ORM，而是通过访问Domain.Entity.IRepository中的仓储接口，再通过依赖注入把Infrastructure.Repository中的实现注入到接口中，这样就有效隔离了领域模型对数据实现的依赖，如果后期需要更换数据库或ORM框架，只需要实现另一个Infrastructure.Repository，并更新依赖注入配置即可。

 &#160; &#160; &#160; &#160;下面介绍轻量级依赖注入的实现方法，先贴代码：　
```c#
public sealed class DependencyInjector
    {
        /// <summary>
        /// 根据名称和构造函数的参数加载相应的类
        /// </summary>
        /// <typeparam name="T">需要加载的类所实现的接口</typeparam>
        /// <param name="className">类的名称</param>
        /// <param name="args">构造函数的参数(默认为空)</param>
        /// <returns>类的接口</returns>
        public static T GetClass<T>(string className, object[] args = null) where T : class
        {
            //获取接口所在的命名空间
            string factoryName = typeof(T).Namespace;
            //通过依赖注入配置文件获取接口实现所在的命名空间
            string dllName = ConfigurationManager.AppSettings[factoryName];
            //获取类的全名
            string fullClassName = dllName + "." + className;
            //根据dll和类名，利用反射加载类
            object classObject = Assembly.Load(dllName).CreateInstance(fullClassName, true, BindingFlags.Default, null, args, null, null); ;
            return classObject as T;
        }
    }
```
&#160; &#160; &#160; &#160;代码中的typeof(T).Namespace，其中T是接口的类型，获取接口的命名空间以后，再通过我们的配置文件，就可以获取类的命名空间，再加上类名，通过c#的反射机制，就可以加载类的实现。也许你会说，我们只知道接口，并不知道类名是什么，别急，我们对该代码进一步进行封装：
```c#
public sealed class IoC
{
    /// <summary>
    /// //通过接口名称和构造函数的参数得到实现
    /// </summary>
    /// <typeparam name="T">接口类型</typeparam>
    /// <param name="args">构造函数的参数</param>
    /// <returns>接口的实现</returns>
    public static T Resolve<T>(object[] args = null) where T : class
    {
            //获取类名
            string className = typeof(T).Name.Substring(1);
            //通过判断fullName中是否包含`符号来判断是否是泛型
            string fullName = typeof(T).FullName;
            int flag = fullName.IndexOf('`');
            //如果是泛型，需要特殊处理
            if (flag != -1)
            {
                int dot = fullName.LastIndexOf('.', flag);
                //这里去掉方法名前面的点和I
                className = fullName.Substring(dot + 2);
            }
            return DependencyInjector.GetClass<T>(className, args);
     }
}
```
&#160; &#160; &#160; &#160;在这里我们就用到了上面所说的”约定胜于配置“，我们约定接口的名称是在类名的前面加上I，并且接口和实现必须是独立的程序集。如果类名是Helper，那么接口的名字就必须是IHelper，我们通过去掉接口前面的I来获得实现的类名。如果是泛型接口，还需要进行特殊的处理，这里我们通过判断类型的全名中是否包含”`“符号来判断是否泛型接口。

&#160; &#160; &#160; &#160;下面举一个使用该依赖注入的例子：
```c#
public static Book GetById(string bookID)
{
       IRepository<Book> bookRep = IoC.Resolve<IRepository<Book>>();
       return bookRep.GetByKeys(bookID);
}

public int GetBookCount()
{
       IBookRepository bookRep = IoC.Resolve<IBookRepository>();
       return bookRep.GetCount();
}
```
 &#160; &#160; &#160; &#160;在这里我们是不知道接口的实现的，通过我们封装的IoC.Resolve方法，可以把我们配置文件中的接口实现”注入到接口“中，通过这种解耦的方式，后期我可以更加灵活地对代码进行重构。