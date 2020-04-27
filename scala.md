# scala
## print
```scala
var name = "abc"
var age = 10
println(s"Hello,scala name:$name age:${age + 10}")
```
## scala变量声明
声明变量可以指定类型，也可以省略
scala是强类型语言，确定类型后不可以改变

## scala里所有数据类型都是对象 没有java里的原生类型
- scala数据类型分为AnyVal和AnyRef两大类
- Unit等价于void
- Nothing任何类型的子类型
- Null可以赋值给引用类型，不能赋值给值类型（Int...)
- Any所有其他类的超类
- AnyRef所有引用类的基类
- Int类型4个字节，范围-(2^31+1) ---- 2^31 (-0)
- Float 四字节 单精度浮点数
- Double 8字节 双精度
- char 2字节

```scala
var f:Double = 1.5f //float低精度赋值给double高精度没问题
var d:Float = 1.5 //默认为double，高精度赋值给低精度float不可以
var c:Char = 'a' + 1//错误
val st = "12.5"
//println(st.toInt) //error
println(st.toDouble.toInt)
```

## var和val
var可变 val不可变
为什么这么设计
val不可变，没有线程安全，效率高，推荐使用val
val修饰的变量实际上是加了final
```scala
val dog = new Dog
dog.age = 10;//能不能修改属性的值取决于age属性是用var还是val修饰
```

## 标示符命名规则
```text
Scala中的标识符声明，基本和Java是一致的，但是细节上会有所变化。
首字符为字母，后续字符任意字母和数字，美元符号，可后接下划线_
数字不可以开头。
首字符为操作符(比如+ - * / )，后续字符也需跟操作符 ,至少一个(反编译)
操作符(比如+-*/)不能在标识符中间和最后.
用反引号`....`包括的任意字符串，即使是关键字(39个)也可以 [true]

hello    // ok
hello12 // ok
1hello  // error
h-b   // error
x h   // error
h_4   // ok
_ab   // ok
Int    // ok, 在scala中，Int 不是关键字，而是预定义标识符,可以用，但是不推荐
Float  // ok
_   // 不可以，因为在scala中，_ 有很多其他的作用，因此不能使用
Abc    // ok
+*-   // ok
+a  // error
```
## 运算符
```scala
// a % b = a - (a / b) * b
println(-10 % 3) //-1
println( 10 % -3) //1
//scala中没有"++"和"--"运算符

//scala没有三元运算符，因为scala的if和else有返回值
val result = if(flag) 1 else 0
```
## 循环
scala中没有break和continue
```scala
breakable{
  for(i <- Range(1,100,2)) {
    if(i == 50){
      break()
    }
  }
}
```

## 惰性函数
```scala
lazy val res = sum(10,20) //lazy 不能修饰var变量
```

## 构造函数
```scala
class A(name:String){}
class A1(val name:String){}//name 只读属性
class A2(var name:String){}//name 读写属性
```

## BeanProperty
生成get/set方法
```scala
@BeanProperty val name:String = _
```
## 辅助构造器必须先调用类构造器

## 访问修饰符
- private修饰 只有类内部和他的伴生对象能访问
- 默认 属性和方法都是public
- protected 只有子类能访问 同包不能
- 没有public

## 隐式转换
```scala
implicit def doubleToInt(d: Double): Int = d.toInt
val num: Int = 3.5
```

## sealed
密封类 样例类的超类声明时加上sealed，该样例类只允许在声明文件里被使用

## 偏函数
map函数不支持偏函数，collect支持
```scala
object PartialFunDemo {

  def main(args: Array[String]): Unit = {
    val list = List(1, 2, 3, 5, "abc")
    println(list.collect(partialFun))
    println(list.collect { case i: Int => i + 2 })//简化
  }

  val partialFun = new PartialFunction[Any,Int] {
    override def isDefinedAt(x: Any): Boolean = x.isInstanceOf[Int]

    override def apply(v1: Any): Int = v1.asInstanceOf[Int] + 1
  }
}
```