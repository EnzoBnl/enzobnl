<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->
# General
## Idiomatic coding
A code snippet is said *idiomatic* if it uses its programming language **the way the language is intended to be used**.

Programming using only idioms ensures you a certain level of :
- **performance**: all the benchmarks are by default based on idiomatic ways to perform tasks using the language.
- **readability**: your code will be more easily intelligible by the rest of the community because the set of idioms represents a standard.
- 
# Java
## `&&` and `||` vs `&` and `|`
When the boolean operator is doubled, it means that its evaluation will be **lazy**:

- Given that `a` evaluates to `false`: `a && b` will not trigger the evaluation of `b` but `a & b` will.
- Given that `a` evaluates to `trur`: `a || b` will not trigger the evaluation of `b` but `a | b` will.

## Modifiers order
from [open jdk guidelines](http://cr.openjdk.java.net/~alundblad/styleguide/index-v6.html):
```java
public / private / protected
abstract
static
final
transient
volatile
**default**
synchronized
native
strictfp
```

## `-Xms` & `-Xmx`
Passing the following arguments to the JVM:
```bash
java -Xms256m -Xmx2048m
```
will reserve an initial amount of memory of 256 MB for JVM's heap consumption and allow it to use up to 2048 MB during its runtime.

## Java 8: Interfaces
We will take the following interface as an example for this section:
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
`Square` definition allows to get a concrete implementation of Square interface by implementing its `calculate `abstract method like follow:
```java
Square sq = (int x) -> x*x;
```

In fact `@FunctionalInterface` is here to guarantee that the functional interface **has only one abstract method**.

### Making a lambda Serializable
At instantiation time, you can use a union cast to make your lambda serializable (`Serializable` is an empty interface):
```java
(Runnable & Serializable) () -> System.out.println("runnable is running")
```

### `default` method in interface vs abstract method in abstract classes ?
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

### `static` methods
Unlike for (abstract) classes,  `static` method in interfaces cannot be accessed through instances:
- `sq.printName();` does not compile
- `Square.printName();` compiles

### interface attributes
Interface attributes cannot receive any modifier and are by default `public static final`. Like for methods, *staticity* in interface attribute implies that it cannot be reached from an instance:
- `sq.NAME;` does not compile
- `Square.NAME;` compiles

### `++`

Given `counter` a numeric primitive (`int`, ...) or an instance of a corresponding boxed type (`Integer`, ...), the statement 

```java
return ++counter;
```

is a syntax sugar for

```java
counter += 1;
return counter;
```

and

```java
return counter++;
```

is a syntax sugar for

```java
counter += 1;
return counter - 1;
```
### Getting objects names `getName` vs `getCanonicalName` vs `getSimpleName` vs `getTypeName`
Source: [SO post by Nick Holts](https://stackoverflow.com/a/15203417/6580080)

||`getName` | `getCanonicalName` | `getSimpleName` | `getTypeName`|
|--|--|--|--|--|
|`int.class (primitive)`|`int`|`int`|`int`|`int`|
|`String.class` (ordinary class)|`java.lang.String`|`java.lang.String`|`String`|`java.lang.String`|
|`java.util.HashMap.SimpleEntry.class` (nested class)|`java.util.AbstractMap$SimpleEntry`|`java.util.AbstractMap.SimpleEntry`|`SimpleEntry`|`java.util.AbstractMap$SimpleEntry`|
|`new java.io.Serializable(){}.getClass()` (anonymous inner class)|`ClassNameTest$1`|`null`|``|`ClassNameTest$1`|
```
int.class (primitive):
    getName():          int
    getCanonicalName(): int
    getSimpleName():    int
    getTypeName():      int

String.class (ordinary class):
    getName():          java.lang.String
    getCanonicalName(): java.lang.String
    getSimpleName():    String
    getTypeName():      java.lang.String

java.util.HashMap.SimpleEntry.class (nested class):
    getName():          java.util.AbstractMap$SimpleEntry
    getCanonicalName(): java.util.AbstractMap.SimpleEntry
    getSimpleName():    SimpleEntry
    getTypeName():      java.util.AbstractMap$SimpleEntry

new java.io.Serializable(){}.getClass() (anonymous inner class):
    getName():          ClassNameTest$1
    getCanonicalName(): null
    getSimpleName():    
    getTypeName():      ClassNameTest$1
```

### Diamond problem


# Scala
## Constructors parameters scope
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

### `Anyval`'s `equals`
This is a special case where `==` and `equals` are not equivalent: the first cast types before performing equality check.
```scala
'a' == 97  // true
97L == 'a'  // true
'a'.equals(97)  // false
97L.equals('a')  // false
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

# Ada
## Bindings to Java concepts
mainly from Adacore's [Ada for C++ and Java developper](https://learn.adacore.com/pdf_books/courses/Ada_For_The_CPP_Java_Developer.pdf)
_____

### Imports
*Java:*
```java
// unecessary
import java.lang.System;
```

*Ada:*
```ada
-- `use` allows to use package elements without namespace
with Ada.Text_IO; -- use Ada.Text_IO;
```

### Program entry point
*Java:*
```java
public SomeClass {
  public static void main(String[] args) {
    ...
  }
}
```

*Ada:*
```ada
procedure Some_Procedure is
begin
  ...
end Some_Procedure;
```

### Print
*Java:*
```java
System.out.println("hello");
```

*Ada:*
```ada
-- Using `with` and `use` of `Ada.Text_IO.` package
Put_Line("hello");
```

### Print
*Java:*
```java
System.out.println("hello");
```

*Ada:*
```ada
-- Using `with` and `use` of `Ada.Text_IO.` package
Put_Line("hello");
```

### Declarations
*Java:*
```java
...
  int outerScopeField1
  int outerScopeField2
  void proc(int arg) {
    int localVar1 = arg * outerScopeField2;
    int localVar3 = arg * localVar1;
    int localVar2 = 2;
    this.outerScopeField2 = localVar1 * localVar2;
    this.outerScopeField1 = arg;
  }
```

*Ada:*
```ada
procedure Proc 
 (Arg : Integer;
  Outer_Scope_Field_1 : out Integer;
  Outer_Scope_Field_2 : in out Integer)
is
  Local_Var_1, Local_Var_2 : Integer;
  Local_Var_3 : Integer := 2;
begin
  Local_Var_1 := Arg * Outer_Scope_Field_2;
  Local_Var_3 := Arg * Local_Var_1;
  Outer_Scope_Field_2 := Local_Var_1 * Local_Var_2;
  Outer_Scope_Field_1 := Arg;
end Proc;
```

### Condition
*Java:*
```java
if (a > 0) {
  ...;
  ...;
} else if (a < 0) {
  ...;
  ...;
} else {
  ...;
  ...;
}
```

*Ada:*
```ada
if A > 0 then
  ...;
  ...;
elsif A < 0 then
  ...;
  ...;
else
  ...;
  ...;
end if;
```

### Switching
*Java:*
```java
switch (var) {
  case 0: case 1: case 2:
    ...;
    break;
  default:
    ...;
}
```
*Ada:*
```ada
case Var is
  when 0 .. 2 =>
    ...;
  when others =>
    ...;
end case;
```

### Loops
*Java:*
```java
while (var > 0) {
  ...;
}

for (int i = N; i >= 0, i--) {
  ...;
}

for (int i : intArray) {
  ...;
}
```
*Ada:*
```ada
while Var > 0 loop
  ...;
end loop;

for i in reverse 0 .. N loop
  ...;
end loop;

for i of Int_Array loop
  ...;
end loop;
```

### Ada's trong typing vs Java's implicit conversions
*Java:*
```java
double a = 1 / 3;
double b = 1 / (double)3;
```
*Ada:*
```ada
...  -- declarations
  A : Float;
  B : Float;
is
  A := Float (1 / 3);
  B := Float (1) / Float (3);
  ...
```

### Class/Type


### Compilation unit (Package / .java file)

# C
## variable, pointer types, heap, stack
```c
// Let's create a variable: an allocated memory range on the stack,
// sized to hold the data of a certain type, here `int` (4 bytes):
int a; 
// at this point `a` has an allocated memory range but it is uninitialized: 
// its memory range contains the trash that was written here previously.

// let's create another variable of type `int*` this time. 
// It will be mapped to an allocated memory range on the stack, 
// that can hold a memory address (64 or 32 bits) 
// which is intended to point to an `int`'s data (4 bytes).
int *p;
// at this point if we want to modify the 4 bytes of stack's data pointed
// by the memory address held by the variable `p` using 
// `*p = ...` we will end up with a segfault: core dump.
// It is because `p` is uninitialized and the memory address it holds is trash,
// and most of the time out of our current allowed memory space.

// variable `p` initialisation: let's put into the  memory range of the
// variable `p` the address of the memory range of `a`:
p = &a;

// now let's initialise the `int` data both held inside variable `a` and pointed by the memory address held inside variable `p`, using two differetn ways:
a = 1;
// at this point a == 1 and *p == 1
*p = 2;
// at this point a == 2 and *p == 2

// let's make variable `p` hold a memory address of 
// an allocated a memory range on the STACK, 
// that can contains the data of 2 `int`s (8 bytes, not initialized):
int trash[2];  
p = trash;
// with initialization:
int ones[2] = {1, 1};  
p = ones;

// let's make variable `p` hold a memory address of 
// an allocated a memory range on the HEAP, 
// that can contains the data of 2 `int`s (8 bytes, not initialized):
p = (int *)malloc(2*sizeof(int));

// let's put the value held by the variable `a`
// inside the second `int` slot of the memory range pointed
// by the address held by variable `p`.
// In two different ways:
*(p + 1*sizeof(int)) = a;
p[1] = a;

// let's make variable `p` hold a memory address of 
// an allocated a memory range on the STACK, 
// that can contains the data of 2 `int`s (8 bytes, not initialized):
int trash[2];  
p = trash;


```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTEwMTEyNDA1MTMsLTM2Nzk2OTEyNSw2MT
A2NDE1NDcsNjI4Njg3MDcxLDE2MjQzODg1NzMsLTE2MzY0NzIw
MjEsNjEwOTAyNDc1LC0yNjczOTk3NSwxOTc5OTM0MzEzLDIzMz
AzNjE3NiwtNDUwODI0MzEyLC00NTI2NzAyOTcsLTEwODQ3MzQz
MjYsLTExNTUzNjkyMzksMTUxNTA0NTMzMywtMTIyNzYwODQ2MC
wyMDI1MzA5ODg1LDEzOTg1OTU2NywtOTIzMDAwMDEwLC04MjY0
NzUyNzRdfQ==
-->