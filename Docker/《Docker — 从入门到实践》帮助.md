# 本文背景
* 系统：macOS Sierra 10.12.1
* Docker版本：Docker version 1.12.3, build 6b644ec
* 《Docker —— 从入门到实践》PDF版本：6448a627f6c4a9f0ca762951f390d3f9a571d310

# 问题列表 
1. 执行`docker run -d -p 80:80 --name webserver nginx`报错：`docker: Cannot connect to the Docker daemon. Is the docker daemon running on this host?.`

    解决方式：`sudo docker run -d -p 80:80 --name webserver nginx`
    ps：该解决方式暂时不是最优。TODO
    
2. 执行`docker run -d -p 80:80 --name webserver nginx`报错：``

