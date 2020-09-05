#Docker OpenStack Swift Proxy Server

This is a docker file that creates an OpenStack swift proxy image. You can
specify the object nodes' ip, port, and storage device. Furthermore, you can
specify the ip address, path, and password of the machine where you want to scp
the ring files (you can use specify same machine while launching object server
container so that object server container can have updated ring file).


## startmain.sh

This Dockerfile uses supervisord to manage the processes.
Dockerfile we will be starting multiple services in the container, such as
rsyslog, memcached, and the required OpenStack Swift daemons for launching
swift proxy.


## Usage

首先需要在storage node上面为swift-disk分配一块地方，目前采用跟saio的步骤一样

1. 为回环设备创建文件
```bash
$ sudo mkdir -p /srv
$ sudo truncate -s 1GB /srv/swift-disk
$ sudo mkfs.xfs /srv/swift-disk
```

2. 编辑/etc/fstab并添加

```bash
$ /srv/swift-disk /mnt/sdb1 xfs loop,noatime 0 0
```

3. 创建swift数据挂载点

```bash
$ sudo mkdir /mnt/sdb1
$ sudo mount -a
```

4. 用```df -h```查看创建的回环设备，假设为 /dev/loop2，那么下面的 sdb1 就替换为 loop2

```bash
chloe@host:~$ docker run -d -p 12345:8080 -e SWIFT_OBJECT_NODES="192.168.3.68:8010:sdb1;192.168.0.153:5010:sdd1" -e SWIFT_PWORKERS=64  -e SWIFT_SCP_COPY=root@192.168.3.68:~/docker-swift-proxy/files:654321 -t swift-proxy
```

注意，上面的命令中的端口号是映射前面部分的端口号！！也就是主机的端口号！

Over here, we mapped 8080 port of container to port 12345 on host. 


Note that we mentioned only one port and next two are calculated automatically by adding 1 and 2. 


Similarly, storage device that container
can use for storing the data in case of each container is also specified. Please, be sure you specify
the correct device which has enough disk space. Using incorrect device can be catastrophic.


SWIFT_PWORKERS is used to set the proxy workers dynamically.

The ring files created at the proxy server needs to be copied to the object servers as well. SWIFT_SCP_COPY
contains the remote location path where ring files can be copied to so that we
can copy these files before launching object, container, and account servers on object servers. root@192.168.3.68:~/docker-swift-proxy/files is the remote path, whereas 654321 is the `scp password`.

At this point OpenStack Swift proxy is running.


```bash
chloe@host:~$ docker ps
CONTAINER ID        IMAGE                                     COMMAND                CREATED             STATUS              PORTS                     NAMES
f7bd815a49ee        swift-proxy                               "/bin/sh -c /usr/loc   4 seconds ago       Up 2 seconds        0.0.0.0:12345->8080/tcp   kickass_bohr
```

善于使用docker的日志

```bash
$ docker logs <container name>
```

以及进入docker环境中

```bash
$ docker exec -it <container name> /bin/bash
$ exit 退出
```

Next, we need to launch object server containers on the machines we specified. To launch object servers please look at the following:
https://github.com/cy0713/docker-swift-object


Once object server containers are up and running, we can use the swift python client to access Swift using the Docker forwarded port, in this example port 12345.

```bash
chloe@host:~$ swift -A http://127.0.0.1:12345/auth/v1.0 -U test:tester -K testing stat
       Account: AUTH_test
    Containers: 0
       Objects: 0
         Bytes: 0
 Accept-Ranges: bytes
    Connection: keep-alive
   X-Timestamp: 1450494080.48790
    X-Trans-Id: tx8a5e8267911a4ac99f01c-0056a89c11
  Content-Type: text/plain; charset=utf-8
```

Try uploading a file:

首先要创建文件 swift.txt

```bash
$ sudo vim swift.txt
```

```bash
chloe@host:~$ swift -A http://127.0.0.1:12345/auth/v1.0 -U test:tester -K testing upload swift swift.txt
swift.txt
```

That's it!
