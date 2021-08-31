<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->

# Diverse themes

## Vectorization
**Vectorize a piece of code:** 

-> Action to (re)write code to leverage the **SIMD (Single Instruction, Multiple Data)** capacities of your hardware, either by writing code sufficiently straightforward to be **auto-vertorizable** by your medium/high level language compiler (C, C++) or by using **dedicated libraries in these languages (SSE)**. 

Under the hood, the assembly code generated will use specific SIMD instructions that can **apply a given transformation at the same time on up to all the data** (typically subsequence of an array) **present in a given register**.

## Asymptotic analysis

$f(n)=O(g(n))\iff \exists n_o \in N, \exists c\in R^{+*}, \forall n>n_o,\vert\frac{f(n)}{g(n)}\vert<=c$

$f(n)=o(g(n))\iff \forall \epsilon \in R^{+*}, \exists n_o \in N,  \forall n>n_o,\vert\frac{f(n)}{g(n)}\vert<=\epsilon$

$f(n)=\Omega(g(n))\iff \exists n_o \in N, \exists c\in R^{+*}, \forall n>n_o,\vert\frac{f(n)}{g(n)}\vert>=c$

$f(n)=\omega(g(n))\iff \forall \Delta \in R^{+*}, \exists n_o \in N,  \forall n>n_o,\vert\frac{f(n)}{g(n)}\vert>=\Delta$

$f(n)=\Theta(g(n))\iff \exists n_o \in N, \exists c\in R, \forall n>n_o,\frac{f(n)}{g(n)}<=c+\epsilon$

$f(n)\sim g(n)\iff lim_{n \to\infty} \frac{f(n)}{g(n)} = 1$


## Memories accessing latencies
From http://ommil.com/scalax14/#/7/7

|Computer| 	Human Timescale |	Human Analogy|
|:--:|:--:|:--:|
|L1 cache reference 	|0.5 secs 	|One heart beat|
|Branch mispredict 	|5 secs 	|Yawn|
|L2 cache reference 	|7 secs| 	Long yawn|
|Mutex lock/unlock 	|25 secs 	|Making a cup of tea|
|Main memory reference 	|100 secs 	|Brushing your teeth|
|Compress 1K bytes with Zippy 	|50 min |	Scala compiler CI pipeline|
|Send 2K bytes over 1Gbps network 	|5.5 hr 	|Train London to Edinburgh|
|SSD random read 	|1.7 days 	|Weekend|
|Read 1MB sequentially from memory 	|2.9 days| 	Long weekend|

## MultiThreading
### Simultaneous access of the main memory (RAM)
- It is physically impossible for multiple threads to get access the memory (same memory address or not, read or write: it does not matter) at the same time because the *address bus* is a single entry point and thread need to compete to get the claimed access (the *data bus* in the other direction is also a single point). 
- This bottleneck can be partially bypassed thanks to hardware and architectures features (coherance bus, CPUs caches, MIMD/SIMD access ...) and things may look like threads get pure parallel accesses in certain executions, but it is not truly the case.

*more here:* [check this great SO response](https://softwareengineering.stackexchange.com/a/278774/341648)

### Synchronized block
```java
synchronized(someObject) {
  /* this code is guaranteed to be run by only 1 thread at a time, the thread that has obtained the lock on someObject */
}
```
Les blocks synchronized utilisent des sémaphores + section de code critique au niveau assembleur pour guarantir que le verrou est lock par une seule thread.

### Deadlock
Occurs when 2 or more locks are accessed in a specific oder that blocks all the threads waiting for each others
```java
public class TestThread {  
  public static Object Lock1 = new Object();  
  public static Object Lock2 = new Object();  
  
  public static void main(String args[]) {  
    ThreadDemo1 T1 = new ThreadDemo1();  
  ThreadDemo2 T2 = new ThreadDemo2();  
  T1.start();  
  T2.start();  
  }  
  
  private static class ThreadDemo1 extends Thread {  
    public void run() {  
      synchronized (Lock1) {  
        System.out.println("Thread 1: Holding lock 1...");  
  
        try { Thread.sleep(10); }  
        catch (InterruptedException e) {}  
        System.out.println("Thread 1: Waiting for lock 2...");  
  
        synchronized (Lock2) {  
          System.out.println("Thread 1: Holding lock 1 & 2...");  
        }  
      }  
    }  
  }  
  private static class ThreadDemo2 extends Thread {  
    public void run() {  
      synchronized (Lock2) {  
        System.out.println("Thread 2: Holding lock 2...");  
  
        try { Thread.sleep(10); }  
        catch (InterruptedException e) {}  
        System.out.println("Thread 2: Waiting for lock 1...");  
  
        synchronized (Lock1) {  
          System.out.println("Thread 2: Holding lock 1 & 2...");  
        }  
      }  
    }  
  }  
}
```
Run:
```
Thread 1: Holding lock 1...
Thread 2: Holding lock 2...
Thread 1: Waiting for lock 2...
Thread 2: Waiting for lock 1...
```

### Fix Deadlocks
Fixable by reordering locks access



## Web
### Security
#### General practice hash on server side:
1. Client side: Password is sent in clear in the request, it's safe because all is encrypted if you pass through a SSL connection
2. Server side: Password is hashed before stored

If SSL if broken: clear password is revelated !
**Consequence (2): The hacker can access the account with the needed clear password + access to other accounts where same password has been used by unconscious user**

If passwords database is hacked, revealed hashed password are unusable to connect to any accounts (except accounts where same hashing is performed client side !)

#### Bad good idea: hash on client side
1. Client side: Password is hashed and sent
2. Server side: hashed Password recieved is stored as it is.

If SSL if broken OR passwords database is hacked: hash password(s) is/are revelated !
**Consequences (3): The hacker can access this/any account(s) with the needed hashed password(s) + access to other account on services that uses the same hash system client side**

#### Strong idea: Hash both client side and server side
1. Client side: Password is hashed and sent
2. Server side: hashed Password recieved is again hashed and then stored.

If SSL if broken: primary hash of the password is revealed:
**Consequence(1.5): The hacker can access this with the needed primary hashed password + access other accounts where same hash is used by client side and same password by user**

Passwords database is hacked: hash password(s) is/are revelated : nothing bad can happen with this !

### HTTP:
#### Requetes
Types:
**GET**  
C'est la méthode la plus courante pour demander une ressource. Une requête **GET** est sans effet sur la ressource, il doit être possible de répéter la requête sans effet.  
**HEAD**  
Cette méthode ne demande que des informations sur la ressource, sans demander la ressource elle-même.  
**POST**  
Cette méthode doit être utilisée lorsqu'une requête modifie la ressource.  
**OPTIONS**  
Cette méthode permet d'obtenir les options de communication d'une ressource ou du serveur en général.  
**CONNECT**  
Cette méthode permet d'utiliser un proxy comme un tunnel de communication.  
**TRACE**  
Cette méthode demande au serveur de retourner ce qu'il a reçu, dans le but de tester et d'effectuer un diagnostic sur la connexion.  
**PUT**  
Cette méthode permet d'ajouter une ressource sur le serveur.  
**DELETE**
```
Ligne de commande (Commande, URL, Version de protocole)
En-tête de requête
<nouvelle ligne>
Corps de requête
```
examples:
```
GET /fichier.ext?variable=valeur&variable2=valeur2 HTTP/1.1
Host: www.site.com
Connection: Close
<nouvelle ligne>```
```
```
POST /fichier.ext HTTP/1.1
Host: www.site.com
Connection: Close
Content-type: application/x-www-form-urlencoded
Content-Length: 33
<nouvelle ligne>
variable=valeur&variable2=valeur2
```
#### Reponses
```
Ligne de statut (Version, Code-réponse, Texte-réponse)
En-tête de réponse
<nouvelle ligne>
Corps de réponse
```
example:

```
HTTP/1.1 200 OK  
Date: Thu, 11 Jan 2007 14:00:36 GMT  
Server: Apache/2.0.54 (Debian GNU/Linux) DAV/2 SVN/1.1.4  
Connection: close  
Transfer-Encoding: chunked  
Content-Type: text/html; charset=ISO-8859-1

178a1

<!DOCTYPE html PUBLIC "-//W3C//DTD XHTML 1.0 Strict//EN" "http://www.w3.org/TR/xhtml1/DTD/xhtml1-strict.dtd">  
<html xmlns="http://www.w3.org/1999/xhtml" xml:lang="fr">  
<head>  
<meta http-equiv="content-type" content="text/html; charset=ISO-8859-1" />  
<meta http-equiv="pragma" content="no-cache" />

<title>Accueil - Le Site du Zéro</title>
```

### API REST/RESTful (REpresentational State Transfer)
**Modèle de maturité de Richardson**: it's all about make full use of http protocol *(whereas SOAP APIs put everything into POST requests targeting always /api URI and all the logic is added server side)*
- niveau 0: Stateless: The server needs to recieve in the request all the information it needs to answer (cookies usage)
- niveau 1: Leverage URI hierarchisation possibility to adress resources
- niveau 2: Leverage HTTP request types (GET, POST...) particular semantics 


## Git
### pull / rebase / merge
`git pull` and `git rebase` are not interchangeable, but they are closely connected.

`git pull` fetches the latest changes of the current branch from a remote and applies those changes to your local copy of the branch. Generally this is done by merging, i.e. the local changes are merged into the remote changes. So `git pull` is similar to `git fetch & git merge`.

Rebasing is an alternative to merging. Instead of creating a new commit that combines the two branches, it moves the commits of one of the branches on top of the other. 

You can pull using rebase instead of merge (`git pull --rebase`). The local changes you made will be rebased on top of the remote changes, instead of being merged with the remote changes.

Atlassian has some excellent [documentation on merging vs. rebasing](https://www.atlassian.com/git/tutorials/merging-vs-rebasing/workflow-walkthrough).

### Co-authoring a commit
In commit message:
```
Commit title

Commit body

Co-authored-by: Foo <foo@bar.com>
Co-authored-by: Bar <bar@foo.com>
```

## Assembly/Hardware
### Windows/Linux binaries incompatibility ?
- *There is no difference. The assembly code is the same if the processor is the same. x86 code compiled on Windows is binary compatible with x86 code on Linux. The compiler does not produce OS-dependent binary code, but it may package the code in a different format (e.g. PE vs. ELF).
The difference is in which libraries are used. In order to use OS stuff (I/O for example) you must link against the operating system's libraries. Unsurprisingly, Windows system libraries are not available on a Linux machine (unless you have Wine of course) and vice-versa.*

- *There's no difference in the binary code, opcodes are the same. The difference is in the syntax of the mnemonic form, i.e. mov %eax,%ebx (AT&T) and mov ebx,eax (Intel) have the same binary*

## 10 points to study for wise software engineering
### Main
1. Java, Python
2. Software designing: Design patterns, Testing
3. Data structures: Specificities, complexities, algorithms
4. Data bases: SQL, noSQL, Big data
5. Computer Architecture: Latencies, assembler

### Secondary
1. a 3rd language: Scala (functional prog) or Kotlin(android) or C++(games) or ...
2. Some more low level knowledge: C, JVM
3. Web APIs building: RESTful principles...
4. Master 1 to 3 specialized frameworks: Node.js, React, Spark, Kubernetes, Unity...
5. Practice with CI/CD & cloud
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTE0MzczODA2NTEsMTg5MjA4MzM2Miw3Mj
gxNTI0NzcsLTE2MzE2OTE3OTksMTEyMjc0OTAwMSwtMTUzMjE0
MDQ1OSwtMTQ0MTgxNTQzOSwxNDczOTU4NDQxLDE0NzM5NTg0ND
EsLTE2NTA2NDEwODgsLTIwODg1NDU1MDYsMTI2NTI3NjcxLDE2
MTMxMzY3MTEsLTIwMzc5MDk4NzksLTE5ODE2MjI3NzMsMjcxNj
E3ODI2XX0=
-->