# OpenShift 容器平台上的 Jupyter 笔记本帮助落实数学实验教学  

前些日子, 上初中的女儿让我帮忙在 MatePAD 上调试她的数学老师设计的实验的程序, 很遗憾, 我没找到移动终端上可用的 SciPy<sup>1</sup> 环境, 这个数学实验没能在课上进行.  

我非常赞赏这个数学实验. 将实验引入数学教学在学界讨论了二十年, 已经达成了共识<sup>2,3,4</sup>, 但在落实的时候, 受到了诸多因素的制约, 毕竟没法安排数学课都在计算机教室上. 那么在移动设备普及的今天, 是否有方案让每个学生在班级教室通过移动设备的浏览器就能进行数学实验呢?  

OpenShift 容器平台<sup>5</sup>上的 Jupyter 笔记本<sup>6</sup>是一个成熟的开源<sup>7</sup>方案. 基于平台的能力, 每个学生用移动设备的浏览器访问校内数学实验系统时, 都会启动一个独立的实验环境的容器, 互不影响, 简单的操作就可以实现动态的可视化; 并且资源占用少, 一台高配的 PC 机就能支持一个班的学生进行实验.  

下面以使用蒙特·卡罗方法<sup>8</sup>模拟计算圆周率的实验为例, 展示使用 OpenShift 容器平台上的 Jupyter 笔记本进行数学实验的过程. 无论是 iPad 上的 safari 浏览器, 还是 MatePAD 上的 chrome 浏览器, 都能够正常进行实验.  

## 搭建基于 OpenShift 容器平台上的 Jupyter 笔记本的校内数学实验系统  
请参考[红帽官网的安装手册](https://access.redhat.com/documentation/zh-cn/openshift_container_platform/4.2/html/installing/index)和众多的博客文章, 如, [OpenShift 4.2 离线安装补充记录](https://www.cnblogs.com/ericnie/p/11764124.html), 进行安装, 4台 PC 机即可搭建一个容器云环境.  

Jupyter 笔记本的镜像, 请参考[jupyter-on-openshift 项目](https://github.com/jupyter-on-openshift/)进行构建.  
> 需要注意的是, 若要 Jupyter 笔记本在页面上实现动态的可视化效果, 则需要在构建镜像时安装 FFmpeg<sup>9</sup> 软件.  

在控制台页面创建 JupyterHub 应用模板:  
![上传 JupyterHub 应用模板]()  

然后通过模板创建 JupyterHub 应用:  
![输入配置参数, 创建 JupyterHub 应用]()

至此这个校内数学实验系统已经就绪.  
![系统就绪](./jupyterhub-screenshots/application_system_for_math_experience.png)

## 进行实验  
用浏览器访问[实验系统的网址](), 系统会自动初始化一个实验环境的 POD:  
![自动启动实验环境](./jupyterhub-screenshots/auto_started_lab_env.png)
![实验环境就绪](./jupyterhub-screenshots/lab_pod_ready.png)

创建一个 python 笔记本:  
![创建 python 笔记本](./jupyterhub-screenshots/new_python_notebook.png)

按步骤运行实验代码:  
```
```
![第一象限的圆](./jupyterhub-screenshots/Pi_simulated_by_Monte_Carlo_method-axis.png)

```
```
![动态效果](./jupyterhub-screenshots/Pi_simulated_by_Monte_Carlo_method.mp4)

完成实验后停止实验环境(或者由老师在课后统一清理):  
![进入控制面板](./jupyterhub-screenshots/goto-control-panel.png)
![停止实验环境](./jupyterhub-screenshots/shutdown-lab-pod.png)


## 参考
1. SciPy 是一个开源的 Python 算法库和数学工具包, https://zh.wikipedia.org/wiki/SciPy
2. 《素质教育的新途径——开设数学实验与数学建模课》, 2000年, 何铭,杨振华,邱中华,孔告化,许立炜,胡国雷,唐加山, 电子高等教育学会2000年学术年会论文集
3. 《数学实验与数学建模》, 2001年, 姜启源, 数学的实践与认识第31卷第5期
4. 《中学数学实验初探》, 2002年, 许德刚, https://cdmd.cnki.com.cn/Article/CDMD-10718-2003083035.htm
5. OpenShift 容器平台是企业级的云计算平台软件产品, https://www.redhat.com/zh/technologies/cloud-computing/openshift
6. Jupyter 笔记本是一个功能强大的基于Web的交互式计算环境, https://zh.wikipedia.org/wiki/Jupyter  
7. 开源是开放源代码(Open Source)的简称, 是一种提供程序源代码的做法, https://zh.wikipedia.org/wiki/%E5%BC%80%E6%94%BE%E6%BA%90%E4%BB%A3%E7%A0%81
8. 蒙特·卡罗方法(Monte Carlo method), 也称统计模拟方法, 是一种以概率统计理论为指导的一类非常重要的数值计算方法, https://baike.baidu.com/item/%E8%92%99%E7%89%B9%C2%B7%E5%8D%A1%E7%BD%97%E6%96%B9%E6%B3%95/8664362?fr=aladdin
9. FFmpeg是数字多媒体处理工具, https://baike.baidu.com/item/ffmpeg/2665727
