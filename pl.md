<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Java
## Java 8: Interfaces
We will take the following interface as an example for this section: (java8)
### functional interface, `default` and `static`
```java
@FunctionalInterface
interface Square{
  String NAME = "greatSquare";
  int calculate(int x);
  default int calculateFromString(String s){
    return this.calculate(Integer.parseInt(s));
  }
  static void printName(){
    System.out.println("I'm Square interface named" + Square.NAME);
  }
}
```
### functional interface
`Square` definition ");
  }
}
```
allows to get a concrete implementation of Square interface by implementing its `calculate `abstract method like follow:
```java
Square sq = (int x) -> x*x;
```

In fact `@FunctionalInterface` is here to guarantee that the functional interface **has only one abstract method**.

#### `default` method in interface vs abstract method in abstract classes ?
 [(post)](https://stackoverflow.com/a/19998827/6580080)
 
- When doubting, use interface with defaults method in priority, because it has more constraints leading to more compiler optimizations
- Interfaces with default methods can be used as block of behaviors that can enrich a class, because one can only extend one (abstract) class but implement as many interfaces as needed ! Usage can be made in the style of *Scala Mixins*:

```java
interface Printer {  
  default void print(String content) {  
    System.out.println(content);  
  }  
}  
  
interface Named {  
  default String getName() {  
    return this.toString();  
  }  
}  
  
class NamePrinter implements Named, Printer {  
  void printName() {  
    this.print(this.getName());  
  }  
}
```

Use interface whenever you can, it's like a **DIP** applied to library writing instead of client writing.
#### `static` methods
Unlike for (abstract) class
On interfaces,  `static` method in interfaces cannot be accessed through instancescan only:
- `sq.printName();` does not compile
- `Square.printName();` compiles

### interface attributes
Interface attributes cannot receive any modifier and are by default `public static final`. Like for methods, *staticity* in interface attribute implies that it cannot be reached from an instance:
- `sq.NAME;` does not compile
- `Square.NAME;` compiles
Note: both this 2 lines would have compiled if `Square ` were an *abstract class*.
### Diamond problem

# Scala
## Constructors parameattributers scope
```scala
(case) class C(... a: Int)
```

|Visibility |	Accessor? |	Mutator?|
|:--:|:--:|:--:|
|var |	Yes |	Yes|
|val| 	Yes |	No|
|Default visibility (no var or val) 	|No for classes, Yes for case classes| 	No|
|Adding the private keyword to var or val |	No |	No|

## Equals
`==` is infix alias for `equals(other: Any): Boolean` method
```scala
val g = 1
val f = new AnyRef {
 override def equals(other: Any) = true
}
f.equals(g)   true: Boolean
f==g          true: Boolean
g==f          false: Boolean
```
## Closures
This is a closure:
The variable *a* in *f* in captured by f and the modifications of a are influencing f executions.
```scala
var a = new StringBuilder
a.append("a")
val f = ()=>println(a)
f()
a = new StringBuilder
a.append("b")
f()
```
Result:
```
a
b
```
This is not a closure: i is copied to *g* scope and the closure it returns contains only a copy of the StringBuilder, so modifying the original StringBuilder variable from the outer scope doesn't influence closure returned by *g*.

```scala
var a = new StringBuilder
a.append("a")
val g = (i:StringBuilder)=> (()=>{println(i)})
val f = g(a)
f()
a = new StringBuilder
a.append("b")
f()
```
Result:
```
a
a
```
**BUT**: as *a* is an AnyRef, the copy is just a copy of the reference and thus, if the object itself (not the variable *a*) is modified, it will influence the execution of the closure returned by *g*.
```scala
var a = new StringBuilder
a.append("a")
val g = (i:StringBuilder)=> (()=>{println(i)})
val f = g(a)
f()
val b = a
a = new StringBuilder
a.append("b")
f()
b.append("c")
f()
```
Result:
```
a
a
ac
```
**CONCLUSION:**
- AnyVal passed to function as argument *x*: *x* is a val containing the value (in bytecode it's a primitive) of the AnyVal object.
- AnyRef passed to function as argument *x*: *x* is a val containing the reference to the AnyRef object.
- AnyVal or AnyRef catched by a closure through variable *a*: the closure is directly linked to the variable of the outer scope if it is reachable. 

NOTE: for the last case, if the variable is disapearing from scope, last reference contained in the variable is used (this last reference is not a copy of the object and can be operated on by another owner of the same ref):

```scala
var b = new StringBuilder()
val f = () =>
{
  var a = new StringBuilder().append("1")
  val g = () => println(a)
  a = new StringBuilder().append("2")
  g()  // 2
  a = new StringBuilder().append("3")
  g()  // 3
  b = a
  g
}

val gReturned = f()
gReturned()  // 3
b.append("4")
gReturned()  // 34
```
Returns
```
2
3
3
34
```

## (Idem for Java, Python) String equality
String **literals** *value equality* is the same as *reference equality* because string literals are interned by compiler for memory (shared space) and comparison time (need only location comparison instead of comparing every char) optimisation
## Implicit conversions
[[...]](https://stackoverflow.com/questions/6131309/scala-arrays-vs-vectors)Expanding on Kevin's answer and explaining why it's not possibly for _scaladoc_ to tell you what implicit conversion exists: implicit conversions only come into play when your code would not compile otherwise.
You can see it as an error recovery mechanism for type errors that is activated during compilation.
## Partial application
```scala
val f = (x: Int, n: Int) => Math.pow(x, n).toInt
val square = f(_: Int, 2)
square(3)
```
## Pass collection as arguments in variable numbes:
```scala
object A{
	def b(s: String*) = s.foreach(println)
}
// Unpacking Transversable into separated args:
A.b(Seq("a", "b", "c"): _*)
```

## Blocks
```scala
val a = {println(5); i: Int => println(i); i*2}
```
 - `a` is the result of the evaluation of the block = a function typed as ```Int => Int``` and that is impure because it prints its input `i: Int`
 - This line execution will only print '5'
 - `a(1)` only prints `1` and returns 2


**Block is evaluated** even if directly passed to **a function inputing another function**:
```scala
object Test extends App{
  def b(f: Int => Unit): Unit = {  
    println("start")  
    f(1)  
    println("end")  
  }  
  b({println(5); i: Int => println(i)})
}

>>> 5
>>> start
>>> 1
>>> end
```

To **delay** this evaluation, we use lazy evaluation with **calls-by-name** parameters:

```scala
object Test extends App{
  def b(f: => Int => Unit): Unit = {  
    println("start")  
    f(1)  
    println("end")  
  }  
  b({println(5); i: Int => println(i)})
}

>>> start
>>> 5
>>> 1
>>> end
```

## Final class members
Difference between `val const` and `final val const` ?
`final` members **cannot be overridden**, say, in a sub-class or trait.

## isInstanceOf vs isInstance
For reference types (those that extend `AnyRef`), there is no difference in the end result. `isInstanceOf` is however much encouraged, because it's more idiomatic (and likely much more efficient).

For primitive value types, such as numbers and booleans, there is a difference:

```scala
scala> val x: Any = 5
x: Any = 5

scala> x.isInstanceOf[Int]
res0: Boolean = true

scala> classOf[Int].isInstance(x)
res1: Boolean = false
```

That is because primitive value types are boxed when upcasted to `Any` (or `AnyVal`). For the JVM, they appear as `java.lang.Integer`s, not as `Int`s anymore. The `isInstanceOf` knows about this and will do the right thing.

In general, there is no reason to use `classOf[T].isInstance(x)` if you know right then what `T` is. If you have a generic type, and you want to "pass around" a way to test whether a value is of that class (would pass the `isInstanceOf` test), you can use a `scala.reflect.ClassTag` instead. `ClassTag`s are a bit like `Class`es, except they know about Scala boxing and a few other things. (I can elaborate on `ClassTag`s if required, but you should find info about them elsewhere.)

## Self-Type
```scala
case class Word(letters: String)

trait Splittable {
  this: Word =>  
  // all the `Word` trait or class attributes and methods are accessible 
  // through `this` even if `Splittable` does not iherit from `Word`
  def split(by: String): Seq[String] = this.letters.split(by).toSeq
}

val word = new Word("Hello World!") with Splittable

println(word.split(" "))  // prints "ArraySeq(Hello, World!)"
```
Here the trait `Splittable` **is forced to be mixed with** `Word`.
`Splittable`'s purpose is reduced to the extension of a `Word`'s behavior.
Allows to implement a nice *Decorator Pattern* in a functional style.


# Python
## multithreading 
Mainly there is:
- a concurrency lib: `threading` (only one interpreter)
- a true parallel lib: `multiprocessing`

### `threading` lib
#### Critical section
*ex: accessing or updating a shared state*

```python
import threading
class CustomThread(threading.Thread):
	# static class field
	shared_lock = threading.Lock()
    
	def run(self):
    	# do not critical things
        
        with self.shared_lock:
        	# lock taken, do critical things
        
        # lock released, do not critical things
```

## for-else, break, continue
**break**: end loop

**for ...: ... else:...** : else block is only executed if *break* not called

**continue**: ignore rest of the loop's block and jump to next loop iteration

### repr, str
In notebook cell
```python
class A:
    def __repr__(self):
        return "repr"
#   def __str__(self):
#       return "str"
print(A())
A()
```
outs:
```
repr
repr
```
```python
class A:
    def __repr__(self):
        return "repr"
    def __str__(self):
        return "str"
print(A())
A()
```
outs:
```
str
repr
```
--> str is dominant over repr for prints, but cell always output last expression repr

### Usefull imports for OOP
```python
from overrides import overrides  # decorator '@overrides' 
from abc import ABC, abstractmethod  #  'class C(ABC)' is abstract and decorator '@abstractmethod' usable.
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbMTk4NTEzNTMxLC0xNTMyNjk3OTY5LC0xMT
E2NTQ4MDA5LC01NTI1MjYxODgsLTE1OTI5ODM0NjMsLTE4MDIx
NjgyXX0=
-->