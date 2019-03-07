---
title: Java反射机制详解
categories:
  - 后端
tags:
  - Java编程
keywords: Java,Java反射
date: 2018-12-19 02:08:32
---



对于一般的开发者，很少需要直接使用Java反射机制来完成功能开发，但是反射是很多框架譬如 Spring， Mybatis 实现的核心，反射虽小，能量却很大。

本文主要介绍反射相关的概念以及API的使用，关于反射的应用将在下一篇文章中介绍


### 反射的介绍

**反射(Reflection)** 是 Java 在运行时（Run time）可以访问、检测和修改它本身状态或行为的一种能力，它允许运行中的 Java 程序获取自身的信息，并且可以操作类或对象的内部属性。

**Class 类介绍**：Java虚拟机为每个类型管理一个Class对象，包含了与类有关的信息，当通过 javac 编译Java类文件时，生成的同名 .class 文件保存着该类的 Class 对象，JVM 加载一个类即是加载该 .class 文件。



`Class` 和 `java.lang.reflect` 一起对反射提供了支持，java.lang.reflect 包中最常用的几个类的关系如下：

![reflect package](http://image.laijianfeng.org/20181218_182511.png)

其中最主要的三个类 `Field`、`Method` 和 `Constructor` 分别用于描述类的域、方法和构造器，它们有一个共同的父类 `AccessibleObject`，它提供了访问控制检查的功能。

- Field ：描述类的域（属性），可以使用 get() 和 set() 方法读取和修改 Field 对象关联的字段；
- Method ：描述类的方法，可以使用 invoke() 方法调用与 Method 对象关联的方法；
- Constructor ：描述类的构造器，可以用 Constructor 创建新的对象。

下面将通过几个程序来学习Java反射机制。


### 准备两个类用于实验

我们特别定义两个类，Person和Employee，其中Employee继承自Person，且各自都有一个private，protected，public修饰的域（属性），Employee还有private，public修饰的方法

```java
public class Person {
    public String name; // 姓名 公有
    protected String age;   // 年龄 保护
    private String hobby;   // 爱好   私有

    public Person(String name, String age, String hobby) {
        this.name = name;
        this.age = age;
        this.hobby = hobby;
    }
    public String getHobby() {
        return hobby;
    }
}

public class Employee extends Person {
    public static Integer totalNum = 0; // 员工数
    public int empNo;   // 员工编号 公有
    protected String position;  // 职位 保护
    private int salary; // 工资   私有

    public void sayHello() {
        System.out.println(String.format("Hello, 我是 %s, 今年 %s 岁, 爱好是%s, 我目前的工作是%s, 月入%s元\n", name, age, getHobby(), position, salary));
    }
    private void work() {
        System.out.println(String.format("My name is %s, 工作中勿扰.", name));
    }
    public Employee(String name, String age, String hobby, int empNo, String position, int salary) {
        super(name, age, hobby);
        this.empNo = empNo;
        this.position = position;
        this.salary = salary;
        Employee.totalNum++;
    }
}
```

### 获取 Class 对象

获取 Class 对象的方式有三种：使用 Class 类的 forName 静态方法；直接获取某一个对象的 class；调用某个对象的 getClass() 方法

```
public class ClassTest {
    public static void main(String[] args) throws ClassNotFoundException, IllegalAccessException, InstantiationException {
        Class c1 = Class.forName("reflect.Employee");   // 第1种，forName 方式获取Class对象
        Class c2 = Employee.class;      // 第2种，直接通过类获取Class对象
        Employee employee = new Employee("小明", "18", "写代码", 1, "Java攻城狮", 100000);
        Class c3 = employee.getClass();    // 第3种，通过调用对象的getClass()方法获取Class对象

        if (c1 == c2 && c1 == c3) {     // 可以通过 == 比较Class对象是否为同一个对象
            System.out.println("c1、c2、c3 为同一个对象");
            System.out.println(c1);     // class reflect.Employee
        }
    }
}
```

#### 通过反射来创建实例

通过反射来生成对象主要有两种方式

- 使用Class对象的newInstance()方法来创建Class对象对应类的实例
- 先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例

```java
public class NewInstanceTest {
    public static void main(String[] args) throws IllegalAccessException, InstantiationException, NoSuchMethodException, InvocationTargetException {
        Class c = Date.class;
        Date date1 = (Date) c.newInstance();    // 第1种方式：使用Class对象的newInstance()方法来创建Class对象对应类的实例
        System.out.println(date1);      // Wed Dec 19 22:57:16 CST 2018

        long timestamp =date1.getTime();
        Constructor constructor = c.getConstructor(long.class); 
        Date date2 = (Date)constructor.newInstance(timestamp);  // 第2种方式：先通过Class对象获取指定的Constructor对象，再调用Constructor对象的newInstance()方法来创建实例
        System.out.println(date2);  // Wed Dec 19 22:57:16 CST 2018
    }
}
```

### 获取类的全部信息

上面我们定义了两个类，现在有个需求：获取Employee的类名，构造器签名，所有的方法，所有的域（属性）和值，然后打印出来。该通过什么方式来实现呢？

没错，猜对了，就是通过反射来获取这些类的信息，在上面介绍中我们知道JVM虚拟机为每个类型管理一个Class对象，

为了完成我们的需求，我们需要知道一些API如下：

#### 获取类信息的部分API


`String getName()` 获取这个Class的类名


`Constructor[] getDeclaredConstructors()` 返回这个类的所有构造器的对象数组，包含保护和私有的构造器；相近的方法 getConstructors() 则返回这个类的所有**公有**构造器的对象数组，不包含保护和私有的构造器


`Method[] getDeclaredMethods()` 返回这个类或接口的所有方法，包括保护和私有的方法，不包括超类的方法；相近的方法 getMethods() 则返回这个类及其超类的**公有**方法的对象数组，不含保护和私有的方法

`Field[] getDeclaredFields()` 返回这个类的所有域的对象数组，包括保护域和私有域，不包括超类的域；还有一个相近的API `getFields()`，返回这个类及其超类的**公有**域的对象数组，不含保护域和私有域


`int getModifiers()` 返回一个用于描述Field、Method和Constructor的**修饰符**的整形数值，该数值代表的含义可通过Modifier这个类分析

`Modifier 类` 它提供了有关Field、Method和Constructor等的访问修饰符的信息，主要的方法有：toString(int modifiers)返回整形数值modifiers代表的修饰符的字符串；isAbstract是否被abstract修饰；isVolatile是否被volatile修饰；isPrivate是否为private；isProtected是否为protected；isPublic是否为public；isStatic是否为static修饰；等等，见名知义

#### 打印类信息程序

```java
public class ReflectionTest {
    public static void main(String[] args) throws ClassNotFoundException {
        String name;
        if (args.length > 0) {
            name = args[0];
        } else {
            Scanner in = new Scanner(System.in);
            System.out.println("输入一个类名（e.g. java.util.Date）："); // reflect.Employee
            name = in.next();
        }
        try {
            Class cl = Class.forName(name);
            Class superCl = cl.getSuperclass();
            String modifiers = Modifier.toString(cl.getModifiers());
            if (modifiers.length() > 0) {
                System.out.print(modifiers + " ");
            }
            System.out.print("class " + name);
            if (superCl != null && superCl != Object.class) {
                System.out.print(" extends " + superCl.getName());
            }
            System.out.println("\n{");

            printConstructors(cl); // 打印构造方法
            System.out.println();
            printMethods(cl);   // 打印方法
            System.out.println();
            printFields(cl);    // 打印属性
            System.out.println("}");
        } catch (ClassNotFoundException e) {
            e.printStackTrace();
        }
        System.exit(0);
    }

    /**
     * 打印Class对象的所有构造方法
     */
    public static void printConstructors(Class cl) {
        Constructor[] constructors = cl.getDeclaredConstructors();

        for (Constructor c : constructors) {
            String name = c.getName();
            System.out.print("  ");
            String modifiers = Modifier.toString(c.getModifiers());
            if (modifiers.length() > 0) {
                System.out.print(modifiers + " ");
            }
            System.out.print(name + "(");
            // 打印构造参数
            Class[] paramTypes = c.getParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                if (i > 0) {
                    System.out.print(", ");
                }
                System.out.print(paramTypes[i].getName());
            }
            System.out.println(");");
        }
    }

    /**
     * 打印Class的所有方法
     */
    public static void printMethods(Class cl) {
        Method[] methods = cl.getDeclaredMethods();
        //Method[] methods = cl.getMethods();
        for (Method m : methods) {
            Class retType = m.getReturnType();  // 返回类型
            System.out.print("  ");
            String modifiers = Modifier.toString(m.getModifiers());
            if (modifiers.length() > 0) {
                System.out.print(modifiers + " ");
            }
            System.out.print(retType.getName() + " " + m.getName() + "(");
            Class[] paramTypes = m.getParameterTypes();
            for (int i = 0; i < paramTypes.length; i++) {
                if (i > 0) {
                    System.out.print(", ");
                }
                System.out.print(paramTypes[i].getName());
            }
            System.out.println(");");
        }
    }

    /**
     * 打印Class的所有属性
     */
    public static void printFields(Class cl) {
        Field[] fields = cl.getDeclaredFields();
        for (Field f: fields) {
            Class type = f.getType();
            System.out.print("  ");
            String modifiers = Modifier.toString(f.getModifiers());
            if (modifiers.length()> 0) {
                System.out.print(modifiers + " ");
            }
            System.out.println(type.getName() + " " + f.getName() + ";");
        }
    }
}

```

运行程序，然后在控制台输入一个我们想分析的类的全名，譬如 reflect.Employee，可得到下面的输出

```
输入一个类名（e.g. java.util.Date）：
reflect.Employee
public class reflect.Employee extends reflect.Person
{
  private reflect.Employee(java.lang.String, java.lang.String, java.lang.String);
  public reflect.Employee(java.lang.String, java.lang.String, java.lang.String, int, java.lang.String, int);

  public static void main([Ljava.lang.String;);
  public void sayHello();
  private void work();

  public static java.lang.Integer totalNum;
  public int empNo;
  protected java.lang.String position;
  private int salary;
}
```

上面的输出中我们得到的类的构造器，所有方法和所有的域（属性），包括修饰符，名称和参数类型都是准确的，看来反射机制能完成我们的需求。

小结一下，我们通过 getDeclaredConstructors() 获取构造器信息，通过 getDeclaredMethods() 获得方法信息，通过 getDeclaredFields() 获得域信息，再通过 getModifiers() 和 Modifier类 获得修饰符信息，汇总起来就得到了整个类的类信息。

### 运行时查看对象数据域的实际内容

上面我们已经获取到了类的信息，现在又有一个需求：在运行时查看对象的数据域的实际值。这个场景就像我们通过IDEA调试程序，设置断点拦截到程序后，查看某个对象的属性的值。

我们知道java反射机制提供了查看类信息的API，那么它应该也提供了查看Field域实际值和设置Field域实际值的API，没错，猜对了，确实有相关的API，但是有个疑问，有一些属性是private修饰的私有域，这种是否也能直接查看和设置呢？看完下面的API即可知道答案

#### 运行时查看对象数据域实际内容的相关API

`Class<?> getComponentType()` 返回数组类里组件类型的 Class，如果不是数组类则返回null

`boolean isArray()` 返回这个类是否为数组，同类型的API还有 isAnnotation、isAsciiDigit、isEnum、isInstance、isInterface、isLocalClass、isPrimitive 等

`int Array.getLength(obj)` 返回数组对象obj的长度

`Object Array.get(obj, i)` 获取数组对象下标为i的元素

`boolean isPrimitive()` 返回这个类是否为8种基本类型之一，即是否为boolean, byte, char, short, int, long, float, 和double 等原始类型

`Field getField(String name)` 获取指定名称的域对象

`AccessibleObject.setAccessible(fields, true)` 当访问 Field、Method 和 Constructor 的时候Java会执行访问检查，如果访问者没有权限将抛出SecurityException，譬如访问者是无法访问private修饰的域的。通过设置 setAccessible(true) 可以取消Java的执行访问检查，这样访问者就获得了指定 Field、Method 或 Constructor 访问权限

`Class<?> Field.getType()` 返回一个Class 对象，它标识了此 Field 对象所表示字段的声明类型

`Object Field.get(Object obj)` 获取obj对象上当前域对象表示的属性的实际值，获取到的是一个Object对象，实际使用中还需要转换成实际的类型，或者可以通过 getByte()、getChar、getInt() 等直接获取具体类型的值

`void Field.set(Object obj, Object value)` 设置obj对象上当前域表示的属性的实际值


#### 查看对象数据域实际内容程序

了解完上述相关API之后，我们敲出下面的程序来验证

```java
public class ObjectAnalyzer {
    private ArrayList<Object> visited = new ArrayList<>();

    public String toString(Object obj) {
        if (obj == null) {
            return "null";
        }
        if (visited.contains(obj)) {    // 如果该对象已经处理过，则不再处理
            return "...";
        }
        visited.add(obj);

        Class cl = obj.getClass(); // 获取Class对象
        if (cl == String.class) {   // 如果是String类型则直接转为String
            return (String) obj;
        }
        if (cl.isArray()) {        // 如果是数组
            String r = cl.getComponentType() + "[]{\n";     // 数组的元素的类型
            for (int i = 0; i < Array.getLength(obj); i++) {
                if (i > 0) {   // 不是数组的第一个元素加逗号和换行，显示更加美观
                    r += ",\n";
                }
                r += "\t";
                Object val = Array.get(obj, i);
                if (cl.getComponentType().isPrimitive()) { // Class为8种基本类型的时候为 true，直接输出
                    r += val;
                } else {
                    r += toString(val); // 不是8中基本类型时，说明是类，递归调用toString
                }
            }
            return r + "\n}";
        }
        // 既不是String，也不是数组时，输出该对象的类型和属性值
        String r = cl.getName();
        do {
            r += "[";
            Field[] fields = cl.getDeclaredFields();    // 获取该类自己定义的所有域，包括私有的，不包括父类的
            AccessibleObject.setAccessible(fields, true); // 访问私有的属性，需要打开这个设置，否则会报非法访问异常
            for (Field f : fields) {
                if (!Modifier.isStatic(f.getModifiers())) { // 通过 Modifier 可获取该域的修饰符，这里判断是否为 static
                    if (!r.endsWith("[")) {
                        r += ",";
                    }
                    r += f.getName() + "=";     // 域名称
                    try {
                        Class t = f.getType();  // 域（属性）的类型
                        Object val = f.get(obj);   // 获取obj对象上该域的实际值
                        if (t.isPrimitive()) {     // 如果类型为8种基本类型，则直接输出
                            r += val;
                        } else {
                            r += toString(val);     // 不是8种基本类型，递归调用toString
                        }
                    } catch (IllegalAccessException e) {
                        e.printStackTrace();
                    }
                }
            }
            r += "]";
            cl = cl.getSuperclass(); // 继续打印超类的类信息
        } while (cl != null);
        return r;
    }
}
```



#### 测试验证结果

接下来验证一下获取数据域实际值是否正确，分别打印数组、自定义类的对象的实际值

```java
public class ObjectAnalyzerTest {
    public static void main(String[] args) {
        int size = 4;
        ArrayList<Integer> squares = new ArrayList<>(size);
        for (int i = 0; i < size; i++) {
            squares.add(i * i);
        }
        ObjectAnalyzer objectAnalyzer = new ObjectAnalyzer(); // 创建一个上面定义的分析类ObjectAnalyzer的对象
        System.out.println(objectAnalyzer.toString(squares)); // 分析ArrayList<Integer>对象的实际值

        Employee employee = new Employee("小明", "18", "爱好写代码", 1, "Java攻城狮", 100); // 分析自定义类Employee的对象的实际值
        System.out.println(objectAnalyzer.toString(employee));
    }
}
```

输出如下

```java
java.util.ArrayList[elementData=class java.lang.Object[]{
	java.lang.Integer[value=0][][],
	java.lang.Integer[value=1][][],
	java.lang.Integer[value=4][][],
	java.lang.Integer[value=9][][]
},size=4][modCount=4][][]
reflect.Employee[empNo=1,position=Java攻城狮,salary=100][name=小明,age=18,hobby=爱好写代码][]
```

其中`ArrayList<Integer>`打印了类名和5个元素的类型和值，`Employee` 打印了类名，自己定义的3个基本类型的属性的实际值，和父类Person的3个基本类型的属性的实际值

需要注意的是，position，age 是 protected 保护域，salary，hobby 是 private 私有域，Java的安全机制只允许查看任意对象有哪些域，但是不允许读取它们的值

程序中是通过 `AccessibleObject.setAccessible(fields, true)` 将域设置为了可访问，取消了Java的执行访问检查，因此可以访问，如果不加会报异常 IllegalAccessException

小结一下，我们通过 setAccessible(true) 绕过了Java执行访问检查，因此能够访问私有域，通过 Field.getType() 获得了属性的声明类型，通过了 Field.get(Object obj) 获得了该域属性的实际值，还有一个没用上的 Field.set(Object obj, Object value) 设置域属性的实际值


### 调用任意方法

上面我们已经获取了类的构造器，方法，域，查看和设置了域的实际值，那么是不是还可以在调用对象的方法呢？嘿嘿，又猜对了，机智，类的方法信息，获取都获取了，当然就要调用一下，来都来了

上面查看Field的实际值是通过 Field 类的 get() 方法，与之类似，Method 调用方法是通过 Method 类的 invoke 方法 

#### 调用任意方法相关的API

`Method getMethod(String name, Class<?>... parameterTypes)` 获取指定的 Method，参数 name 为要获取的方法名，parameterTypes 为指定方法的参数的 Class，由于可能存在多个同名的重载方法，所以只有提供正确的 parameterTypes 才能准确的获取到指定的 Method

`Object invoke(Object obj, Object... args)` 执行方法，第一个参数执行该方法的对象，如果是static修饰的类方法，则传null即可；后面是传给该方法执行的具体的参数值

#### 调用任意方法程序

```java
public class MethodTableTest {
    public static void main(String[] args) throws NoSuchMethodException, InvocationTargetException, IllegalAccessException {
        Employee employee = new Employee("小明", "18", "写代码", 1, "Java攻城狮", 100000);
        Method sayHello = employee.getClass().getMethod("sayHello");
        System.out.println(sayHello);   // 打印 sayHello 的方法信息
        sayHello.invoke(employee);      // 让 employee 执行 sayHello 方法

        double x = 3.0;
        Method square = MethodTableTest.class.getMethod("square", double.class);  // 获取 MethodTableTest 的square方法
        double y1 = (double) square.invoke(null, x);    // 调用类方法 square 求平方，方法参数 x 
        System.out.printf("square    %-10.4f -> %10.4f%n", x, y1);

        Method sqrt = Math.class.getMethod("sqrt", double.class);   // 获取 Math 的 sqrt 方法
        double y2 = (double) sqrt.invoke(null, x);  // 调用类方法 sqrt 求根，方法参数 x 
        System.out.printf("sqrt      %-10.4f -> %10.4f%n", x, y2);
    }

    // static静态方法 计算乘方
    public static double square(double x) {
        return x * x;
    }
}
```

执行结果

```
public void reflect.Employee.sayHello()
Hello, 我是 小明, 今年 18 岁, 爱好是写代码, 我目前的工作是Java攻城狮, 月入100000元

square    3.0000     ->     9.0000
sqrt      3.0000     ->     1.7321
```

相信大家都看懂啦，通过 getMethod() 获取指定的 Method，再调用 Method.invoke() 执行该方法

### 反射的优缺点

#### 反射的优点：

- **可扩展性** ：应用程序可以利用全限定名创建可扩展对象的实例，来使用来自外部的用户自定义类。

- **类浏览器和可视化开发环境** ：一个类浏览器需要可以枚举类的成员。可视化开发环境（如 IDE）可以从利用反射中可用的类型信息中受益，以帮助程序员编写正确的代码。

- **调试器和测试工具** ： 调试器需要能够检查一个类里的私有成员。测试工具可以利用反射来自动地调用类里定义的可被发现的 API 定义，以确保一组测试中有较高的代码覆盖率。

#### 反射的缺点：

尽管反射非常强大，但也不能滥用。如果一个功能可以不用反射完成，那么最好就不用。在我们使用反射技术时，下面几条内容应该牢记于心。

- **性能开销** ：反射涉及了动态类型的解析，所以 JVM 无法对这些代码进行优化。因此，反射操作的效率要比那些非反射操作低得多。我们应该避免在经常被执行的代码或对性能要求很高的程序中使用反射。

- **安全限制** ：使用反射技术要求程序必须在一个没有安全限制的环境中运行。如果一个程序必须在有安全限制的环境中运行，如 Applet，那么这就是个问题了。

- **内部暴露** ：由于反射允许代码执行一些在正常情况下不被允许的操作（比如访问私有的属性和方法），所以使用反射可能会导致意料之外的副作用，这可能导致代码功能失调并破坏可移植性。反射代码破坏了抽象性，因此当平台发生改变的时候，代码的行为就有可能也随着变化。


> 参考：   
> 《Java核心技术》卷一     
> https://github.com/CyC2018/CS-Notes/blob/master/docs/notes/Java%20%E5%9F%BA%E7%A1%80.md#%E4%B8%83%E5%8F%8D%E5%B0%84   



![关注_小旋锋_微信公众号](http://image.laijianfeng.org/20180913_001328.png)

