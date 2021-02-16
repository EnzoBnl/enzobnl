<!--NOTE HEAD START-->
<link rel="icon" type="image/png" href="./imgs/favicon_db.png" />
<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.0.0/mermaid.min.js"></script>
<script type="text/x-mathjax-config">MathJax.Hub.Config({tex2jax: {skipTags: ['script', 'noscript','style', 'textarea', 'pre'],inlineMath: [['$','$']]}});</script>
<script src="https://cdn.mathjax.org/mathjax/latest/MathJax.js?config=TeX-AMS-MML_HTMLorMML" type="text/javascript"></script>
<script>document.body.style.background = "#f2f2f2";</script>
<!--NOTE HEAD END-->
# Syntax reminders

## Big Data related
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

### GCP
```bash
# install SDK
curl https://dl.google.com/dl/cloudsdk/release/install_google_cloud_sdk.bash | bash
# submit spark job
gcloud dataproc jobs submit spark --cluster=dataproc-xxxxxxx-dev --region=global --class com.xxxxxxx.analysis.testparquetjoin.Main --jars gs://xxxxxxx-dataproc/testparquetjoin/testparquetjoin-0.1-jar-with-dependencies.jar
# list jobs
gcloud dataproc jobs list --cluster=dataproc-xxxxxxx-dev --region=global
# kill job by id
gcloud dataproc jobs kill --region=global
# rename file on Cloud Storage
gsutil mv gs://old_name gs://new_name
# switch google cloud project
gcloud config set project <project-id>
```


## Terminals
### unix
```bash
# change default jdk: in .bashrc:
export JAVA_HOME=/opt/java/<jdk>
export PATH=$JAVA_HOME/bin:$PATH
# create a new session
sudo adduser <user_name>
# Disk Usage Analyzer
sudo baobab
# du: list folder sizes with a given depth, at bytes level
du -h --max-depth=1 --block-size=1

# show running processes
ps -A
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
# compression de tous les fichiers gzip du répertoire, 8 à la fois, en faible prio avec nice:  
nice parallel -j 8 gzip -- *.json
// tar extract tgz
tar -xvzf /path/to/yourfile.tgz
// archive and compress with gzip
tar -cz filename -f filename.tar.gz
// unzip
unzip file.zip -d destination_folder
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
// install specific version
`apt-get install «pkg»=«version»`
// list available versions on apt
apt-cache policy <package name>
// fix broken installations
sudo apt-get --fix-broken install
// list files installed by a package
dpkg -L <package_name>
// install a .deb package
dpkg -i package_name.deb
// update paths
source ~/.bashrc
// You can make a file executable as follows:
chmod a+x exampleName.AppImage
// get historic back after reset and clear
[shortcut] Ctrl + L
// create a symlinl (symbolic link)
sudo ln -s origin target
// generate ssh pub/private keys
// 1. generate the keys
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
// 2. run ssh agent in background
eval "$(ssh-agent -s)"
// 3. add it to the agent
ssh-add ~/.ssh/id_rsa
// -> public key is in ~/.ssh/id_rsa.pub

// ssh bridge: maps port 9000 of 10.240.0.44 accessed through <host_url> to localhost:9000
ssh -L 9000:10.240.0.44:9000 <host_url>
// put a directories's jars in classpath var, in ~/.bashrc
export CLASSPATH="/home/enzo/Prog/lib/jar/*"
// log as another user in terminal
sudo su - username
# make host resolution made by a server
ssh -D 1080 <server>
# chromium with port redirection
chromium-browser --proxy-server="socks5://127.0.0.1:1080" --user-data-dir="/home/enzo/chrome-proxy-profile"
# desactivate CORS for dev between file:// and http protocols
chromium-browser --disable-web-security --user-data-dir
# quit ssh connection
exit
// shebang commentary
#!/usr/bin/python
#!/bin/bash

//grant current user with privilegess on all the content of folderx
sudo chown $USER folderx -R

// empty root trash
sudo rm -r /root/.local/share/Trash/files

# Null device in UNUX systems
/dev/null
```


### PowerShell
```
//display env:
$env:HADOOP_HOME
// remove file
del <file>
//remove directory :
rd /s /q <directory> 
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
### Gradle
```
# build without test
gradle build -x <test task>
# run specific test classes
gradle test --tests SomeClassTest
# in build.gradle, to have an auto generated 'run' task:
plugins {  
  [...]
  id "application"  
}  
  
apply plugin : "java"  
ext {  
  javaMainClass = "com.bla.MyMainClass"  
}  
  
application {  
  mainClassName = javaMainClass  
}
```

### Maven:
```
//one java/scala test:
mvn test -Dtest=com....   [-f pom.xml]
mvn test -Dsuites=com....   [-f pom.xml]
mvn package -DskipTests=true

// build specific modules (. is parent pom)
mvn -pl .,library1,library2 clean install ...

sudo mvn install:install-file -U -Dfile=/path/bla.jar    -DgroupId=com.bla    -DartifactId=bla    -Dversion=X.x    -Dpackaging=jar   -DgeneratePom=true -DlocalRepositoryPath=/root/.m2/repository
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

// Deep Learning
pip install tensorflow-gpu
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
[How to write good commit messages](https://chris.beams.io/posts/git-commit/#imperative)

```bash 
// use ssh auth o github
- generate ssh pairs and add them to agent
- copy entire content (including algo name and email)
- paste it as a new ssh key in section Setting -> SSH/GPG
- git remote set-url origin git@github.com:ORGA_OR_USER/REPO.git
// rename commit message
git commit --amend -m"new message"
// rename branch
git branch -m <oldname> <newname>
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
// chackout a remote tag
git checkout tags/<name>
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
// roll back to commit -X
$ git commit -m "Something terribly misguided"             # (1)
$ git reset HEAD~X                                          # (2)
<< edit files as necessary >>                              # (3)
$ git add .                                              # (4)
$ git commit -c CommitHashThatYouWantTheMessageAndAuthorship                               # (5)
# after rebasing, force git pull with stop if someone else pushed on the branch:

$ sudo git push origin branch-x --force-with-lease
# override local branch with remote

$ sudo git fetch --all
$ sudo git reset --hard origin/master

# Pull unitary commits onto current branch
git cherry-pick <commit-hash>

# find a pattern of path in entire root

sudo find / -name *middleisthis*
```

## Text edition
### Pandoc
```
c:/applications/anaconda2/scripts/pandoc --latex-engine=xelatex -H preamble.tex -V fontsize=12pt -V documentclass:book -V papersize:a4paper  -V classoption:openright --chapters --bibliography=papers.bib --csl="csl/nature.csl" title.md summary.md zusammenfassung.md acknowledgements.md toc.md "introduction/intro1.md" "introduction/intro2.md" chapter2_paper.md chapter3_extra_results.md chapter4_generaldiscussion.md appendix.md references.md -o "phdthesis.pdf"
c:/applications/anaconda2/scripts/pandoc .\plan.md -o plan.pdf  --read=markdown --latex-engine=xelatex
```
<!--stackedit_data:
eyJoaXN0b3J5IjpbLTg5OTQ3NzcwMSwtMjEzNzY3MDE4NCwtMT
UwNTU2MDExNSwxMDQxNjE1MDE5LDE1MDI0ODQxMTIsMTYzNzk3
MTQ3MSwtMTAwNTQ2NDkzNCw3OTgzMDI0MjMsLTE5MzA4MDU5Nj
UsMTc0Mjg0MzEwOSwxMjczNTk3MDk2LDE3MzQ5NDgxNzAsODYw
MzIzNzkxLC0xNzMwMDQ3MTMzLC02NTYyMjU5MzEsMTU3NDY0Mz
U2MywtMTgxNDYwOTkzMywtNDU0NDQ3NDg5LC0yMDU3Mjc0MTM3
LDE4MDQ3ODM1ODldfQ==
-->