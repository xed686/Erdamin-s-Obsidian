# Java 入门笔记：目标是"能看懂代码"

> **适合你的背景**：会 Python 和 C，没接触过 Java。
> **学习目标**：不是开发，而是**读得懂 Java 代码**（业务代码、框架配置、反编译出来的代码），为后面的代码审计、反序列化漏洞理解、Android 逆向打基础。
> **阅读方法**：每个知识点都和你熟悉的 Python / C 对比，尽量只讲"读代码时会遇到什么"。

---

## 目录

1. [Java 是什么，怎么运行](#1-java-是什么怎么运行)
2. [第一个程序：先认识代码骨架](#2-第一个程序先认识代码骨架)
3. [变量与数据类型](#3-变量与数据类型)
4. [运算符与流程控制](#4-运算符与流程控制)
5. [字符串、数组、集合](#5-字符串数组集合)
6. [类与对象（Java 的核心）](#6-类与对象java-的核心)
7. [继承、接口、抽象类、多态](#7-继承接口抽象类多态)
8. [访问修饰符、static、final](#8-访问修饰符staticfinal)
9. [异常处理](#9-异常处理)
10. [泛型、Lambda、Stream（现代代码常见写法）](#10-泛型lambdastream现代代码常见写法)
11. [注解（Annotation）：看懂框架代码的关键](#11-注解annotation看懂框架代码的关键)
12. [包、import 与项目结构](#12-包import-与项目结构)
13. [IO、反射、序列化（安全重点）](#13-io反射序列化安全重点)
14. [Web 开发常见代码：Servlet / Spring / JDBC](#14-web-开发常见代码servlet--spring--jdbc)
15. [安全视角：读 Java 代码时盯哪些点](#15-安全视角读-java-代码时盯哪些点)
16. [反编译与逆向入门（含 Android）](#16-反编译与逆向入门含-android)
17. [速查表：Python / C / Java 对照](#17-速查表pythoncjava-对照)
18. [练习建议](#18-练习建议)

---

## 1. Java 是什么，怎么运行

### 1.1 和你已知语言的对比

| | C | Python | Java |
|---|---|---|---|
| 类型 | 静态类型，编译型 | 动态类型，解释型 | **静态类型，先编译成字节码再由虚拟机运行** |
| 内存管理 | 手动 `malloc/free` | 自动（垃圾回收） | **自动（垃圾回收 GC）**，没有指针运算 |
| 面向对象 | 无 | 支持，可选 | **强制：所有代码必须写在类里** |
| 语句结尾 | `;` | 换行 | **`;`** |
| 代码块 | `{}` | 缩进 | **`{}`** |

### 1.2 运行流程

```
Hello.java  --javac 编译-->  Hello.class（字节码）  --java 命令/JVM 运行-->  结果
```

- **JDK**：开发工具包（含编译器 `javac`、运行命令 `java`）。
- **JRE**：运行环境。**JVM**：Java 虚拟机，真正执行字节码的东西。
- **跨平台**：同一个 `.class` 文件可在 Windows / Linux / macOS 上由各自的 JVM 运行。
- **`.jar`**：把很多 `.class` 打包的压缩包（本质是 zip）。**`.war`**：Web 应用包。

> 逆向小知识：`.class` / `.jar` 里是字节码，**非常容易被反编译回接近源码的 Java**（比 C 的二进制好读得多），见第 16 章。

---

## 2. 第一个程序：先认识代码骨架

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello, Java!");
    }
}
```

逐行解释（这也是你读 90% 的 Java 文件时会反复看到的结构）：

| 代码 | 含义 | 对应你熟悉的 |
|------|------|--------------|
| `public class Hello` | 定义一个叫 `Hello` 的类；**文件名必须是 `Hello.java`** | Python 的 `class Hello:` |
| `public static void main(String[] args)` | **程序入口**，JVM 从这里开始执行 | C 的 `int main(int argc, char *argv[])` |
| `String[] args` | 命令行参数数组 | C 的 `argv` |
| `System.out.println(...)` | 打印并换行 | Python 的 `print()` / C 的 `printf` |
| `void` | 方法没有返回值 | C 的 `void` |

**读法**：看到 `public static void main` 就知道这是入口。Web 项目里入口常是 Spring Boot 的启动类（见第 14 章）。

注释写法（和 C 一样）：

```java
// 单行注释
/* 多行注释 */
/** 文档注释，常出现在方法上方 */
```

---

## 3. 变量与数据类型

### 3.1 基本类型（和 C 很像，但大小固定）

| 类型 | 大小 | 例子 | 备注 |
|------|------|------|------|
| `byte` | 1 字节 | `byte b = 10;` | 范围 -128~127，**有符号**（处理二进制/加密时常见 `& 0xFF`） |
| `short` | 2 字节 | `short s = 100;` | 少见 |
| `int` | 4 字节 | `int a = 10;` | 最常用 |
| `long` | 8 字节 | `long n = 100L;` | 数字后加 `L` |
| `float` | 4 字节 | `float f = 1.5f;` | 加 `f` |
| `double` | 8 字节 | `double d = 3.14;` | 默认小数类型 |
| `char` | 2 字节 | `char c = 'A';` | **单引号**，一个 UTF-16 字符 |
| `boolean` | — | `true` / `false` | 不能用 0/1 代替 |

与 C 的区别：Java 里**没有 unsigned**，且不同平台大小一致；`boolean` 不是整数。

### 3.2 引用类型

除了上面 8 个基本类型，其他都是**引用类型**（对象）：`String`、数组、`List`、自定义类……变量里存的是"对象的引用"（类似指针，但不能做指针运算）。

```java
int a = 10;                    // 基本类型：直接存值
String s = "hello";            // 引用类型：s 指向一个字符串对象
String t = null;               // null：不指向任何对象（对 null 调用方法会抛 NullPointerException）
```

### 3.3 声明与类型推断

```java
int x = 5;                      // 必须写类型（和 Python 不同）
final int MAX = 100;            // final：常量，不能再赋值
var list = new ArrayList<String>();   // var：编译器自动推断类型（Java 10+），仍是静态类型
```

### 3.4 类型转换

```java
int i = 10;
double d = i;          // 小→大：自动转换
int j = (int) 3.9;     // 大→小：强制转换，结果 3（直接截断）
String s = String.valueOf(123);      // 数字 → 字符串
int n = Integer.parseInt("123");     // 字符串 → 数字
```

### 3.5 包装类（读代码常遇到）

每个基本类型都有对应的对象类型，集合里只能放对象：

| 基本类型 | 包装类 |
|----------|--------|
| `int` | `Integer` |
| `long` | `Long` |
| `double` | `Double` |
| `boolean` | `Boolean` |
| `char` | `Character` |
| `byte` | `Byte` |

```java
Integer a = 10;       // 自动装箱
int b = a;            // 自动拆箱
```

---

## 4. 运算符与流程控制

几乎和 C 一致：

```java
// 条件
if (score >= 90) {
    ...
} else if (score >= 60) {
    ...
} else {
    ...
}

// 三元
String r = (age >= 18) ? "adult" : "minor";

// switch
switch (role) {
    case "admin":
        ...
        break;
    default:
        ...
}

// 循环
for (int i = 0; i < 10; i++) { ... }
while (cond) { ... }
do { ... } while (cond);

// 增强 for（类似 Python 的 for x in list）
for (String name : names) {
    System.out.println(name);
}
```

### 需要注意的坑

| 现象 | 说明 |
|------|------|
| `==` 比较**引用类型**时比的是"是不是同一个对象" | 比较字符串内容要用 `a.equals(b)`，不是 `a == b` |
| 逻辑运算符 | `&&`、`||`、`!`（没有 Python 的 `and/or/not`） |
| 整数除法 | `5 / 2` 结果是 `2`（和 C 一样，不是 Python 的 2.5） |
| `i++` / `++i` | 和 C 一样 |

---

## 5. 字符串、数组、集合

### 5.1 String

```java
String s = "hello";
s.length();                 // 5
s.charAt(0);                // 'h'
s.substring(1, 3);          // "el"
s.indexOf("l");             // 2
s.equals("hello");          // true  ← 比较内容
s.equalsIgnoreCase("HELLO");
s.contains("ell");
s.split(",");               // 切分成 String[]
s.toUpperCase();
s.trim();
"a" + "b";                  // 拼接
String.format("id=%d, name=%s", 1, "tom");   // 类似 C 的 printf 格式
```

要点：**String 不可变**，上面这些方法都是返回新字符串，不会修改原字符串。大量拼接用 `StringBuilder`：

```java
StringBuilder sb = new StringBuilder();
sb.append("a").append("b");
String result = sb.toString();
```

### 5.2 数组

```java
int[] arr = new int[5];              // 长度 5，默认全 0
int[] arr2 = {1, 2, 3};              // 直接初始化
arr2[0];                             // 访问
arr2.length;                         // 长度（是属性，不是方法，没有括号）
int[][] matrix = new int[3][4];      // 二维数组
```

和 C 的区别：越界会抛 `ArrayIndexOutOfBoundsException`，而不是未定义行为。

### 5.3 集合框架（读业务代码最常见）

| 类型 | 作用 | 类似 Python | 例子 |
|------|------|-------------|------|
| `List` | 有序可重复 | `list` | `ArrayList`、`LinkedList` |
| `Set` | 不重复 | `set` | `HashSet`、`TreeSet` |
| `Map` | 键值对 | `dict` | `HashMap`、`TreeMap` |

```java
List<String> list = new ArrayList<>();
list.add("a");
list.get(0);
list.size();
for (String s : list) { ... }

Map<String, Integer> map = new HashMap<>();
map.put("age", 20);
map.get("age");
map.containsKey("age");
for (Map.Entry<String, Integer> e : map.entrySet()) {
    e.getKey();  e.getValue();
}

Set<String> set = new HashSet<>();
set.add("x");
```

> 读法：`List<String>` 里的 `<String>` 叫**泛型**，表示"元素是 String 类型"（第 10 章）。

---

## 6. 类与对象（Java 的核心）

### 6.1 一个典型的类

```java
public class User {
    // 成员变量（字段）
    private String name;
    private int age;

    // 构造方法（和类同名，没有返回值）
    public User(String name, int age) {
        this.name = name;      // this 类似 Python 的 self
        this.age = age;
    }

    // getter / setter（几乎每个 Java 业务类都有）
    public String getName() { return name; }
    public void setName(String name) { this.name = name; }

    // 普通方法
    public void sayHello() {
        System.out.println("Hi, " + name);
    }
}
```

使用：

```java
User u = new User("tom", 20);    // new 创建对象，调用构造方法
u.sayHello();
u.setName("jerry");
```

### 6.2 对照理解

| Java                | Python        | C                        |
| ------------------- | ------------- | ------------------------ |
| `class User {}`     | `class User:` | `struct User {}`（C 没有方法） |
| `new User(...)`     | `User(...)`   | 手动分配 `malloc`            |
| `this.name`         | `self.name`   | 通过结构体指针 `p->name`        |
| 构造方法                | `__init__`    | 无                        |
| `getXxx()/setXxx()` | 直接访问属性        | 直接访问字段                   |

### 6.3 POJO / Bean / DTO / Entity（常见"名词"）

这些都只是**只有字段 + getter/setter 的普通类**，用来装数据：

- **Entity / PO**：对应数据库表的一行。
- **DTO / VO**：在前后端或各层之间传递数据。
- **Bean**：被框架（如 Spring）管理的对象。

> 看到 `@Data`、`@Getter`、`@Setter`（Lombok 库），说明 getter/setter 是编译时自动生成的，源码里看不到。

### 6.4 方法重载（Overload）

同名方法，参数不同，是不同的方法（Python 没有这种写法）：

```java
public int add(int a, int b) { ... }
public double add(double a, double b) { ... }
public int add(int a, int b, int c) { ... }
```

---

## 7. 继承、接口、抽象类、多态

### 7.1 继承

```java
public class Animal {
    public void eat() { System.out.println("eating"); }
}

public class Dog extends Animal {     // Dog 继承 Animal
    @Override
    public void eat() {               // 重写父类方法
        super.eat();                  // 调用父类方法
        System.out.println("dog eats bones");
    }
}
```

- `extends`：继承（**Java 只允许单继承**）。
- `super`：指代父类。
- `@Override`：声明这是重写父类的方法。

### 7.2 接口（interface）

接口只规定"有哪些方法"，不管怎么实现；类用 `implements` 实现它（可以实现多个接口）：

```java
public interface Shape {
    double area();
}

public class Circle implements Shape {
    private double r;
    public Circle(double r) { this.r = r; }
    @Override
    public double area() { return Math.PI * r * r; }
}
```

> **读代码非常重要的一点**：Java 业务代码大量使用"面向接口编程"，你看到的调用对象类型往往是**接口**，真正执行的是某个**实现类**。读代码时要学会"找实现类"（IDE 里 `Ctrl+Alt+B`，或全局搜索 `implements 接口名`）。

### 7.3 抽象类

```java
public abstract class Base {
    public abstract void run();    // 抽象方法：只有声明，没有方法体
    public void common() { ... }   // 也可以有普通方法
}
```

不能直接 `new Base()`，必须由子类继承并实现抽象方法。

### 7.4 多态

```java
Animal a = new Dog();    // 父类类型的变量，指向子类对象
a.eat();                 // 实际执行的是 Dog 的 eat()
```

**看到的是声明类型，运行的是实际类型**。这也是 Java 反序列化漏洞"利用链"能成立的重要基础之一。

### 7.5 `instanceof` 与类型转换

```java
if (obj instanceof User) {
    User u = (User) obj;     // 强制类型转换
}
```

### 7.6 内部类与匿名类（读老代码时常见）

```java
// 匿名内部类：临时实现一个接口
Runnable r = new Runnable() {
    @Override
    public void run() { System.out.println("run"); }
};
new Thread(r).start();
```

---

## 8. 访问修饰符、static、final

### 8.1 访问修饰符

| 修饰符 | 同类 | 同包 | 子类 | 其他 |
|--------|------|------|------|------|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| （不写，默认） | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

### 8.2 static

```java
public class Util {
    public static int add(int a, int b) { return a + b; }    // 静态方法：属于类，不属于对象
    public static final String VERSION = "1.0";              // 静态常量
}

Util.add(1, 2);      // 直接用类名调用，不需要 new
```

- `static` 成员属于**类**，所有对象共享；非 `static` 成员属于**某个对象**。
- 静态方法里**不能**直接访问非静态成员。
- `static { ... }`：**静态代码块**，类第一次被加载时自动执行一次。反序列化、反射利用里经常见到。

### 8.3 final

| 位置 | 含义 |
|------|------|
| 变量 | 只能赋值一次（常量） |
| 方法 | 不能被子类重写 |
| 类 | 不能被继承（如 `String`） |

### 8.4 常见写法速读

```java
public static final String SECRET = "abc123";   // 公有静态常量——硬编码密钥的典型样子
private static final Logger log = LoggerFactory.getLogger(Xxx.class);   // 日志对象，几乎每个类都有
```

---

## 9. 异常处理

### 9.1 try / catch / finally

```java
try {
    int x = Integer.parseInt("abc");
} catch (NumberFormatException e) {
    e.printStackTrace();          // 打印堆栈
} catch (Exception e) {
    ...
} finally {
    // 无论是否异常都执行（常用于关闭资源）
}
```

### 9.2 throw 与 throws

```java
public void read() throws IOException {      // 声明：这个方法可能抛出 IOException，调用者要处理
    throw new IOException("failed");         // 主动抛出异常
}
```

### 9.3 异常分类

| 类型 | 说明 | 例子 |
|------|------|------|
| **受检异常**（Checked） | 编译器强制要求处理（try/catch 或 throws） | `IOException`、`SQLException` |
| **运行时异常**（Runtime） | 不强制处理 | `NullPointerException`、`ArrayIndexOutOfBoundsException` |

### 9.4 try-with-resources（自动关闭资源）

```java
try (FileInputStream in = new FileInputStream("a.txt")) {
    ...
}    // 括号里的资源会自动关闭
```

> 安全提示：异常信息、堆栈被直接返回给前端（报错页面泄露类名、路径、SQL 语句）是常见的信息泄露点。

---

## 10. 泛型、Lambda、Stream（现代代码常见写法）

### 10.1 泛型

```java
List<String> list = new ArrayList<>();         // 只能放 String
Map<String, List<Integer>> map = new HashMap<>();

// 泛型类 / 方法
public class Box<T> {
    private T value;
    public T get() { return value; }
}
```

读法：`<T>`、`<E>`、`<K, V>` 是"类型占位符"，用的时候替换成具体类型。看到 `<?>` 表示"任意类型"。

### 10.2 Lambda 表达式（类似 Python 的 lambda）

```java
// 传统写法
Runnable r = new Runnable() {
    public void run() { System.out.println("hi"); }
};

// Lambda 写法
Runnable r2 = () -> System.out.println("hi");

// 带参数
Comparator<Integer> c = (a, b) -> a - b;
list.forEach(s -> System.out.println(s));
list.forEach(System.out::println);      // 方法引用，等价于上面一行
```

`->` 左边是参数，右边是方法体。

### 10.3 Stream API（看到一长串 `.` 链式调用就是它）

```java
List<String> result = users.stream()
        .filter(u -> u.getAge() > 18)        // 过滤
        .map(User::getName)                  // 取出名字
        .sorted()                            // 排序
        .collect(Collectors.toList());       // 收集成 List
```

对照 Python：

```python
result = sorted([u.name for u in users if u.age > 18])
```

读法：**从上往下读，一步一步处理数据**。常见操作：`filter`（过滤）、`map`（转换）、`forEach`（遍历）、`collect`（收集结果）、`sum/count`（统计）。

### 10.4 Optional（避免 null）

```java
Optional<User> opt = findUser(id);
opt.ifPresent(u -> ...);
User u = opt.orElse(defaultUser);
```

---

## 11. 注解（Annotation）：看懂框架代码的关键

注解以 `@` 开头，是给**编译器或框架**看的"标记"，自己不执行逻辑，但框架会读取它并执行相应行为。

### 11.1 标准注解

| 注解 | 作用 |
|------|------|
| `@Override` | 声明重写父类方法 |
| `@Deprecated` | 已过时 |
| `@SuppressWarnings` | 压制编译器警告 |
| `@FunctionalInterface` | 声明函数式接口 |

### 11.2 Spring / Web 框架常见注解（读 Web 项目必认）

| 注解 | 含义 |
|------|------|
| `@SpringBootApplication` | Spring Boot **启动类**标记（入口） |
| `@RestController` / `@Controller` | 这是一个**处理 HTTP 请求的控制器** |
| `@RequestMapping("/user")` | 把 URL 路径映射到类或方法 |
| `@GetMapping` / `@PostMapping` | 限定 GET / POST |
| `@RequestParam` | 从 URL 参数 / 表单取值 |
| `@PathVariable` | 从路径里取值（如 `/user/{id}`） |
| `@RequestBody` | 把请求体（JSON）转成对象 |
| `@Service` / `@Repository` / `@Component` | 声明由框架管理的对象（业务层 / 数据层 / 通用） |
| `@Autowired` / `@Resource` | **自动注入**依赖，不需要自己 `new` |
| `@Value("${xxx}")` | 读取配置文件里的值 |
| `@Mapper` / `@Select` / `@Insert` | MyBatis 的数据库映射 |
| `@Entity` / `@Table` / `@Id` | JPA 实体与表映射 |
| `@Data` / `@Getter` / `@Setter` / `@Slf4j` | Lombok：编译期自动生成代码 |

### 11.3 示例：读懂一个 Controller

```java
@RestController
@RequestMapping("/user")
public class UserController {

    @Autowired
    private UserService userService;      // 框架自动注入，不用 new

    @GetMapping("/info")                  // 访问 GET /user/info?id=1
    public User info(@RequestParam("id") Long id) {
        return userService.findById(id);  // 返回的对象自动转成 JSON
    }
}
```

**读法**：先看类上的 `@RequestMapping` 得到前缀 `/user`，再看方法上的 `@GetMapping("/info")` 得到完整路径 `/user/info`，参数来自 `id`，最后追进 `userService.findById`。

> 安全提示：代码审计时，**先从所有 `@RequestMapping / @GetMapping / @PostMapping` 的方法入手，列出所有接口**，这就是"攻击面"。

---

## 12. 包、import 与项目结构

### 12.1 package 与 import

```java
package com.example.demo.controller;     // 当前文件所在的包（对应目录 com/example/demo/controller/）

import java.util.List;                   // 导入某个类
import java.util.*;                      // 导入整个包的所有类
import static java.lang.Math.PI;         // 静态导入
```

- 包名一般是公司域名倒写，如 `com.alibaba.xxx`、`org.apache.xxx`。
- **`java.lang` 包（如 `String`、`System`、`Math`）默认自动导入。**

### 12.2 典型 Maven 项目结构

```
project/
├── pom.xml                      ← 依赖和构建配置（类似 Python 的 requirements.txt）
└── src/
    ├── main/
    │   ├── java/                ← Java 源码（按包名分目录）
    │   │   └── com/example/demo/
    │   │       ├── DemoApplication.java      ← 启动类
    │   │       ├── controller/               ← 接口层
    │   │       ├── service/                  ← 业务层
    │   │       ├── mapper (或 dao)/          ← 数据层
    │   │       └── entity (或 model)/        ← 实体类
    │   └── resources/
    │       ├── application.yml (或 .properties)  ← 配置文件（数据库账号、密钥等）
    │       └── mapper/*.xml                      ← MyBatis 的 SQL
    └── test/                    ← 测试代码
```

### 12.3 pom.xml 怎么读

```xml
<dependency>
    <groupId>org.apache.logging.log4j</groupId>
    <artifactId>log4j-core</artifactId>
    <version>2.14.1</version>
</dependency>
```

每个 `<dependency>` 是一个依赖库，由 `groupId + artifactId + version` 唯一确定。

> 安全提示：
> - **依赖版本**是漏洞判断的关键（如 Log4j 2.14.1、Fastjson 1.2.x、Shiro、Spring4Shell 的受影响版本）。
> - `application.yml` / `application.properties` 里常有数据库密码、密钥、Redis 地址，是审计和信息泄露的重点。
> - 三层调用链：`Controller → Service → Mapper/DAO → 数据库`，读代码顺着这条线走就行。

---

## 13. IO、反射、序列化（安全重点）

这三个概念是 Java 安全（尤其是**反序列化漏洞**）的核心，即使不写代码也要看懂。

### 13.1 文件 IO

```java
File f = new File("/tmp/a.txt");
f.exists();  f.isDirectory();  f.getName();

// 读文件
try (BufferedReader br = new BufferedReader(new FileReader("a.txt"))) {
    String line;
    while ((line = br.readLine()) != null) {
        System.out.println(line);
    }
}

// 字节流
FileInputStream in = new FileInputStream("a.bin");
FileOutputStream out = new FileOutputStream("b.bin");
```

命名规律：

- `...InputStream / ...OutputStream`：**字节流**。
- `...Reader / ...Writer`：**字符流**。
- 外层套 `Buffered...`：加缓冲。

> 安全提示：文件路径由用户输入拼接 → **路径穿越 / 任意文件读取**；用户可控文件名保存上传文件 → **任意文件上传**。

### 13.2 反射（Reflection）

反射：**程序在运行时，根据类名字符串动态获取类、创建对象、调用方法、读写字段**，哪怕方法是 `private`。

```java
Class<?> cls = Class.forName("com.example.User");        // 根据类名字符串加载类
Object obj = cls.getDeclaredConstructor().newInstance(); // 创建对象
Method m = cls.getDeclaredMethod("sayHello");            // 获取方法
m.setAccessible(true);                                   // 突破 private 限制
m.invoke(obj);                                           // 调用方法
Field f = cls.getDeclaredField("name");
f.setAccessible(true);
f.set(obj, "hacker");                                    // 修改字段
```

读法要点：

| API | 作用 |
|-----|------|
| `Class.forName("xxx")` | 通过字符串加载类 |
| `getMethod` / `getDeclaredMethod` | 获取方法（后者包含 private） |
| `setAccessible(true)` | **绕过访问控制** |
| `invoke(obj, args)` | 执行方法 |

> 安全提示：**类名、方法名、参数如果可被用户控制 + 反射调用 = 任意代码执行的潜在入口**。Java 利用链几乎都用反射把"看似无害的调用"串成命令执行（如反射调用 `Runtime.exec`）。

### 13.3 序列化与反序列化

- **序列化**：把内存中的对象转成字节流（便于存文件、网络传输）。
- **反序列化**：把字节流还原成对象。

```java
// 要能被序列化，类必须实现 Serializable 接口
public class User implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private transient String password;     // transient：该字段不参与序列化
}

// 序列化
ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("user.bin"));
oos.writeObject(user);

// 反序列化
ObjectInputStream ois = new ObjectInputStream(new FileInputStream("user.bin"));
User u = (User) ois.readObject();          // ← 危险函数
```

**为什么危险**：反序列化时，JVM 会还原对象并自动调用该类里某些特殊方法（如 `readObject()`）。如果攻击者能控制被反序列化的字节流，就能构造一个"特殊对象"，让 JVM 在还原过程中依次触发一系列已有类的方法（**利用链 / Gadget Chain**），最终执行任意命令。**漏洞本身不在某一行代码，而在"反序列化不可信数据"这件事 + 类路径里存在可利用的类**。

识别要点：

| 现象 | 说明 |
|------|------|
| 数据以 `ac ed 00 05` 开头（十六进制） | 标准 Java 序列化数据的魔数 |
| Base64 后以 `rO0AB` 开头 | 同上，Base64 形式（Cookie、参数里常见） |
| 代码里出现 `ObjectInputStream.readObject()` | 反序列化入口 |
| `XMLDecoder`、`XStream`、`Fastjson`、`Jackson（开启 enableDefaultTyping）`、`Hessian` 等 | 其他形式的"对象反序列化"入口 |

---

## 14. Web 开发常见代码：Servlet / Spring / JDBC

### 14.1 Servlet（最基础的 Java Web 写法，老项目常见）

```java
@WebServlet("/hello")
public class HelloServlet extends HttpServlet {
    @Override
    protected void doGet(HttpServletRequest req, HttpServletResponse resp) throws IOException {
        String name = req.getParameter("name");       // 获取参数（Source）
        resp.setContentType("text/html;charset=utf-8");
        resp.getWriter().write("Hello " + name);      // 输出（Sink，可能导致 XSS）
    }
}
```

| 对象 / 方法 | 作用 |
|-------------|------|
| `HttpServletRequest req` | 请求对象 |
| `req.getParameter("x")` | 取 GET/POST 参数 |
| `req.getHeader("x")` | 取请求头 |
| `req.getCookies()` | 取 Cookie |
| `req.getSession()` | 获取会话 |
| `HttpServletResponse resp` | 响应对象 |
| `resp.getWriter().write()` | 输出内容 |
| `resp.sendRedirect(url)` | 重定向（url 可控 → 开放重定向） |
| `doGet / doPost` | 处理 GET / POST 请求的方法 |
| `web.xml` | 老式的 Servlet / Filter 配置文件 |

### 14.2 Filter 与拦截器

- **Filter**（过滤器）、**Interceptor**（拦截器）在请求到达 Controller **之前**运行，常用于登录校验、权限控制。
- 审计时要看：**哪些路径被放行了**（`excludePathPatterns`、`permitAll`、`anonymous`），以及路径匹配规则是否可被绕过。

### 14.3 JDBC 访问数据库（SQL 注入最直接的地方）

```java
Connection conn = DriverManager.getConnection(url, user, password);

// ❌ 危险：字符串拼接 SQL
Statement st = conn.createStatement();
ResultSet rs = st.executeQuery("select * from user where id = " + id);

// ✅ 安全：预编译，参数用 ? 占位
PreparedStatement ps = conn.prepareStatement("select * from user where id = ?");
ps.setLong(1, id);
ResultSet rs2 = ps.executeQuery();

while (rs2.next()) {
    String name = rs2.getString("name");
}
```

**一眼判断**：用 `Statement` + `+` 拼接 → 高度可疑；用 `PreparedStatement` + `?` → 一般安全。

### 14.4 MyBatis：`#{}` 与 `${}` 的区别（审计必会）

```xml
<!-- 安全：#{} 是预编译参数 -->
<select id="getUser" resultType="User">
    select * from user where id = #{id}
</select>

<!-- 危险：${} 是直接字符串拼接 -->
<select id="getByName" resultType="User">
    select * from user where name = '${name}'
</select>

<!-- ORDER BY、表名、列名无法预编译，常被迫用 ${}，是 SQL 注入高发点 -->
<select id="list">select * from user order by ${sortField}</select>
```

### 14.5 Spring Boot 配置文件示例

```yaml
server:
  port: 8080
spring:
  datasource:
    url: jdbc:mysql://127.0.0.1:3306/test
    username: root
    password: 123456          # 硬编码密码，审计重点
management:
  endpoints:
    web:
      exposure:
        include: "*"          # 暴露所有 Actuator 端点，可能泄露信息
```

---

## 15. 安全视角：读 Java 代码时盯哪些点

### 15.1 核心思路（和前端是同一个模型）

> **Source（用户可控输入）→ 中间处理 → Sink（危险函数）**
> 看有没有过滤、有没有校验、有没有权限控制。

### 15.2 常见 Source

| 来源 | 代码 |
|------|------|
| URL / 表单参数 | `request.getParameter()`、`@RequestParam`、`@PathVariable` |
| 请求体 | `@RequestBody`、`request.getInputStream()` |
| 请求头 / Cookie | `request.getHeader()`、`request.getCookies()` |
| 上传文件 | `MultipartFile`（`getOriginalFilename()` 文件名也是可控的） |
| 数据库读出的数据 | 存储型漏洞的二次来源 |

### 15.3 常见 Sink 速查表

| 漏洞类型 | 危险 API / 特征 |
|----------|------------------|
| **命令执行** | `Runtime.getRuntime().exec()`、`ProcessBuilder`、`ScriptEngine.eval()`、`GroovyShell.evaluate()` |
| **SQL 注入** | `Statement` + 字符串拼接、MyBatis 的 `${}`、`createQuery("..." + x)`（Hibernate 的 HQL 拼接） |
| **反序列化** | `ObjectInputStream.readObject()`、`XMLDecoder.readObject()`、`XStream.fromXML()`、Fastjson `JSON.parse/parseObject`（旧版本）、`Yaml.load()`（SnakeYAML 旧用法） |
| **XXE** | `DocumentBuilderFactory`、`SAXParser`、`XMLReader`、`SAXReader` 未禁用外部实体 |
| **SSRF** | `new URL(x).openConnection()`、`HttpURLConnection`、`HttpClient`、`RestTemplate` 的 URL 可控 |
| **任意文件读写 / 路径穿越** | `new File(用户输入)`、`FileInputStream`、`Files.readAllBytes(Paths.get(x))`、`MultipartFile.transferTo()` |
| **文件上传** | 未校验后缀 / 内容类型 / 保存路径，直接用原文件名 |
| **JNDI 注入 / 远程类加载** | `InitialContext.lookup(用户输入)`、Log4j2 的 `${jndi:...}` |
| **表达式注入（EL / SpEL / OGNL）** | `SpelExpressionParser.parseExpression(x)`、`Ognl.getValue`、Struts2 |
| **模板注入** | Freemarker、Velocity、Thymeleaf 的模板内容可控 |
| **反射滥用** | `Class.forName(x)`、`Method.invoke` 参数可控 |
| **开放重定向** | `response.sendRedirect(x)` |
| **XSS** | 直接把参数写进响应，不做编码；模板里使用 `th:utext`、`<%= %>`、`${}` 不转义输出 |
| **越权（IDOR）** | 通过 `id` 查询数据，**没有校验该数据是否属于当前用户** |
| **认证 / 密钥问题** | 硬编码密码、弱加密（`MD5`、`DES`、固定 key 的 `AES`）、`Random` 代替 `SecureRandom`、JWT 密钥写死 |
| **信息泄露** | `e.printStackTrace()` 返回前端、Actuator / Swagger 未授权、调试接口 |

### 15.4 一个读代码的示范

```java
@GetMapping("/download")
public void download(@RequestParam String file, HttpServletResponse resp) throws IOException {
    File f = new File("/data/files/" + file);              // ① file 来自用户，直接拼接路径
    FileInputStream in = new FileInputStream(f);           // ② 打开文件
    IOUtils.copy(in, resp.getOutputStream());              // ③ 写回响应
}
```

分析：`file` 是 Source；`new File("/data/files/" + file)` 没有做规范化与校验；`file=../../etc/passwd` 即可穿越目录 → **任意文件读取**。
修复：校验文件名白名单；`getCanonicalPath()` 后确认仍在基目录内；不允许包含 `..`、`/`。

### 15.5 常见的"已知漏洞组件"（看到版本号要警觉）

Log4j2（Log4Shell）、Fastjson、Shiro（RememberMe 反序列化）、Spring 系列（Spring4Shell、SpEL 相关）、Struts2、XStream、Jackson 反序列化、WebLogic、Tomcat（文件包含类）。
做法：**先看 `pom.xml` / `lib` 目录里的依赖和版本**，再对照公开漏洞信息。

---

## 16. 反编译与逆向入门（含 Android）

### 16.1 Java 字节码特点

`.class` 文件保留了类名、方法名、字段名、泛型等大量结构信息（除非被混淆），所以反编译后几乎可以还原为可读的 Java。

### 16.2 常用反编译工具

| 工具 | 说明 |
|------|------|
| **JD-GUI** | 经典，打开 `.jar` 直接看源码 |
| **Bytecode Viewer** | 集成多个反编译器，可对比结果 |
| **CFR / Fernflower / Procyon** | 命令行或 IDE 内置的反编译引擎 |
| **IntelliJ IDEA** | 内置 Fernflower，直接点击 `.class` 即可查看 |
| **jadx** | **Android 的首选**：直接把 `.apk` / `.dex` 反编译成 Java |

命令示例：

```bash
javap -c Hello.class      # 查看字节码指令（JDK 自带）
jar tf app.jar            # 列出 jar 里的文件
unzip app.jar -d out/     # jar 本质是 zip，可以直接解压
```

### 16.3 混淆会让代码长什么样

```java
public class a {
    private b c;
    public void a(String d) { ... }
}
```

类名、方法名、变量名被替换为 `a`、`b`、`c`（ProGuard / R8），**结构仍在，语义被抹掉**。读法：看**字符串常量、调用的系统 API、调用关系**来推断功能。

### 16.4 Android 与 Java 的关系（为后续 Android 逆向铺路）

- Android 应用主要用 Java（或 Kotlin）编写，编译成 **`.dex`**（Dalvik 字节码，不是 `.class`，但 jadx 能转回 Java）。
- 一个 APK 里的关键内容：

| 文件 / 目录 | 作用 |
|-------------|------|
| `AndroidManifest.xml` | 声明组件（Activity、Service、Receiver）、权限、入口 |
| `classes.dex` | 代码 |
| `res/`、`assets/` | 资源 / 原始资源 |
| `lib/` | `.so` 本地库（C/C++ 写的，**这里用到你的 C 基础**） |

- 常见 Android 概念（读 jadx 反编译出的代码时会遇到）：

| 名称 | 含义 |
|------|------|
| `Activity` | 一个界面 |
| `Intent` | 组件间传递消息 / 启动组件 |
| `onCreate()` | Activity 创建时的入口方法 |
| `WebView` | 内嵌网页（和你前端 JS 知识衔接） |
| `System.loadLibrary("xxx")` | 加载 `.so` 本地库 |
| `native` 关键字 | 方法在 C/C++ 里实现 |

```java
public class MainActivity extends AppCompatActivity {
    static { System.loadLibrary("native-lib"); }           // 加载 so
    public native String getKey();                          // 实际逻辑在 C 代码里
    @Override
    protected void onCreate(Bundle savedInstanceState) {   // 入口
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
    }
}
```

---

## 17. 速查表：Python / C / Java 对照

| 概念 | Python | C | Java |
|------|--------|---|------|
| 打印 | `print(x)` | `printf("%d", x)` | `System.out.println(x)` |
| 变量 | `a = 1` | `int a = 1;` | `int a = 1;` |
| 字符串 | `"abc"` | `char*`/数组 | `String s = "abc";` |
| 字符串比较 | `a == b` | `strcmp(a,b)==0` | `a.equals(b)` |
| 列表 | `[1,2,3]` | 数组 | `List<Integer>` / `int[]` |
| 字典 | `{"a":1}` | 无（结构体/自实现） | `Map<String,Integer>` |
| 遍历 | `for x in xs:` | `for(i=0;i<n;i++)` | `for (T x : xs)` |
| 函数 | `def f(a):` | `int f(int a)` | `public int f(int a)`（必须在类里） |
| 类 | `class A:` | `struct` | `class A {}` |
| 对象自身 | `self` | — | `this` |
| 继承 | `class B(A):` | — | `class B extends A` |
| 接口 | 无（鸭子类型） | 无 | `interface` + `implements` |
| 异常 | `try/except` | 返回码 | `try/catch` |
| 空值 | `None` | `NULL` | `null` |
| 导入 | `import os` | `#include` | `import java.util.List;` |
| 入口 | `if __name__ == "__main__":` | `int main()` | `public static void main(String[] args)` |
| 包管理 | `pip` / `requirements.txt` | 无统一 | Maven `pom.xml` / Gradle |
| 内存 | 自动 | 手动 | 自动（GC） |
| 类型 | 动态 | 静态 | 静态 |

---

## 18. 练习建议

### 18.1 目标明确：读代码，不是写代码

| 阶段 | 做什么 | 完成标志 |
|------|--------|----------|
| 1 | 通读第 2~10 章，在在线环境里把示例代码跑一遍（可用 JDoodle、OnlineGDB、Replit 等在线 Java 运行环境，或本机装 JDK） | 能看懂简单的类、循环、集合、异常 |
| 2 | 读第 11~14 章，在 GitHub 上找一个小型 Spring Boot 的 CRUD 示例项目，**只读不写**：找启动类 → Controller → Service → Mapper | 能说出一个请求从 URL 到数据库的完整路径 |
| 3 | 读第 13、15 章，对照 Sink 表在示例项目里"搜"危险函数 | 能找出示例项目中的拼接 SQL / 路径拼接 |
| 4 | 找一个故意有漏洞的 Java 靶场项目做代码审计 | 能写出"Source → Sink"分析 |
| 5 | 用 jadx 打开一个自己编译的简单 APK（或开源 Demo APK） | 能找到 `AndroidManifest.xml` 里的入口 Activity |

### 18.2 推荐资源（英文 / 国际）

- **Oracle Java Tutorials**（`docs.oracle.com/javase/tutorial`）：官方入门教程。
- **Baeldung**：大量 Java / Spring 的英文教程，带代码示例，适合"查某个概念"。
- **OWASP WebGoat**：用 Java 写的故意有漏洞的教学项目，**既练漏洞又能练读 Java 代码**。
- **OWASP Cheat Sheet Series**：Deserialization、SQL Injection Prevention、XXE Prevention 等的 Java 修复示例。
- **PortSwigger Web Security Academy** 的 Insecure Deserialization 专题（含 Java 部分）。
- **ysoserial**（GitHub，Java 反序列化利用链研究工具）：读它的源码可以理解利用链概念，**仅在自己的靶场环境研究**。
- **jadx**（GitHub）：Android 反编译工具。

### 18.3 自检清单

- [ ] 我能认出程序入口（`main` 方法 / `@SpringBootApplication`）。
- [ ] 我能读懂一个带 getter/setter 的实体类。
- [ ] 我能区分 `class`、`interface`、`abstract class`，并知道遇到接口要去找实现类。
- [ ] 我能读懂 `List<String>`、`Map<K,V>`、`stream().filter().map().collect()`。
- [ ] 我能说出 `@RestController`、`@RequestMapping`、`@Autowired` 各自的作用。
- [ ] 我能说出 `Statement` 与 `PreparedStatement`、`#{}` 与 `${}` 的安全差别。
- [ ] 我能说出反射、序列化、反序列化各自是什么，以及为什么 `readObject` 危险。
- [ ] 我能打开 `pom.xml`，看懂依赖，并想到去核对依赖版本是否有已知漏洞。

---

## 一页总结

1. **Java = 静态类型 + 面向对象 + 编译成字节码在 JVM 上运行**，所有代码都在类里，入口是 `main`。
2. 读 Java 的捷径：**包 → 类 → 注解 → 方法**；Web 项目顺着 **Controller → Service → Mapper → 数据库** 读。
3. 看到 **接口** 就去找实现类；看到 **注解** 就想"框架在背后做了什么"。
4. 安全审计的套路和前端一样：**Source → Sink**；Java 特有的重点是 **反序列化、反射、JNDI、表达式注入、SQL 拼接、XXE、依赖版本**。
5. Java 字节码很容易反编译，这对 **jar 分析** 和 **Android 逆向** 都是优势。
6. 始终在**授权范围**内做测试和研究。
