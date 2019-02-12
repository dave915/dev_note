# Jenkins를 이용한 Play Framework 배포

## 배포전 참고 사항
jenkins의 Execute shell을 이용하여 배포시 Jenkins는 빌드시 생성된 subprocess들을 찾아서 죽임.<br>
Jenins는 daemonize를 사용하여 process 빌드ID를 변경하는 방법을 알려주고있다.
* 참고: https://wiki.jenkins.io/display/JENKINS/Spawning+processes+from+build

## daemonize 설치
``` shell
wget https://github.com/bmc/daemonize/archive/release-1.7.8.tar.gz
tar -zxvf release-1.7.8.tar.gz
cd daemonize-release-1.7.8
sh configure
make
sudo make install
./install-sh
daemonize -v
```

## 젠킨스 배포
여러 서버에 배포 할 경우 각 서버마다 build를 다시하면 github시점 문제가 발생 할 수 있기 때문에 build와 deploy job을 분리시켜 build한번 후 각 서버로 deploy

### Build job
git 연동
![](/images/server/jenkins_scala1.png)

Execute shell
 * build 수행 후 deploy job으로 결과파일을 옮겨둠

![](/images/server/jenkins_scala2.png)

 참고: sbt 패키징

 ```shell
 # target/universal 하위에 zip 파일로 떨어짐
sbt clean dist

or

# target/universal 하위에 tar 파일로 떨어짐
sbt clean stage
tar -cvf de_todo_back.tar target/universal/stage
 ```
 참고:  https://www.playframework.com/documentation/2.6.x/Deploying#Running-a-production-server-in-place

 ### Deploy Job
 Execute shell
 ![](/images/server/jenkins_scala3.png)

 각 서버의 deploy.sh
 ```shell
 #!/bin/sh
pwd
currentDateTime=`date +"%Y%m%d%H%M%S"`
mv ./de_todo_back.tar /home/service/todo-back/de_todo_back.tar
cd /home/service/todo-back
tar -xvf de_todo_back.tar
mv stage $currentDateTime
ls /home/service/todo-back/release
if [ -e /home/service/todo-back/release/RUNNING_PID ]; then
        rm /home/service/todo-back/release/RUNNING_PID
fi
ls /home/service/todo-back/release
pkill -9 -ef todo_back
rm release
ln -s $currentDateTime release
ls -al /home/service/todo-back
rm de_todo_back.tar
cd release/bin
pwd
chmod 755 ./de_todo_back
daemonize -E BUILD_ID=todo_back /home/service/todo-back/release/bin/de_todo_back -Dconfig.file=/home/service/todo-back/release/conf/application-prod.conf &
 ```
 참고:  conf폴더 하위에 환경별로 config파일을 만든뒤 -Dconfig.file 옵션을 이용하여 각 환경에 맞는 config 수행
