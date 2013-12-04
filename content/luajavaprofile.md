Title: luajava性能优化分析
Date: 2013-12-02 20:20
Category:编程 
Author: zhengwen
tags:lua android

最近在做一个lua的android项目,项目时间仓促,加上对lua完全陌生,于是选了个文档资料最全的lua bridge:luajava

luajava目前只支持到5.1,不过升级到5.2困难不大,主要是一些宏开关以及静态变量;

结果通过lua开始操作android ui时候,频繁掉祯,profile后发现90%+的cpu耗时在luajava中;

掉坑了

看了一下luajava本身代码其实不大;native层就一个luajava.c,java层,除了LuaJavaApi.java,其他java类都是Value Object

#lua逻辑分析
先看lua创建java object的代码,luajava.c中
```C
int javaNewInstance( lua_State * L )
{     
	//some code …..
	method  = ( *javaEnv )->GetStaticMethodID( javaEnv , luajava_api_class , "javaNewInstance" ,"(ILjava/lang/String;)I" );
	 //先调用java层创建javaobject
	javaClassName = ( *javaEnv )->NewStringUTF( javaEnv , className );
	ret = ( *javaEnv )->CallStaticIntMethod( javaEnv , luajava_api_class , method, (jint)stateIndex , javaClassName );
}
```
然后是:

```Java
public static int javaNewInstance(int luaState, String className) throws LuaException {
	 LuaState L = LuaStateFactory.getExistingState(luaState);
	synchronized (L) {
			Class clazz;
			try {
				clazz = Class.forName(className);
			 } catch (ClassNotFoundException e) {
				 throw new LuaException(e);}
			bject ret = getObjInstance(L, clazz);
			L.pushJavaObject(ret);
			return 1;
	}
}
```

getObjInstance的逻辑暂时不论,先看L.pushjavaObject; 最终通过jni调用luajava.c中的 pushJavaObject,没错,是luajava.c中!!!
```Java
int pushJavaObject( lua_State * L , jobject javaObject ){
	//创建userdata略
	/* pushes the __index metamethod */
	lua_pushstring( L , "__index" );
	lua_pushcfunction( L , &objectIndex );   lua_rawset( L , -3 );
	lua_rawset( L , -3 );
}
```
这儿看明白了,luajava创建的javaobject,对应的userdata的__index元方法是objectIndex,
任何方法的调用统一到objectIndex

继续翻代码,最终找到了LuaJavaApi.java中的objectIndex… 是不是特别绕..

luajava为了提高效率,将类创建\方法查找的逻辑全部放在java层,但是类创建/方法查找又需要将结果通过jni放回lua stack中!!

profile中显示70%的耗时在objectIndex,再看一下java层中的objectIndex代码
```Java
public static int objectIndex(int luaState, Object obj, String methodName) throws LuaException {
	 //some code
	 Method[] methods = clazz.getMethods();
	 Method method = null;
	// gets method and arguments
	for (int i = 0; i < methods.length; i++) {
		if (!methods[i].getName().equals(methodName))
			continue;
		 Class[] parameters = methods[i].getParameterTypes();
		 if (parameters.length != top - 1)
		 	continue;
		 boolean okMethod = true;
		for (int j = 0; j < parameters.length; j++) {
			try {
				 objs[j] = compareTypes(L, parameters[j], j + 2);
			} catch (Exception e) {
				okMethod = false;
				break;
			}
		}
	if (okMethod) {
 		method = methods[i];
 	}
 }
```

由于c和lua的stack中没有传递参数个数,加上java和lua的对象类型没有一一对应;

luajava的做法是先通过getmethods获取所有同名方法, 然后compareTypes将method的参数类型逐个取出与lua stack中的参数比较(又是一堆jni操作);

#性能优化
确定原因后,就可以进行性能优化, 

1. 首先,在jni层将常用的method static化
2. 其次,compareTypes之前先确定是否有同名方法,java方法无重载时候跳过compareTypes逻辑

优化之后测试,效率还是远远不能达到产品化的需求;getmethods的cpu耗时还是很高,因此必须做cache;

但是由于前面提到的参数问题,在objectIndex中不能唯一确定方法签名;这儿有两个方法:

1. 牺牲编码效率,lua中调用java method,通过类似jni调用方式,将方法签名一并带进来,objectIndex解析形成正确的方法签名

2. 牺牲java的方法重载,约定java class不做重载方法

博主选的是方法2..原因是项目中交互很多,但是调用的class有限..

做cache的逻辑基本上同java jit,热点代码检测

优化之后,勉强能达到产品化的性能要求;不过由于luajava的机制问题,优化的点基本到头了;

#jnlua
如果对lua c api的android开发者,不建议使用luajava,而使用[jnlua](https://code.google.com/p/jnlua/),

纯jni实现,原理上性能应该会比luajava高一头

jnlua分两个版本,0.9支持lua 5.2,1.0支持lua5.2

0.9已经移植到android平台,1.0暂无(作者在wiki中说明未来会移植)

缺点是文档很少,下篇我再说说jnlua


