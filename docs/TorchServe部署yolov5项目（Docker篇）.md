# TorchServe部署yolov5项目（Docker篇）



## 一、前言

​	没啥说的，要不然你也不会对这篇blog感兴趣

## 二、打包mar

1.下载项目

```
#下载分支mini版本（推荐先用mini版本上手，然后用完成版本验证）
git clone -b mini https://github.com/lbz2020/torchserve_yolov5.git
#下载完整版本
git clone https://github.com/lbz2020/torchserve_yolov5.git
```

2.进入项目主目录 torchserve_yolov5（终端命令窗口，我是在Mac OS 下操作的，没能照顾到Windows用户）

​	**注意：如果没有特别说明或指定,全文命令的相对路径都是在项目主目录进行**

3.安装依赖

```
#升级pip
pip install --upgrade pip
#安装依赖（使用国内镜像加速）
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```

4.进入子目录 yolov5，将.pt文件转成.torchscript文件

```
#打开子目录 yolov5
cd yolov5
#由.pt文件导出.torchscript文件
python export.py --weights yolov5s.pt --include torchscript
```

5.回到项目主目录

```
#返回上一级目录
cd ..
```

6.打包成.mar文件

```
#将所有文件打包成.mar文件
torch-model-archiver --model-name demo --version 1.0 --serialized-file yolov5/yolov5s.torchscript --handler yolov5_model_handler.py --extra-files index_to_name.json
```

torch-model-archiver参数解释

- --model-name ：模型名字 demo

- --version：模型版本号 1.0

- --serialized-file：由.pt生成的.torchscript文件 yolov5/yolov5s.torchscript

- --handler 针对yolov5模型的处理器，包括预处理、推理、后处理等过程 yolov5_model_handler.py

- --extra-files 一起打包的额外文件，这里包含了标签索引的映射文件 index_to_name.json

  **执行上面命令后，会在项目主目录下生成一个`demo.mar`文件**

7.在项目主文件夹下创建一个文件夹`model-store`，接着把`demo.mar`文件放到文件夹`model-store`里面去。目的是待会用容器启动的时候做一个文件夹的映射。

```
#注意：以下是Linux命令!!!
#创建一个文件夹model-store
mkdir model-store
#将demo.mar文件移到到文件夹model-store中
mv demo.mar ./model-store
```

## 三、docker镜像

这里不建议直接pull官方提供的docker镜像

```
#拉取torchserve官方镜像（不用管）
docker pull pytorch/torchserve
```

注意：为什么不直接使用官方提供的镜像呢？因为官方提供的镜像里面只提供最基本的python依赖包。我们在写`handler`文件的时候一般都会用到其他依赖的，例如opencv等。这些依赖都是官方docker镜像没有提供的。后面我会做一个公开的镜像放在dockerhub上，以便供大家拉取。



1.假设我们都下载并安装了[docker](https://www.docker.com/)

2.制作TorchServe的docker镜像

接下我们制作要做的其实就是在官方`Dockerfile`里添加yolov5的一些python依赖

```
RUN python -m pip install matplotlib>=3.2.2 numpy>=1.18.5 opencv-python>=4.1.1 Pillow>=7.1.2 PyYAML>=5.3.1 requests>=2.23.0 scipy>=1.4.1 torch>=1.7.0 torchvision>=0.8.1 tqdm>=4.64.0 tensorboard>=2.4.1 pandas>=1.1.4 seaborn>=0.11.0 ipython psutil thop>=0.1.1 -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

​	a.修改Dockerfile文件

​		找到Dockerfile文件：在主目录/serve/docker文件夹里面

​		然后打开Dockerfile文件，定位到第76行的地方，把下面的命令添加上去

```
RUN python -m pip install matplotlib>=3.2.2 numpy>=1.18.5 opencv-python>=4.1.1 Pillow>=7.1.2 PyYAML>=5.3.1 requests>=2.23.0 scipy>=1.4.1 torch>=1.7.0 torchvision>=0.8.1 tqdm>=4.64.0 tensorboard>=2.4.1 pandas>=1.1.4 seaborn>=0.11.0 ipython psutil thop>=0.1.1 -i https://pypi.tuna.tsinghua.edu.cn/simple/
```

​	b.在终端中执行命令

```
#打开目录path，主目录/serve/docker
#必须定位到docker目录中，否则会有很多未知问题
cd serve/docker
#创建镜像（Linux）
./build_image.sh
```

​	windows有很多bug，而且制作镜像的时间较长，大概一个多小时创建完，然后没去尝试了。

**two thousands years later ------------------------->>>>**

在docker面板上可以看到创建成功的Docker镜像 `pytorch/torchserve:latest-cpu`

**end --------------------------------<<< ！**

## 四、部署

​		docker命令要会**[亿点点](https://www.runoob.com/docker/docker-command-manual.html)**哈哈哈哈哈哈哈哈哈哈

1.启动docker镜像

```
#启动命令格式（dev环境，不会后台运行）
docker run --rm -it -p 宿主主机端口:容器内部端号 --name 容器名字 -v 宿主主机文件夹:容器内部文件夹 镜像名:镜像标签号

#自己本地制作的镜像
docker run --rm -it -p 8080:8080 -p 8081:8081 --name object_detector -v model-store文件夹路径:/home/model-server/model-store pytorch/torchserve:latest-cpu
#从我远程pull的镜像
docker run --rm -it -p 8080:8080 -p 8081:8081 --name object_detector -v model-store文件夹路径:/home/model-server/model-store pytorch/torchserve:v1.0

#记得改成自己的路径哦
```

​	model-store文件夹路径：步骤二 - 7里提到的 model-store 文件夹

​	容器内部文件夹路径 `/home/model-server/model-store` 这个路径是固定的，在镜像build的时候创建好的



so，容器启动完成就可以了吗，莫得

还要注册模型，要不然容器里面啥都没。我们通过restapi操作。



2.查询已经注册的模型

```
#查询已经注册的模型
curl "http://localhost:8081/models"
```

3.注册模型并且为模型分配资源

```
#注册模型并且为模型分配资源
curl -v -X POST "http://localhost:8081/models?initial_workers=1&synchronous=false&url=demo.mar"
```

4.查看注册模型的基本信息

```
#查看注册模型的基本信息
curl "http://localhost:8081/models/demo"
```

5.推理（见证奇迹的时候到了）

```
curl http://localhost:8080/predictions/demo -T docs/zidane.jpg
```

6.注销模型(不用管)

```
curl -X DELETE http://localhost:8081/models/模型名称
```



然后，然后，就恭喜你啦。代码和其它啥的我都放在[GitHub](https://github.com/lbz2020/torchserve_yolov5.git)上。上面的每一步都是自己操作一步完后写下，然后再操作下一步。所以项目是没问题的。。。。。。



文档我会继续更新的，有很多细节好像都没有提及到。还有一些操作的图片整理到文章里后可能会更加清晰。	--20221012 init



## 五、总结

​	没啥好总结的，一个晚上一个奇迹。有时候看着[官方文档](https://github.com/pytorch/serve/tree/master/docs) 就很累。。。。多看官方文档。有啥问题可以给我留言

​	

​	TorchServe难点在于handle.py怎么写，yolov5本身就是工程化的项目，所以handle里面写了许多工程化的处理。其实我这个项目还是不够完善的。。。

## 六、issue

**1.官方提供的镜像有坑？**

官方可不能为每一种模型都提供对应环境的模型吧，而且每种模型都有不同的依赖。

**2.为什么选择TorchServe部署？而不是选择TF Serving部署，或者选择Flask**

根本原因是自己太菜了。。。

直接原因Flask框架部署AI模型存在[内存泄漏问题](https://zhuanlan.zhihu.com/p/441992730)，导致另外一个AI项目在生产环境中运行特别不稳定，而且这个Flask框架简单使用还行，在并发量高一点的场景就垮了。

TF Serving 不知道好不好用，反正我没用了。我只是一个小菜鸡！

**3.想到啥问题我再更新，给我留言也可以**



