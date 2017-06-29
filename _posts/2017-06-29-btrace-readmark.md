---
layout: post
title: Btrace 阅读小结
category: java
tag: [java]
---

1、简述
	BTrace的最大好处，是可以通过自己编写的脚本，获取应用的一切调用信息，而不需要不断地修改代码。特别严格的约束，保证自己的消耗特别小，只要定义脚本时不作大死，直接在生产环境打开也没影响。引用[@btrace](http://calvin1978.blogcn.com/articles/btrace1.html) 文章

2、使用方法
	2.1 我们先举个栗子
{% highlight java %}
	import com.sun.btrace.annotations.*;
	import static com.sun.btrace.BTraceUtils.*;

	@BTrace
	public class HelloWorld {
        @OnMethod(
                        clazz="com.game.module.role.service.RoleService",
                        method="getRoleInfo",
                        location=@Location(Kind.CALL)
                        )
        public static void call() {
                println("role info call ,t = "+ timeMillis() );
        }
	}
{% endhighlight %}

	然后ps找出要监控的java应用的pid，
 	./btrace $pid HelloWorld.java  或者 
	./btrace $pid HelloWorld.java > mylog 就跑起来了（$pid 是我们游戏服的进程ID)。
	./btracec HelloWorld.java 预编译命令、生成class文件
	基本上不用任何BTrace的知识，都能猜出HelloWorld会干啥。通过JVM Attach API，btrace把自己绑进了被监控的进程，按HelloWorld.java里的定义，进行AOP式的代码植入(在运行时，动态地将代码切入到类的指定方法、指定位置上的编程思想就是面向切面的编程就是AOP)。如果还想监控其他内容，直接修改HelloWorld.java，再执行一次btrace就可以了，不需要重启应用!
	
	
	2.2 应用场景
		*服务慢，能找出慢在哪一步，哪个函数里么？
		
		*谁调用了System.gc()，调用栈如何？

		*谁构造了一个超大的ArrayList?

		*什么样的入参或对象属性，导致抛出了这个异常？或进入了这个处理分支？
	
	
	2.3 为了避免Btrace脚本的消耗过大影响真正业务，所以定义了一系列不允许的事情
		*不允许调用任何类的任何方法，只能调用BTraceUtils 里的一系列方法和脚本里定义的static方法。
		
		*不允许创建对象，不允许For 循环。
		
		*BTrace植入过的代码，会一直在，直到应用重启为止。

	2.4拦截方法、举几个例子
{% highlight java %}
	package test;

	import com.sun.btrace.annotations.*;
	import static com.sun.btrace.BTraceUtils.*;

	@BTrace
	public class Example {

	//正则式 模糊定位、性能问题btrace不推荐这种搞法
	@OnMethod(clazz="/com\\.game\\.module\\..*/", method="/.*/")
	public static void enterMethod( @ProbeClassName String probeClass, @ProbeMethodName String probeMethod) {
		println("entered: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}

	//正则式 模糊定位、定义切入点
	@OnMethod(clazz="/com\\.game\\.module\\..*/", method="/.*/",location = @Location(Kind.RETURN))
	public static void retMethod( @ProbeClassName String probeClass, @ProbeMethodName String probeMethod) {
		println("returnd: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}

	//切入点是某个父类、然后他的每个子类都被监听
	@OnMethod(clazz="+com.game.common.service.AbstractGameService" ,method="/.*/")
	public static void extClass(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
		println("extclass returnd: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}	
	
	//在指定方法进入时切入
	@OnMethod(
			clazz="com.game.module.shop.manager.ShopManager" ,
			method="getItemsByType"
			)
	public static void shopbegin(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
		println("shopbegin: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}	

	//指定切入点、return返回时
	@OnMethod(
			clazz="com.game.module.shop.manager.ShopManager" ,
			method="getItemsByType" ,
			location = @Location(Kind.RETURN)
			)
	public static void shopend(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
		println("shopbegin: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}

	//指定切入点、指定某行
	@OnMethod(
			clazz="com.game.module.shop.manager.ShopManager" ,
			location = @Location(value = Kind.LINE, line = 166)
			)
	public static void shopLineStart(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
		println("shopLineStart: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}
	
	//指定切入点、指定某行
	@OnMethod(
			clazz="com.game.module.shop.manager.ShopManager" ,
			location = @Location(value = Kind.LINE, line = 238)
			)
	public static void shopLineEnds(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
		println("shopLineEnds: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}
	
	//指定构造函数
	@OnMethod(clazz="java.net.ServerSocket", method="<init>")
	public static void methodInit(@ProbeClassName String probeClass, @ProbeMethodName String probeMethod){
		println("methodInit: " + probeClass + "."  + probeMethod +" t = "+ timeMillis());
	}

	/**
	 * 打印堆栈信息
	 */
	@OnMethod(clazz = "java.lang.System", method = "gc")
	public static void onSystemGC() {
	    println("entered System.gc()");
	    jstack();
	}
	
	}
{% endhighlight %}
	
	
	
	
	
	


