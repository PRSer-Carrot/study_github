# 辽宁项目

​		使用激光点云数据、全景图像数据、惯导数据实现路面标志(车道线)自动提取，自动生成含有大地坐标的shp文件。



## ⭐ 所用数据解析

```
.
├── 点云数据
│   ├── 200225_045712-13-01-33-202_refine_SHCS.las			# 2020年02月25日12:57:12至13:01:33.202
│   ├── ......
│   └── 2020522_01_05_38_000-09-14-04-971_refine_SHCS.las	# 2020年05月22日09:05:38.000至09:14:04.971
│ 
├── 轨迹点
│   └── INSPoseSHCS.txt		# 轨迹点数据
│ 
└── 全景照片
    ├── 00000000000000091.jpg
    ├── ......
    └── 00000000000001043.jpg
```

- 点云数据如下图所示，平均单帧点云数据大小为**300M**，包含**926w**个点，采集时间持续**22s**，包含轨迹点**14个**。

<img src=https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220914100512566.png alt="image-20220914100512566" style="zoom: 50%;" />

- 轨迹点数据如下所示，其中每列字段的含义分别为：**图像编号**、**采集时间(UTC)**、**GPS时间**、**轨迹点的大地坐标XYZ**、**纬度**、**精度**、**轨迹点相对于大地坐标的旋转角RxRyRz**。

<img src="https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220914101433175.png" alt="image-20220914101433175" style="zoom:50%;" />

<img src=https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220914101433175.png alt="image-20220914101433175" style="zoom:50%;" />

- 全景照片数据如下所示，图片名称与轨迹点编号一一对应，分辨率大小为**8192×4096**，由6张平面照片拼接而来。

<img src="https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220914103240868.png" alt="image-20220914103440997" style="zoom: 80%;" />

---









## 1. lid2cam2lid项目

```
建立点云图像坐标映射,实现点云与图像坐标之间的转换(单位球)
 1.输入全景图像路径名，轨迹点文件，获取图片名称更新大地坐标到该点的旋转矩阵

# 通过图像像素获取三维点坐标
 2.转换点云坐标到相机坐标系(先平移再旋转)，根据水平垂直角得到三维点的像素坐标
 3.向全景索引图像对应像素位置赋值对应点的大地坐标
 4.将全景图像在六个面上进行切分并存储成tiff输出
 5.读取对应方向的索引图像，取其像素值即大地坐标

# 通过三维点坐标获取图像像素
 2.转换点云坐标到相机坐标系(先平移再旋转)，计算水平角垂直角
 3.通过水平垂直角向对应方向上投影获取其像素坐标
 4.若无法投影到对应方向上，则返回默认值(0,0)
```

水平角垂直角向对应方向上投影的过程中用到的数学几何知识如下所示（引用正弦定理）

![image-20220915185609041](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220915185609041.png)

点云向全景图片上的投影结果：

<img src=https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220914171751610.png alt="image-20220914171751610" style="zoom: 50%;" />

```
三维点与图像坐标测试用例，实际误差在一个像素左右
	xyz(11994.375977, 1737.564575, 3.415150)	front_uv(563,605)
	xyz(11989.883789, 1740.128784, 3.675979)	front_uv(563,655)
	xyz(11960.062500, 1804.367310, 3.518280)	back_uv(530,478)
	xyz(11961.018550, 1804.924805, 3.536668)	back_uv(530,494)
```

测试某一方向上索引图像与原图像的误差如下：

<img src="https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220914203218465.png" alt="image-20220914203218465"  />

---







## 2. pan2cubic项目

```
批量将全景图像转化为各个方向的平面图像(单位球)
 1.读取文件夹路径，获取全景图像文件名进行遍历
 2.根据目标方向输出的size构建mapx/mapy
 3.获取方向中心位置，遍历目标方向输出size行列，求其对应的全景像素坐标
 4.将求得的像素坐标uv赋给对应的mapx/mapy
 5.使用opencv的remap函数进行重映射生成方向平面图
```

方向平面点向全景图像投影过程中角度求取涉及到的几何知识如下 [参考](https://blog.csdn.net/qq_30832659/article/details/52494713)

<img src="https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220915201952435.png" alt="image-20220915201952435" style="zoom:80%;" />

90度视角切分结果：

![image-20220915202405484](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220915202405484.png)

俯视图切分结果：

![image-20220915202651326](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220915202651326.png)

---







## 3. LN_Project项目

```
整理好的原方法的技术路线，生成强度图并赋予斜率
 1.输入各数据文件路径，找到对应单帧点云的轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.连接轨迹点成轨迹线，从轨迹线垂直方向找路肩(相邻格网高差或detz)
 5.过滤路肩外的点，并对格网进行连通性分析(8邻域)
 6.找距离强度均值作为阈值筛选掉强度过低的点云(距轨迹点平面距离)
 7.将剩余点云强度归一化到0~255(距离改正 不同距离区间归一化方式)
 8.使用格网高差滤掉边界点及道路中车辆，并进行连通性分析
 9.根据估算点数N，取出强度最强的N个点作为Roadmarks
 10.计算整段轨迹线斜率用于后续成图
 11.路标点转换为图像并进行灰度拉伸，然后进行二值化
 12.生成强度灰度图并输出
```

生成的强度灰度图如下：(200225_045712-13-01-33-202_refine_SHCS_trak_2.193841.bmp)

![image-20220916215549356](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220916215549356.png)

---







## 4. Route 1.0项目

```
用图像语义分割后的二值图进行判定是否是路面标志
 1.输入各数据文件路径，找到对应单帧点云的轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.连接轨迹点成轨迹线，从轨迹线垂直方向寻找路肩(相邻格网高差或detz)
 5.过滤路肩外的点，并对格网进行连通性分析(8邻域)
 6.找距离强度均值作为阈值筛选掉强度过低的点云(距轨迹点平面距离)
 7.将剩余点云强度归一化到0~255(距离改正 不同距离区间归一化方式)
 8.使用格网高差滤掉边界点及道路中车辆，并进行连通性分析
 9.读入该帧点云对应的语义分割图像信息(位置、旋转矩阵、图片路径名称)
 10.将点云点循环向各语义分割图投影，若投影成功则留下
 11.根据估算点数N，取出强度最强的N个点作为Roadmarks
 12.路标点转换为图像并进行灰度拉伸，然后进行二值化
 13.利用图像处理的方法对道路标线进行提取并输出shp
```

全景图像切分为方向平面图并进行语义分割后得到的二值图如下：

![image-20220916123545035](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220916123545035.png)

利用语义分割二值图判定路面标志的结果如下：

![image-20220916162748859](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220916162748859.png)

图像处理方法输出shp结果如下：

![image-20220916163939842](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220916163939842.png)

---







## 5. Route 2.0项目

```
使用(全景)图像向点云格网贴纹理
 1.输入各数据文件路径，找到对应单帧点云的轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.遍历格网，取格网内第1点向各全景图像投影
 5.若成功投影则将投影到的像素值赋予格网像素
 6.保存成图并输出
[注]格网像素值为最先投影到的全景图像素值，投影过程中要限制区间排除车顶
```

图像向点云格网贴纹理结果如下：

![image-20220916203524132](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220916203524132.png)

---







## 6. Route 3.0项目

```
建立相对位置RGB索引图象,根据图像语义分割的二值图来生成三维坐标点
 1.输入各数据文件路径，找到对应单帧点云的轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.使用lid2cam2lid中图像像素获取三维点坐标方法建立相对位置索引
 5.遍历该帧语义分割二值图，若像素值非零则计算其三维坐标成点
 6.输出所有人工生成的"路面标志点"
[注]不同的相机拍摄位置其相对坐标索引在大地坐标系下是不同的，所以行不通，
需要考虑到图片的位置姿态，即每张图都要建立一个索引图，太繁琐了pase
```

全景图像切分为方向平面图并进行语义分割后得到的二值图如下：

![image-20220916123545035](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220916123545035.png)

使用lid2cam2lid方法建立相对位置索引图如下所示：

<img src="https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917164030486.png" alt="image-20220917164030486" style="zoom: 50%;" />

用此方法生成的人工“路面标志点”如下所示：

![image-20220917164243322](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917164243322.png)

---









## 7. Route 4.0项目

```
纯点云方法生成强度图(cv),高度图(imgz)及格网坐标属性(coordinate)
 1.输入各数据文件路径，找到对应单帧点云的轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.连接轨迹点成轨迹线，从水平垂直方向寻找路肩(相邻格网高差或detz)
 5.过滤路肩外的点，并将路肩点强度值设为最高666666
 6.找距离强度均值作为阈值筛选掉强度过低的点云(距轨迹点平面距离)
 7.将剩余点云强度归一化到0~254(距离改正 不同距离区间归一化方式)
 8.使用格网高差滤掉边界点及道路中车辆，并进行连通性分析
 9.根据估算点数N，取出强度最强的N个点作为Roadmarks
 10.路标点转换为图像并进行灰度拉伸，输出强度图高度图参考坐标
 11.利用图像处理的方法对道路标线进行提取并输出shp(dll)
还包括通过格网中点的高差/高度来进行归一化并进行边缘检测找路肩的尝试
```

输出数据如下所示：

![image-20220917175840925](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917175840925.png)

其中点云强度灰度图如下：

![image-20220917180032677](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917180032677.png)

格网**高差**边缘检测结果：

![image-20220917180253141](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917180253141.png)

格网**高度**边缘检测结果：

![image-20220917180455197](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917180455197.png)

---







## 8. Route 4.1项目

```
点云提取输出强度图等信息以及生成dll(点云找路边线)
 1.传入轨迹点文件路径/点云文件夹路径/点云名，获取该帧轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.连接轨迹点成轨迹线，从水平垂直方向寻找路肩(相邻格网高差或detz)
 5.过滤路肩外的点，并将路肩点强度值设为最高666666
 6.找距离强度均值作为阈值筛选掉强度过低的点云(距轨迹点平面距离)
 7.将剩余点云强度归一化到0~254(距离改正 不同距离区间归一化方式)
 8.使用格网高差滤掉边界点及道路中车辆，并进行连通性分析
 9.根据估算点数N，取出强度最强的N个点作为Roadmarks
 10.路标点转换为图像并进行灰度拉伸，输出强度图高度图参考坐标
 11.利用图像处理的方法对道路标线进行提取并输出shp(dll)
[注]包含使用C++生成dll并用python调用测试实验部分
```

生成的dll中包含下列四个函数：

![image-20220918160504865](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220918160504865.png)

输出数据如下所示：

![image-20220917175840925](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917175840925.png)

其中点云强度灰度图如下：

![image-20220917180032677](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917180032677.png)

---







## 9. Route 5.0项目

```
点云提取输出强度图等信息以及生成dll(图像找路边线)
 1.输入各数据文件路径，找到对应单帧点云的轨迹点信息
 2.通过轨迹点信息对点云进行初筛(高度和距离)
 3.对初筛后得到的点云进行格网化便于后续处理
 4.过滤道路内车辆，输出高度图zpicture
 5.通过格网detz构造路肩边缘图(高差边缘检测)
 6.连接轨迹点成轨迹线，并将轨迹线转换为格网索引
 7.由轨迹线找边界精修路肩并更新格网状态(左右/上下找 路宽及路肩限制)
 8.找距离强度均值作为阈值筛选掉强度过低的点云(距轨迹点/线平面距离)
 9.将剩余点云强度归一化到0~254(距离改正 不同距离区间归一化方式)
 //根据估算点数N，取出强度最强的N个点作为Roadmarks(弃用)
 10.路标点转换为图像并进行灰度拉伸，输出带路边线的强度图及参考坐标
 11.使用yolov5对强度图进行检测并将对应信息输出
 12.对道路标线及路肩进行提取并输出shp(dll)
[注]生成的dll时用的是_rad方法
```

其中\_dis为使用距离**轨迹线**的距离，\_rad为使用距离**轨迹点**的距离

![image-20220917213011953](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917213011953.png)

输出文件夹如下：

![image-20220917214055379](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917214055379.png)

其中\_dis结果如下所示：

![image-20220917214325155](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917214325155.png)

其中\_rad结果如下所示：

![image-20220917215315211](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220917215315211.png)

---







##  10. pycode项目

```
使用python调用生成的dll文件，从点云中提取输出shp
 1.加载动态库文件creat_dll_up.dll和createshp.dll
 2.选择点云文件夹路径和轨迹点文件路径，遍历逐帧点云进行处理
 3.step one 输入路径，输出强度图/高度图/参考坐标
 4.step two用 yolov5-5.0检测车道线并输出检测框信息
   ├──4.1强度图切成640*640的小图进行检测
   └──4.2将检测结果整合到相对大图的像素坐标
 5.step three 根据检测框信息及强度图生成shp并输出
[注]训练实线检测过程中可能会过拟合，step_one的dll用的是_rad方法
```

输出文件夹如下所示：

![image-20220918174554822](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220918174554822.png)

过程强度图：

![image-20220918174906207](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220918174906207.png)

shp结果图：

![image-20220918175948727](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220918175948727.png)

![image-20220918180104717](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220918180029723.png)

![image-20220918180148183](https://cdn.staticaly.com/gh/PRSer-Carrot/ImageBed@main/img/image-20220918180148183.png)

---

2022/9/18
