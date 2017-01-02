# 本文背景
* 系统：macOS Sierra 10.12.1
* Docker版本：Docker version 1.12.3, build 6b644ec
* 《Docker —— 从入门到实践》PDF版本：6448a627f6c4a9f0ca762951f390d3f9a571d310

# 问题列表 
1. 执行`docker run -d -p 80:80 --name webserver nginx`报错：`docker: Cannot connect to the Docker daemon. Is the docker daemon running on this host?.`

    * 原因：TODO
    * 解决方式：`sudo docker run -d -p 80:80 --name webserver nginx`
    ps：该解决方式暂时不是最优。TODO
    
2. 执行`docker run -d -p 80:80 --name webserver nginx`报错：`docker: Network timed out while trying to connect to https://index.docker.io/v1/repositories/library/nginx/images. You may want to check your internet connection or if you are`

    * 原因：docker.io存在被墙、连接困难的情况。
    * 解决方式：更换阿里云docker镜像。
            
        ![*](images/008CD83F-DF1E-470C-8192-5B5D0F9DFE70.png)
3. 
4. 

