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
3. `nsenter`：随着docker1.3版本开始，`docker exec -it CONTAINER_NAME bash`成为推荐的连接到容器的方式。参考：https://github.com/jpetazzo/nsenter。
    > There are differences between nsenter and docker exec; namely, nsenter doesn't enter the cgroups, and therefore evades resource limitations. The potential benefit of this would be debugging and external audit, **but for remote access, docker exec is the current recommended approach**.
4. 在《使用网络-容器互联》章节， 
    * `sudo docker run -d --name db training/postgres`
    * `sudo docker run -d -P --name web --link db:db training/webapp python app.py`
    后进行`docker ps`，无法得到如下图：
            ![*](images/4A097CA8-F4DC-44AA-91D0-C4397C753576.png)
    得到的结果如下图：
            ![*](images/452D8BD2-8FDF-45F2-B3D8-BA8BF04BE294.png)
    * 原因：TODO
    * 解决方式：`sudo docker inspect -f "{{ .HostConfig.Links }}" web`
        > [/db:/web/db]
        
        示例成功。
       
5. 测试
