# NanGua #
**NanGua**，一种借鉴了 中间件 写法的代码优化解决方案。俗称:并行器。<br/>

**NanGua**为您提供了如下功能:<br/>
&nbsp;&nbsp;嵌套解耦<br/>
&nbsp;&nbsp;灵活封装<br/>
&nbsp;&nbsp;嵌套并行(代码并行、逻辑并行)<br/>
&nbsp;&nbsp;任务并行<br/>
&nbsp;&nbsp;管道分支<br/>
 
**NanGua**最大的特点是解耦多层嵌套，嵌套之间独立，不依赖于上层嵌套，可以灵活封装成独立的函数。<br/>

注：NanGua目前提供了C#、JS版本。<br/>


## Get Started ##
<br/>

**1、创建第一个NanGua**<br/>

	NanGua<object> mw = new NanGua<object>(null);
	mw.Run();
	//Output:中间件文档参考: https://github.com/Sc7-git/NanGua.Net

<br/>
__2、Use函数和两个重要参数：next、context __：<br>

把NanGua看作一个管道；<br>

context就是管道中流动的水，在代码中就理解为上下文对象；<br>

next就是管道上的处理节点,如阀门、分流阀、净水器、滤网等一系列对水进行处理的工具，在代码中就理解为下一个中间件(传统意义上的中间件是指游走在操作系统和应用程序之间，但我认为这种在管道和水之间的处理节点也是一种中间件)。<br>

Use就是在管道上添加处理节点，在代码中理解为注册中间件。之后介绍的Map、When都是基于Use的实现。注：注册完成所有中间件之后需要Run才能运行！<br>

**3、嵌套解耦、灵活封装、嵌套并行(代码并行、逻辑并行)**
多个嵌套函数如何解耦？异步嵌套如何解耦？在不同场景下嵌套顺序不同如何封装(例:a函数中嵌套b函数，但是在另一场景又需要b函数嵌套a函数，此时该如何封装函数才能代码复用。在实际应用中可不止a、b两个函数。)？

	NanGua<object> mw = new NanGua<object>(null);
	//使用Use注册中间件
	mw.Use(next => context =>{
        Console.WriteLine("第一个");
	    //把上层嵌套中的代码挪到这里来
	    //.... 省略业务逻辑
	    //next是下一个中间件，也就是下层嵌套。根据业务场景决定是否调用。
	    next(context);
	})
	//使用Use注册中间件
	.Use(next => context =>{
        Console.WriteLine("第二个");
	    //把下层嵌套中的代码挪到这里来
	})
	.Run();

灵活封装体现在可以把Use注册的中间件独立成一个函数，而非代码块。注：Use可以注册SimpleDelegate<T>和Func<DelegateInput<T>, DelegateOutput<T>>两种类型委托。

    static void Main(string[] args)
    {
        NanGua<object> mw = new NanGua<object>(null);           
        mw.Use(A) //使用Use注册中间件
        .Use(B)//使用Use注册中间件
        .Run();
        Console.ReadKey();
    }

	//Func<DelegateInput<T>, DelegateOutput<T>>
    static DelegateOutput<object> A(DelegateInput<object> next)
    {
        return context =>
        {
            Console.WriteLine(nameof(A));
            next(context);
        };
    }

	//SimpleDelegate<T>
    static void B(DelegateInput<object> next, object context)
    {
        Console.WriteLine(nameof(B));
        next(context);

    }


**4、管道分支** <br/>
从宏观上看就是由单一管道结构转变为树形结构。<br/>
__注意__：map返回的是一个新的NanGua对象，同一个NanGua对象注册的map是同一层级的分支。所以mw.map()和mw.map()是同一级分支，但mw.map()和mw.map().map()则是上下级分支。不要因为链式编程的便捷而带来不必要的麻烦。<br/>
__注意__：因为map返回的并非原本的NanGua对象，所以一定是使用原本的NanGua对象执行Run。

例1：

    NanGua<object> mw = new NanGua<object>(null);
    var map1 = "map1";
    var map2 = "map2";
    mw.Use(next => context =>{
        Console.WriteLine("Use");
        //next(context, map1);
        next(context, map2);
    });
    mw.Map(map1, next => context => {
        Console.WriteLine(map1);
        next(context);
    });
    mw.Map(map2, next => context =>
    {
        Console.WriteLine(map2);
        //next(context);
    });
    mw.Run();
	//Output:Use
	//Output:map2

例2：

    NanGua<object> mw = new NanGua<object>(null);
    var map1 = "map1";
    var map2 = "map2";
    var map1_1 = "map1.1";

    mw.Use(next => context =>
    {
        Console.WriteLine("Use");
        next(context, map1);
        //next(context, map2);
    });

    mw.Map(map1, next => context =>
    {
        Console.WriteLine(map1);
        next(context, map1_1);
    }).Map(map1_1, next => context =>
    {
        Console.WriteLine(map1_1);
        next(context);
    });
    mw.Map(map2, next => context =>
    {
        Console.WriteLine(map2);
        //next(context);
    });
	//Output:Use
	//Output:map1
	//Output:map1.1
	//Output:中间件文档参考: https://github.com/Sc7-git/NanGua.Net


**5、任务并行**<br/>
实现了When、WhenAll、WhenOne，**在js代码中用处更广，在c#异步编程中可以考虑使Task.WhenAll、Task.WhenAny替代。**<br/>
WhenAll和WhenOne都是基于when的，所以只介绍When，When的第二个参数是设置延时对象的通过数量（0是默认所有），通过when可以自定义完成数量。<br/>
When是基于Use实现的，所以When和Use可以实现连接。<br/>
When的用法分三步：1、创建延迟对象，2、在注册函数中返回延延迟对象数组，3、异步任务执行完成后延迟对象调用Complete通知，当通知数量满足设置的通过数量时进入下一个中间件。

	static void Main(string[] args)
	{
	    Stopwatch sw = new Stopwatch();
	    NanGua<object> mw = new NanGua<object>(null);
	    mw.When((context, createDelay) =>{
	        sw.Start();
	        //1、创建延迟对象
	        var d1 = createDelay();
	        var d2 = mw.CreateDelay();
	
	        RequestAsync(3, () =>
	        {
	            Console.WriteLine("异步任务1完成了");
	            //3、异步任务完成时回调通知
	            d1.Complete();
	        });
	        RequestAsync(5, () =>
	        {
	            Console.WriteLine("异步任务2完成了");
	            //3、异步任务完成时回调通知
	            d2.Complete();
	        });
	        //2、在注册函数中返回所有申明的延迟对象
	        return new NanGua<object>.Delay[] { d1, d2 };
	    }, 1).Use(next => context =>{
	         Console.WriteLine("Use");
	         sw.Stop();
	         Console.WriteLine(sw.Elapsed);
	         next(context);
	     })
	      .Run();
        Console.ReadKey();
    }

试试把上面代码中when第二个参数从1改为2、3看看效果是怎么样的。


**其他：**<br/>
One：第一个执行，只允许调用一次。<br/>
End：最后一个执行，允许多次调用。end不是附加，而是覆盖。End是有默认值的。<br/>
