- 编写Dockerfile
- 给工程打包

`mvn -Dmaven.test.skip=true -U clean package`

- 进入工程目录下

`docker build -t springcloud/eureka .`

- 启动docker

`docker run -p 8761:8761 -d springcloud/eureka`