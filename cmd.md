<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->
# Syntax reminders

## Hadoop ecosystem
### Spark / scala
```bash
//Spark-submit
C:\...\wordcountsparkscala\target> spark-submit --class <package.main_class_name> <jar_name.jar> args_0 arg_1 arg_2 ....
```
```scala
//spark2-shell
val textFile = sc.textFile("hdfs:///user/dlff1193/WORK/TESTHDFS/heyJude.txt")
textFile.first
val counts = textFile.flatMap(line => line.split(" ")).map(word => (word, 1)).reduceByKey(_ + _)
counts.saveAsTextFile("hdfs:///userr/dlff1193/WORK/TESTHDFS/example")
counts.collect
val countsFiltered=counts.filter(_._2>10)
countsFiltered.collect
//Compte nb 'e'
val v2=tokens.map(word=>{var x=0;for(l <- word){if(l=='e'){x=x+1;println(x)}};(1,x)})
val r1=v2.reduceByKey(_+_)
//fin compte nb 'e'
//remplacement 'e' par 'a'
val v2=v1.map(word =>{var wordModif:String="";for(l<-word){if(l=='e'){wordModif+='a'}else{wordModif+=l}};(1,wordModif);})
//fin remplacement 'e' par 'a'
```
### PYSPARK
```bash
pyspark --master local[2] (lance jupyter avec sc accessible)
```
### HDFS
```bash
hadoop fs -text /user/dlff1193/WORK/TESTHDFS/resOnOneDayTestTestTestIuEmptyIMEICounter/*00*
hdfs dfs -ls WORK/TESTHDFS
hadoop fs -get WORK/TESTHDFS/*py
```
### MR
```bash
//MapReduce
yarn jar .\target\wordcountmr-1.0-SNAPSHOT.jar com.enzobnl.wordcountmr.WordCount .\pom.xml ./output
hdfs dfs -text ./output/part-r-00000
```
### Kafka:
```
./zookeeper-server-start.bat ..\..\config\zookeeper.properties
./kafka-server-start.bat ../../config/server.properties
//production:
./kafka-console-producer.bat --broker-list localhost:9092 --property "parse.key=true" --property "key.separator=:" --topic topicwithkey
//consumer:
./kafka-console-consumer.bat --bootstrap-server localhost:9092 --property "parse.key=true" --property "key.separator=:" --topic mytopicout
```
## Terminals
### unix
```bash
//NEBOJSA PIPE EXAMPLE : LISTE toute la 1ere colonne
 cat OS_IU_sample | cut -f1 |more 
 hadoop dfs -cat /Data/O19593/SECURE/RAW/TXT/OS_IU_1/2017/05/16/2017_05_16_OSO_3G_37.csv | head -1000 > OS_IU_sample
//SELECT 1000 1eres lignes
hadoop dfs -cat /Data/O19593/SECURE/RAW/TXT/OS_IU_1/2017/05/16/2017_05_16_OSO_3G_37.csv | head -1000 > O
S_IU_sample
hadoop dfs -put OS_IU_sample WORK/TESTHDFS/IU_TEST

// bash helloworld (implementation of sh POSIX norm)
#!/bin/bash
# declare STRING variable
STRING="Hello World"
#print variable on a screen
echo $STRING
// make file executable
$ chmod +x hello_world.sh 
// chmod recursively on folder
chmod u=rwx,g=rx,o=r /.../folder/ --recursive
// tar extract tgz
tar -xvzf /path/to/yourfile.tgz
// env var tmp
export ABC=/bla/blouche
// env var perm for all users
nano /etc/environment
// show permissions
ls -la fileOrDir
// mount read-only
sudo mount -o ro /dev/nvme0n1p5 /media/enzobnl/sdd
sudo unmount /media/enzobnl/sdd
// Fix broken installs
sudo apt-get --fix-broken install
// update paths
source ~/.bashrc
// You can make a file executable as follows:
chmod a+x exampleName.AppImage
// get historic back after reset and clear
[shortcut] Ctrl + L
// create a symlinl (symbolic link)
sudo ln -s origin target
// ssh bridge: maps port 9000 of 10.240.0.44 accessed through <host_url> to localhost:9000
ssh -L 9000:10.240.0.44:9000 <host_url>
// put a directories's jars in classpath var, in ~/.bashrc
export CLASSPATH="/home/enzo/Prog/lib/jar/*"
```
### CMD removes :
```bash
del <fichier>
//remove directory :
rd /s /q <directory> 
```
### PowerShell
```
//display env:
$env:HADOOP_HOME
```

## Java
### Java
```bash
// JAVAC RECURSIVELY
//Linux
$ find -name "*.java" > sources.txt
$ javac @sources.txt

//CMD
> dir /s /B *.java > sources.txt
> javac @sources.txt

//Java outside bin directory (like eclipse does)
java -classpath bin <className>

//java inline opti
javac -o ...
```
## Build/IDEs/VCS
### Maven:
```
//one java/scala test:
mvn test -Dtest=com....   [-f pom.xml]
mvn test -Dsuites=com....   [-f pom.xml]

Goals : package ou assembly:single
```
### pypi //pip
```
//specific version
pip install 'scimple==1.2.5' --force-reinstall

pip install twine
//local install :
pip install .
//pypi install:
python setup.py sdist (bdist_wininst)
cd dist
twine upload <package_name>-<version>.tar.gz
//use:
pip install <package_name>
pip install <package_name> --upgrade
```
### Eclipse :
```
//REMOVE BLANCK LINE
regular ^\s*\n
```

### IntelliJ
```
Doc : Select item and then Ctrl + Q
```
### atom
```bash
apm install hydrogen
```

### vim
```
esc (quit edit mode)
type ':wq' + press enter (save and quit vim)
```
### git
```bash 
//tag
$ git tag -a v1.4 -m "my version 1.4"
//Fusionner i last commits
git rebase -i <after-this-commit>
//lines in repo:
git ls-files | xargs wc -l
//reset to last commit
git reset --hard
fichier .gitignore avec les dossier à pas push (un par ligne, pas besoin du path relatif ./ , juste le nom)
//init :
git init 
git remote add origin http/...
//si on importe : 
git pull origin master
//1st commit, idem for others:
git add .
git commit -m "Initial Commit"
git push origin master
//branch
git checkout (-b for creation only) <name>
git push origin <name>
//sometime :
git push origin heads/<name>
//show refs :
git show-ref (to know if heads/... or sm is needed)
//show last commit log:
git log -1
//update change in .gitignore
git rm -r --cached .
AND:
git add .
git commit ...etc
//rename last commit
git commit --amend -m "New commit message"
//merge stashed changes with pulled (or just current) state
git stash apply
```

## Text edition
### Pandoc
```
c:/applications/anaconda2/scripts/pandoc --latex-engine=xelatex -H preamble.tex -V fontsize=12pt -V documentclass:book -V papersize:a4paper  -V classoption:openright --chapters --bibliography=papers.bib --csl="csl/nature.csl" title.md summary.md zusammenfassung.md acknowledgements.md toc.md "introduction/intro1.md" "introduction/intro2.md" chapter2_paper.md chapter3_extra_results.md chapter4_generaldiscussion.md appendix.md references.md -o "phdthesis.pdf"
c:/applications/anaconda2/scripts/pandoc .\plan.md -o plan.pdf  --read=markdown --latex-engine=xelatex
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg2NzE2ODc1MSwtODc0OTc4NTI5LC0xMz
U4MzM5MTUwLDU5NzA2NTA3MiwtMTAzMDc3MDM5N119
-->