# TorchServe_Yolov5（Docker）

 快速开始

```
#下载依赖
git clone https://github.com/lbz2020/torchserve_yolov5.git

#进入项目主目录 torchserve—yolov5(所有路径都为相对路径，所以终端当前目录必须是项目主目录，即torchserve_yolov5)
cd

#安装依赖（使用国内镜像加速）
pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple

#拉取docker镜像
docker pull lbz1999/torchserve-yolov5:v1.0

#启动容器
docker run --rm -it -p 8080:8080 -p 8081:8081 --name object_detector -v model-store:/home/model-server/model-store lbz1999/torchserve-yolov5:v1.0

#注册模型并且为模型分配资源
curl -v -X POST "http://localhost:8081/models?initial_workers=1&synchronous=false&url=demo.mar"

#推理
curl http://localhost:8080/predictions/demo -T docs/zidane.jpg
```



教程：https://blog.csdn.net/weixin_45710229/article/details/127287410



有啥问题可以直接在[issues](https://github.com/lbz2020/torchserve_yolov5/issues)留言，或者在微信：flybirdstudio 私信我

