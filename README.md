# X11Forward-Server-Container

本仓库用于记录连接服务器或使用Docker Container时，配置X11 Forward的流程。本地机器、远程服务器和Docker Container都是Ubuntu系统。


X11 是一个网络协议，主要用于在 UNIX 和 Linux 系统上实现图形用户界面。X11 的架构分为 X 服务器和 X 客户端：
- X 服务器：负责实际的显示输出和用户输入（键盘、鼠标等）。此处X服务器位于本地机器上。
- X 客户端：是图形应用程序，向 X 服务器发送绘图和事件请求。此处X客户端位于远程服务器或Docker容器里。

X11 Forward的意思就是服务器或Docker容器把X客户端发送的绘图和事件请求转发给本地机器，本地机器的服务器处理这些请求，并在显示器上显示。

### 本地-远程服务器的X11 Forward

在 X11 forwarding 中，远程服务器上的 X 客户端应用程序通过 SSH 通道将图形显示数据发送到本地机器的 X 服务器。

```bash
ssh -X -p port_num user@remote_server
```

其中，-X代表启用X11转发，-Y代表启用受信任的X11转发。

比较方便的方式是在本地~/.ssh/config文件里定义：

```yaml
Host Alias # 别名
    HostName ip_addr
    User username
    Port port_num
    ForwardAgent yes
    ForwardX11 yes #对应 -X
    ForwardX11Trusted yes # 对应 -Y
```

ssh连接服务器时，直接

```
ssh Alias
```


成功连接后，SSH 会在远程服务器上自动设置 DISPLAY 环境变量，指向本地机器的 X 服务器。例如

```bash 
~$ echo $DISPLAY
localhost:11.0
```

在服务器上安装相关包：

```bash
sudo apt install xauth x11-apps
```

运行测试程序`xclock`， 如果在本地显示器上看到时钟，说明X11 Forward配置成功。

### .Xauthority文件

`.Xauthority` 文件用于存储授权信息，确保只有被授权的客户端才能连接到 X 服务器。这些信息包括：
- Display 信息：显示号（display number）和主机名。
- 密钥（Magic Cookie）：每个显示号对应一个密钥，用于验证客户端的连接请求。


使用ssh连接远程服务器时，SSH 服务器会在你的主目录中生成或更新 `.Xauthority`文件，添加对应的授权条目。当你在远程服务器上启动一个图形应用程序时（例如运行 xclock），该应用程序会检查 DISPLAY 变量，并读取 `.Xauthority`文件中的授权信息。应用程序会使用 DISPLAY 变量中指定的显示号和对应的密钥，尝试连接本地机器上的 X 服务器。

本地机器上的 X 服务器接收到连接请求时，会验证该请求的密钥。如果密钥匹配，则允许连接，图形数据就会通过 SSH 隧道传输到本地显示。

使用`xauth`指令查看和管理`.Xauthority`文件内容。

`xauth list`可以查看默认`Xauthority`文件的信息：

```bash
$ xauth info #本地执行
Authority file:       /run/user/1000/gdm/Xauthority
File new:             no
File locked:          no
Number of entries:    2
Changes honored:      yes
Changes made:         no
Current input:        (argv):1
```

可以看到默认的`Xauthority`文件并非`~`文件夹下的。

`xauth list`可以查看文件中的具体条目：

```
$ xauth list
txszju-GS66/unix:  MIT-MAGIC-COOKIE-1  32a9xxxxxxxxxxxxxxxxxxxxxxxxxxxx
#ffff#7478737a6a752d47533636#:  MIT-MAGIC-COOKIE-1  332a9xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

以上两条指令在`xauth`后加`-f Filename`选项可以查看指定文件的信息

```
xauth -f ~/.Xauthority list
```

`xauth generate`用于生成一个新的X11认证条目，并把它添加到`Xauthority`文件中。如果不加`-f`指令，则添加到默认的`Xauthority`文件中：

```bash
xauth generate :display_number . trusted
xauth generate $DISPLAY . trusted # 使用环境变量$DISPLAY，选择本地当前显示器
```

其中的`.`代表默认的认证协议，为`MIT-MAGIC-COOKIE-1`。


### Docker Container配置X11 Forward

首先把本地默认的`Xauthority`文件复制到`~`目录下：

```bash
cp /run/user/1000/gdm/Xauthority ~/.Xauthority #默认文件由xauth info获取
```

此时客户端在Docker容器中。在本地（宿主机）创建Docker容器时，需要给`docker run`指令加上选项：

```bash
--env="DISPLAY=$DISPLAY" #在容器中使用和宿主机相同的$DISPLAY变量
--env="QT_X11_NO_MITSHM=1"  #禁用共享内存，避免一些问题
--volume="/tmp/.X11-unix:/tmp/.X11-unix:rw" #将主机的 X11 Unix 套接字目录挂载到容器中，实现与宿主机X服务器通信
--volume="$HOME/.Xauthority:/root/.Xauthority:rw" #获取Xauthority认证
```

同样可以用`xclock`验证X11 Forward是否成功。

### Docker Container中的GPU使用

首先需要在宿主机上安装nvidia-container-toolkit:
```bash
sudo apt-get update
sudo apt-get install -y nvidia-container-toolkit-base
sudo apt-get install -y nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```


如果想在Docker容器内使用宿主机的GPU资源，需要再`docker run`时额外添加一些选项：

```bash
--gpus all #指定容器可以访问所有GPU
--runtime nvidia #指定容器使用NVIDIA的运行时环境
--env="NVIDIA_VISIBLE_DEVICES=all" #将所有可见的 GPU 设备传递给容
--env="NVIDIA_DRIVER_CAPABILITIES=all"  #为容器启用所有可用的 NVIDIA 驱动程序功能
```

完成后，在容器内验证是否可以识别GPU：
```bash
nvidia-smi #显示GPU信息即可
```

### 本地连接远程服务器，并在服务器上运行Docker Container

这种情况下的X11 Forward变成了两层，按服务器-Docker容器的顺序配置好Xauthority即可。

注意，每次ssh登录服务器时，服务器上`DISPLAY`变量的值可能会变，而服务器上的Docker容器里的`DISPLAY`还是上次登录时（创建容器时）的值，所以需要把容器里的改为和服务器的相同。

最终在容器里运行`xclock`，可以在本地屏幕上看到。

### 尚未解决的问题

在服务器上使用Docker容器运行`rviz`时，遇到报错：
```
[ WARN] [1716453306.731643029]: OGRE EXCEPTION(3:RenderingAPIException): Unable to create a suitable GLXContext in GLXContext::GLXContext at /build/ogre-1.9-B6QkmW/ogre-1.9-1.9.0+dfsg1/RenderSystems/GL/src/GLX/OgreGLXContext.cpp (line 61)
rviz::RenderSystem: error creating render window: OGRE EXCEPTION(3:RenderingAPIException): Unable to create a suitable GLXContext in GLXContext::GLXContext at /build/ogre-1.9-B6QkmW/ogre-1.9-1.9.0+dfsg1/RenderSystems/GL/src/GLX/OgreGLXContext.cpp (line 61)
[ERROR] [1716453306.731773493]: Unable to create the rendering window after 100 tries.
```

可能和OpenGL版本或者显卡驱动版本有关。



