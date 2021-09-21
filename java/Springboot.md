# 1 注解和反射

## 1.1 注解（Annotation Type）

### 1.1.1 内置注解

* @Override

  表明该方法重写超类方法

* @Deprecated

  表明不推荐使用该方法

* @SuppreswWarnings

  用来抑制编译是警告信息

### 1.1.2 元注解

* **@Target**

  用于表述注解的使用范围

* **@Retention**

  表明注解的生命周期

* @Document

  表明注解被包含在javadoc

* @Inherited

  表明子类可以继承父类中的该注解

  

  ```java
  //表示注解能够用在什么范围（方法，或者）
  @Target(value = ElementType.METHOD)
  @Retention(value = RetentionPolicy.RUNTIME)
  public @interface MyAnnotation
  ```

### 1.1.3 自定义注解

```java
@interface MyAnnotation2//声明一个接口
    //注解的参数：参数类型+参数名 default 表明默认值
    String name() default "";
	
```





## 1.2 反射（Reflection）

* 反射允许程序在执行期间借助API取得任何类的内部信息，并能直接操作任何对象的内部属性及方法。
* 加载完类后，会在堆内存的方法区产生一个Class类型的对象，其包含完整的类的结构信息。可以通过这个对象看到类的结构。
* 类的链接：在类加载后，将类的二进制数据合并到JRE中
* 类的初始化：JVM对类进行了初始化

### 1.2.1 类的初始化

* 类的主动引用
  * 当虚拟机启动时，先初始化main方法所在的类
  * new一个类的对象
  * 调用类的静态成员（除了final常量）和静态方法
  * 使用java.lang.reflect包的方法对垒进行反射调用
  * 当初始化一个类，如果其父类没有被初始化，先初始化父类
* 类的被动引用
  * 当访问一个静态域时，只有真正声明这个域的类才会被初始化。（因为可以通过类名访问静态变量，如子类通过类名访问父类静态变量，不会导致子类初始化）
  * 通过数组定义引用，不会触发此类的初始化
  * 引用常量不会触发此类的初始化（常量在链接的阶段就存入调用类的常量池中）



### 1.2.2 类加载器的作用

* 类加载的作用：将class文件的字节码内容加载到内存中，并将静态数据结构转换成方法区的运行时数据结构，然后在堆中生成一个代表个类的java.lang.Class对象，作为方法区中类数据的访问。
* 类缓存：标准的JavaSE类加载器可以按要求查找类，但一旦某个类被加载到类加载器中，它将继续加载（缓存）一段时间。不过JVM垃圾回收机制可以回收机制可以回收这些Class对象。
* 源代码（*.java）-> Java 编译器  ->  字节码（*.class0文件）-> 类装载器 -> 字节码效验码 -> 解释器 -> 操作操作平台



### 1.2.3 setAccessbe

* 是启用和禁用访问安全检查的开关。
* true 则知识反射的对象在使用时应取消Java语言访问检查。
  * 提高反射效率，反射会使语言访问检查被频繁调用，设置true 可提高反射效率
  * **使得原本无法访问的私有成员也可以访问**

* 设置false，则会频繁实施Java语言访问检查



### 1.2.4 反射操作泛型

* java采用泛型擦除机制来引入泛型
  * 泛型擦除：java的泛型是针对编译阶段而言，当编译完成后，所有泛型相关的类型全部擦除，即在JVM中是不会对类型进行检查的
* java 新增了ParameterizedType，GenericArrayType，TypeVariable和WildcardType几种类型来代表不能归一到Class类中的类型但是又和原始类型齐名的类型
* ParameterizedType
  * 表示一种参数化类型比如Collection<String>
* GenericArrayType
  * 表示一种元素类型是参数化类型或者类型变量的数组类型

* TypeVariable
  * 是各种类型变量的公共父接口
* WildcardType
  * 是一种通配符类型表达式

### 1.2.5 ORM 对象关系映射









# 2 MySQL



## 2.2 操作数据库

### 2.2.3 创建数据库表

```mysql
CREATE TABLE IF NOT EXISTS `student`(
    `id` INT(4) NOT NULL AUTO_INCREMENT COMMENT '学号',
    `name` VARCHAR(30) NOT NULL DEFAULT '匿名' COMMENT '姓名',
    `pwd` VARCHAR(20) NOT NULL DEFAULT '123456' COMMENT '密码',
    `sex` VARCHAR(2) NOT NULL DEFAULT '女' COMMENT '性别',
    `birtuday` DATETIME DEFAULT NULL COMMENT '出生日期',
    `address` VARCHAR(100) DEFAULT NULL COMMENT '家庭住址',
    `email` VARCHAR(50) DEFAULT NULL COMMENT '邮箱',
    PRIMARY KEY(`id`) 
)ENGINE=INNODB DEFAULT CHARSET=utf8; 

```

 



### 2.2.6 修改和删除表

>  **修改表**

```mysql
-- 修改表明
ALTER TABLE teacher RENAME AS teacher1

-- 增加表名
ALTER TABLE teacher1 ADD age INT(11)

-- 修改表的字段（修改约束）
ALTER TABLE teacher1 MODIFY age VARCHAR(1) --修改表的约束
ALTER TABEL teacher1 CHANGE age age1 INT(1) --字段重命名

-- 删除表的字段
ALTER TABLE teacher1 DROP age1

-- 删除表
DROP TABLE IF EXISTS teacher1
```



## 2.3 MySQL数据管理 

### 2.3.1 外键	

> 定义外键`key` 
>
> 添加外键约束



```mysql

```



### 2.3.2 DML语言

### 2.3.3 添加

### 2.3.4 修改

### 2.3.5 删除



# Mybatis













# Spring

