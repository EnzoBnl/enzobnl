<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->
# Functionnal Programming Notes (Scala)
## Strucutres algébriques
<div class="mermaid">
graph TB
Ma --> DeGr
DeGr --> Mo
Mo --> Gr
Mo --> MoCo
Ma[Magma: Ensemble doté d'une loi de<br/>composition interne *]
DeGr[Demi-groupe: Magma dont * est associative]
Mo[Monoide: Demi-groupe doté d'un<br/>élément neutre pour *]
MoCo[Monoide Commutatif: Monoide<br/>dont * est commutative]
Gr[Groupe: Monoide admettant un<br/>élément symétrique pour *<br/>pour chacun de ses éléments]
</div>

$$G=Monoide(*,e)$$
est un groupe $$\Leftrightarrow \forall x\in G,\exists x^{-1}\in G,x*x^{-1}=e$$

## Cats
### Semigroup
Cats' semigroup implem (`import cats.Semigroup`):
```scala
trait Semigroup[A] {
  def combine(x: A, y: A): A
}
```
### Monoid
Cats' semigroup implem (`import cats.Monoid`):
```scala
trait Monoid[A] extends Semigroup[A] {
  def empty: A
}
```
### Functor
Abstraction over a type constructor `F[_]` (can be `List[Int]`, `Map[Int, String]`, `Double`) providing ability to `map` over it. 

```scala
trait Functor[F[_]] {
  def map[A, B](fa: F[A])(f: A => B): F[B]
}
```
*Note*: The `_` used is just a convention: `Functor[F[_]]` is the same as `Functor[F[T]]` but as you do not use `T` in the *Functor*, it is clearer to name it in this anonymous fashion. 
But, this will compile thanks to `_`:
```scala
class C[F[_]](f: F[_])
```
but this won't (*not found: type T*):
```scala
class C[F[T]](f: F[T])
```
To name you're force to add it as type parameter too. This compiles:
```scala
class C[T, F[T]](f: F[T])
```

## Monads Examples
### Option[A]
Alternative to OOP's `null`.
Two subtypes:
- `val o = Some(v)`:
```scala
final case class Some[+A](val x : A) extends scala.Option[A]
```
- `val o = None`:
```scala
case object None extends scala.Option[scala.Nothing]
```
For any `o: Option[T]`, the first common supertype to `o` and `None` is `T`.


## Implicits
### Implicit conversions
This
```scala
((i: Int) => i)("1")
```
gives
```
type mismatch;
 found   : String("1")
 required: Int
```
but this compiles just fine:
```scala
implicit def StringToInt(s: String) = Integer.parseInt(s)
((i: Int) => i)("1")
```