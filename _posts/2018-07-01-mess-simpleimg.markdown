---
layout: simple
title: Simple Image
img: simpleimg/title1.jpg
date: 2018-07-01
description: CG homework.
category: Mess
permalink: /:categories/:title.html 
notshow: 1
---


# 一、原文

### 1. 论文信息：
《Stylization and Abstraction of Photographs》，Doug DeCarlo & Anthony Santella，SIGGRAPH 2002。   

 <br>

### 2. 论文目标：
​        针对一张照片，进行图片抽象化和风格化，最后简化为简单的线条和色块。并且图片能够在更受目光关注的地方被更加详细地描绘，而在其他区域只进行粗糙的绘制。产生的效果如下图所示。

原图：
![原论文原图](../assets/img/simpleimg/origin1.JPG)
结果图：
![原论文结果图](..\assets\img\simpleimg\origin2.JPG)  

 <br>

### 3. 论文流程

##### ① 边缘检测

​        论文参考 Meer 和 Georgescu 在2001年的《Edge detection with embedded conﬁdence》，采用Canny边缘检测算法的变体来查找图像边缘。Canny算法可以产生更干净的图像边缘。而该变体相对于Canny的优点在于能发现更微弱的边缘，同时忽略了一些在浓密纹理区域中的边缘。

##### ② 图像分割

​         论文又参考了 Comaniciu 和 Meer 在2002年的论文《Mean shift: A robust approach toward feature space analysis》进行图像的颜色分割。这篇论文将meanshift应用于图像像素的聚类，根据像素的色彩和位置，将图片划分成不同的色块，并消除了较小的颜色区域，因此最终画面能够更加整洁。

##### ③ 建立图像层级

​        首先根据尺度空间的理论，论文使用图片颜色分割的结果构造多层图像金字塔。接着，论文提出，在层与层之间，用相应位置像素的空间和颜色关系，可以建立一个类似于树的结构。精致分块的图像层中，根据位置和颜色的相似度，一些色块区域将作为上一层相对粗糙的图像层级中的某一色块的孩子。

##### ④ 使用视觉模型

​        论文提出了一种符合人的视觉模型，根据人在图片上的聚焦中心和对应的聚焦时间，来判断某一区域的画面是否应该详细绘制。如果函数认为这一块区域能够被人注视到，就要刻画该区域的所有子树；反之，若认为某一块区域无关紧要，就直接刻画该区域，不再遍历子节点。

##### ⑤ 图像平滑

​        经过上述步骤之后，图片就按照细节层次绘制好了，但是图片颜色区域边缘以及线条还不是那么光滑。因此，论文又参考 Finkelstein 和 Salesin 于1994年的《Multiresolution curves》对边缘线进行低通滤波，最终得到简化的插图风格图像。  

  <br>

# 二、实现过程

### 0. 数据结构

##### ① 图片数据

​        图片首先在本地中选择，加载到画布时图片数据为一个 [ r, g, b, a, r, g, b, a…… ] 格式的一维数组。为了处理方便，图片最好是变成矩阵的格式，因此写了两个函数：

- `ImgToMatrix(src)`：输入参数为画布图片，不仅包含像素数据还包含width和height信息，输出{ r, g, b }三个矩阵，对应R、G、B三个通道，而舍弃透明度通道。其中每个矩阵相当于一个高宽等于图片的二维数组。

- `MatrixToImg(mat, canvas)`：输入参数是上述格式的三维矩阵和界面上的画布元素，函数将数据转化为画布图片的数据格式并直接显示在该画布上。  

##### ② 金字塔

​        程序形成的金字塔有两个。

![金字塔结构示意](../assets/img/simpleimg/py.JPG)

​        一个是用于显示图片的segPy数组。segPy的每个元素每对应金字塔的一层，第0层是原图，层号越大，图片越小，划分越粗糙。每层中是一个三通道的图片矩阵，即该层的图像。

​        另一个是用于记录分割结果的idPy数组。同样，idPy每个元素每对应金字塔的一层，每层中的数据记录了：

- `idmap`：该层的分割结果（用整型的编号，同一分割区域的像素用同一整数记录）
- `idmax`：最大的编号
- `idNum`：每个编号区的像素个数（长度等于最大编号的数组）
- `idAvgColor`：每个编号区的平均颜色（长度等于最大编号的数组）

##### ③ 孩子表

​        嵌套的数组。首先每个元素每对应金字塔的一层，每层中有一个长度为该层idmax大小的数组，数组的每个元素是该层该编号的孩子列表。孩子列表中的每个元素是[ levelID, segmentID ]，记录孩子是哪一层的哪一个编号。

![孩子表示意](../assets/img/simpleimg/child.JPG)

##### ④ 邻居表

​        矩阵数组。首先每个元素每对应金字塔的一层，每层是一个相邻矩阵，矩阵的每一个值记录两个块之间有多少邻接像素。

  <br>

### 1. 边缘检测

##### ① 原图转化为灰度图像 

由于Yuv空间的Y通道能较好的体现图片的灰度值，因此先将原图的RGB数据转化到YUV空间，只保留Y通道，对于每一个像素，Y值的计算式子大约为 0.299 * r + 0.587 * g + 0.114 * b。函数名如下：

- `ToGrey(src)`：输入参数是画布的图像数据，输出灰度矩阵。

##### ② 高斯模糊 

为了去除噪声，需要对图片进行一次高斯模糊，实际上就是先生成一个高斯核矩阵，再对图片进行卷积。主要函数如下：

- `GenGauss(halfSize, sigma, rslt)`：生成高斯矩阵，由于要限制高斯矩阵为奇数长宽以便运算，因此输入的参数选为半边长，即（实际边长-1）/2。sigma为方差，rslt是显示高斯矩阵的画布，如果为0，则不发挥作用。

- `ToBlur(src, gauss)`：即使用高斯核矩阵进行卷积。
- `MyConvolution(kernel, src)`：卷积。做法是先将原图向四周扩大核矩阵半边长的长度，扩大的像素的值取原图边缘的像素颜色值。这样卷积之后得到的图片还是原来的大小。

##### ③ 计算梯度 

用两个方向的Sober算子进行卷积，计算水平的梯度、垂直梯度后，应该要根据如下公式计算： 

梯度角度：Θ=atan（垂直 / 水平） 

梯度大小：G= √（垂直² + 水平²） 

但是简化计算起见，我的梯度大小直接使用了垂直 + 水平的值。角度计算出来以后要量化成四个角度，0、45、90、135。更精确的方法是保留角度，对周围的点颜色值进行插值后，然后比较。不过我用的还是量化。 

- `ToGrad(src)`：输入灰度矩阵，输出梯度强度mag和方向dir两个矩阵。

##### ④ 非极大值抑制 

用上面计算得到的梯度方向，在八领域之内对每个点和两边进行比较。如果是三个值里面最大的，就保留，如果不是，就变成0。 

- `ToNMS(mag, dir)`：输入梯度强度mag和方向dir两个矩阵，直接在mag矩阵上修改梯度值，因此没有返回值。
- `Suppress(src, flag, i, j)`：即根据位置判断该像素点与八邻域的颜色值大小关系。

##### ⑤ 双阈值处理 

阈值根据梯度值的直方图来选取。因此要先计算直方图，根据手动设定的比例，找到对应的颜色值作为阈值，然后低于低阈值的舍去，高于高阈值的保留。

针对中间阈值的点，则从某一个高阈值开始，依次对八领域找连通点，如果是中间阈值的，就保留，并存到全局的一个Array，直到它被访问到，再把它从Array中删去。如此循环，直到Array为空，则该连通域被确定，该连通域中遍历到的中间阈值点也全部保留了。

- `ToDoubleThreshold(src, min, max)`：输入灰度图像和高低阈值，输出最后的边缘检测结果。

- `MyFindConnect(src, y, x, flag)`：从当前点出发找连通域的函数，借助一个全局的Array是为了避免递归。

- `MyHist(src, max, bins)`：计算直方图，max为规定的最大颜色值，如果某颜色值大于max，直接归到最后一个bin。因为我写的是均匀分布的直方图，所以这么做是为了防止最大的那个数太大，以至于数据集中在较小的bin上。但我的程序里暂时使用了1000，包括了所有可能的梯度强度值。输出是两个数组，一个记录每个bin的区间中值，另一个是这个bin里面的数量。

- `MyHistBin(rghist, numhist, n)`：按比例计算某个数据在直方图的哪个bin上面。

##### ⑥ 去除短边

再次利用上一步清空的Array，在各线条的连通域中，计算连通域的总像素数。如果低于阈值，则去掉该边。

- `ToCutShort(src, short)`：输入灰度图像和短边阈值，返回结果。  

  <br>

### 2. 图像分割

##### ① 图像颜色聚类

采用meanshift算法，进行图像像素的聚类。

遍历每一个像素，对每一个像素，判断它经过meanshift之后的颜色值。在每一个像素的判断中，都有若干次迭代。每一次迭代中，都会选取一个以模点为中心的（模点初始值为当前像素）窗口，窗口理论上应该为圆形，但是一般为了计算方便都选为正方形窗口。在这个正方形窗口中寻找颜色值在一定容差范围内的点，统计这些点的重心和平均颜色，作为下一次迭代的中心。如果这个窗口位置变得稳定，则结束迭代，将这时的窗口中心颜色赋予当前像素。

因为代码中已经做出了很详细的注释，所以下面就放一段没有任何细节的大致过程了：

```pseudocode
function ToMSSeg( iterationTimes, windowSize, colorThreshold, colorDistance ) 
{ 
    for every pixel in src {
        for each iteration {
            for each pixel in window {
                if (窗口中当前像素颜色值 - 模点颜色值 <= colorThreshold) {
                    记录窗口中当前像素点;
                }
            }
            if (count == 0) break;
            模点颜色值 = 窗口像素记录点的平均颜色;
            模点坐标值 = 窗口像素记录点的平均坐标;
            
            新的窗口中心坐标 = 模点坐标值;
            if(上一次窗口中心颜色 - 模点颜色值 <= colorDistance && 
               上一次窗口中新坐标 == 模点坐标值){
                break;
            }
         }
         当前像素颜色值 = 模点像素颜色值;
    }
}
```

实际写这个函数的时候，参考了OpenCV的meanshift函数的写法，但是我的返回值为一个包括{ r, g, b, x, y }（rgb为颜色值，xy为坐标值）的矩阵结构，也就是说最后不仅把模点像素颜色值赋给了当前像素颜色值，还把模点像素坐标值保存在了结果图像，便于后一步填色处理。

在OpenCV的meanshift函数中，”窗口中当前像素颜色值 - 模点颜色值” 需要计算三次平方，为了提高速度所以它的计算方式是把 -255 至 512 的平方结果都存在一个tab数组中，因此我的函数也仿照了这个写法。由此出现的问题是，由于JS对数组下标不会主动化整，在后期的计算中，颜色值出现小数，导致访问不到tab中的对应结果，而显示的时候，JS又反而能够自动把颜色值对齐到0-255的整数。这体现为只有金字塔第0层能成功聚类，而后面几层都没有反应。当时找了很久原因，最后把所有颜色运算结果都转为整数值方得到解决。

##### ② 图像颜色重填充

重新遍历上一步的结果，对像素进行分类。在OpenCV中采用FloodFill方法进行颜色填充，这个在填充的时候只考虑邻域的像素是否接近，而且填充的是随机颜色。由于我在上一步保存了模点坐标值，所以在这里就按照这个坐标值的临近程度来判断是否为同一个区域了。如果在同一区域中，就把它们赋予相同的编号。

编号完成之后还会再进行多次遍历，计算出每个类别的平均颜色和像素总数。

- `ToDivided(src, sp, sr)`：输入参数是原RGB图像矩阵和一定的颜色、空间误差范围，返回值是记录每个像素的类别号的矩阵、最大的id类别号、每个id对应的平均rgb颜色。

<br>

### 3. 图像层级建立

##### ① 图像金字塔

根据论文，图像按照根号2的比例建立金字塔，金字塔也就是将原图置于底层，依次对下层进行高斯模糊、下采样形成上层。每形成一层，就对该层进行中值滤波，然后图像分割。

我的中值滤波就是对每个3*3窗口，取RGB颜色的中间值。最初是因为考虑到图像分割的时候出现了很多噪音点，使用它可以起到平滑的作用。但是同时导致一定程度上速度变慢，所以后来总是嫌弃分割太慢的我有时会想要不要注释掉它……（最后暂时还是注释掉了）。

- `ToPyMSSeg(src, level, rate, sp, sr, maxit, distance)`：经过这个函数，之前数据结构里的金字塔也就全部形成了。

##### ② 树的构建

根据论文，下层要认上层与它overlap最大的一个ID为亲。而overlap=重叠区域的面积 /（颜色差+1）；加一是为了避免被除数为0，重叠区域的计算中，我的方法是每个下层同ID的像素，对它们的上层进行投票，按照一个下层像素给它在上层的对应点的ID投一票，最后计算票数，即为重叠部分的面积。

由于JS中没有链表这样的结构，因此我决定为每一层构造孩子列表，用孩子认亲的方法构造这个列表如下：

```pseudocode
add level 0 中的所有id → active set R
for A in R {
    初始化爸爸表;
    for 所有和A同level、同ID的像素{
        找出对应的上层的所有ID;
        在爸爸表中按ID分别计数;
        找出四邻域的邻居; 
        在该层邻居表中计数;
    }
    计算按ID平均颜色和该ID中的爸爸表计数，计算overlap;
    找到overlap最大的ID作为准爸爸;
    if 准爸爸没有孩子{
    	认亲;
    	把爸爸加入active set R;
    	把自己扔出active set R;
    }
    else{
        for 准爸爸的所有孩子{
        	if 有孩子和自己在邻居表中的计数>0{
                认亲;
                把自己扔出active set R;
        	}
        }
        if 没有认亲{
            推迟认亲;
        }
    }
}
```

<br>

### 4. 视觉模型

论文中视觉模型基于Eye Track数据，但是数据链接已经失效了。在没有该数据的情况下，我用绘制svg的方式，在画布上用鼠标点击产生一个圆圈，作为视觉中心fixation的替代，但是程序里其实是把它当作了一个正方形。

首先计算这个fixation与哪个区域关联。论文认为这个区域应该是符合：如果n的孩子有大于1/2的面积在fixation圆圈中，则n为这个区域；如果没有这样的n，那么这个区域是包含fixation中心点的叶节点。

因此我的运算方法也就是遍历金字塔中所有处在fixation范围内的点，记录它们的在fixation范围内的像素数，在这些记录的level和ID号中，依次判断是否符合上述条件，若是则直接返回。若没有这样的ID，则取level 0中fixation中心的点的ID。

论文认为当对比度contrast、频率f、关注时间t、以及是否是fixation关联区域的父子这些条件在符合一定不等式的时候才需要绘制细节。下面是我的计算方法：

- 是否是fixation关联区域的父子：提前找出所有fixation的亲友团，存在一个Array中，每次遍历亲友团判断当前区域是否是fixation关联区域的亲友。
- 关注时间t：由于情况和论文不同，这次不是真正的Eye Track数据，因此这里直接采用fixation圆圈的半径来表示t。

- 计算contrast：contrast是两区域之间颜色差异的加权和，颜色差 = 区域平均颜色差异 / 区域平均颜色和，而权重为公共边的边长。这就是之前要在构建孩子列表的时候输出邻居表的原因，这个相邻矩阵中的每个值都是两区域之间（四邻域中）相邻次数的累加，也就是公共边长的两倍。
- 计算频率f：原文中区域频率计算式为 f=1/2D，其中D是最小包围圆的直径，而我直接用了整个区域的面积来替代。

<br>



# 三、结果展示

### 1. 环境 

##### ① 编程语言

​	HTML + CSS + Javascript

##### ② 测试环境

​	Microsoft Edge 浏览器

##### ③ 用到的库 

- math.js：用到了它的一些数学运算以及矩阵结构。 
- jquery.js：利用它选择HTML文档的元素。
- d3.js：用它绘制折线图。

### 2. 界面

![界面](../assets/img/simpleimg/interface.png)

### 3. 运行结果

##### ① 边缘检测结果

低阈值4%，高阈值37%（强度百分比），短边阈值7%（长度百分比）的情况。

左图为原图，中图为原论文的增强Canny算法，而我只用了一般的Canny算法，如右图。

除了方法可能有差异，另外可能是图片清晰度的问题，由于原链接无法访问因此我所使用的原图为259px * 194px的大小，而原论文采用了1024px * 768px。

![原论文原图](../assets/img/simpleimg/origin1.JPG)![原论文边缘](../assets/img/simpleimg/origin3.JPG)![我的结果](../assets/img/simpleimg/edge1.JPG)



##### ② 图像颜色分割结果

窗口大小30%（相对于图片的短边），颜色容差15%（相对于100），步长为窗口大小的15%，迭代3次，3层金字塔的结果。

![图像分割结果](../assets/img/simpleimg/pyramid.jpg)

##### ③ 细节层测描绘结果（未进行平滑处理）

![图像分割结果](../assets/img/simpleimg/tmp.jpg)

##### ④ 界面链接

[https://sunnsta.github.io/mess/mess-simpleimg0.html](https://sunnsta.github.io/mess/mess-simpleimg0.html)

##### ⑤ 源码链接

JS文件：

[https://github.com/SunnSta/SunnSta.github.io/tree/master/assets/scripts/simpleimg](https://github.com/SunnSta/SunnSta.github.io/tree/master/assets/scripts/simpleimg)

- main.js：界面交互代码

- CannyEdge.js：边缘检测代码

- MeanshiftSegment.js：图像颜色分割、金字塔构造、孩子列表构造代码

- Render.js：视觉模型相关代码


<br>

  

# 四、感想

### 1. 关于Javascript

​	本次作业开始，由于心想着要做一个比较好看的界面，因此就这么排除了Qt和OpenCv的HighGUI，心情紧张地选择了使用静态网页。这学期其实还是刚刚接触Javascript，网页也不是很熟练，虽然Javascript的语法比较简明，但总是因为某些想当然的误用，比如Array用法和matrix的混淆，又比如JS函数调用的问题，导致出错，并为之调试了很久。作业期间也想过使用JS的效率问题，毕竟如果使用python和C++的话可以直接使用很完善的OpenCv库中的函数，他们做出的优化也是自己不可及的。最后我想，不管选择哪种语言总要有做出一点牺牲吧，我可能是一个并不想好好写代码而只是想做一个美工的人，那么多一种尝试也很有意思。

### 2. 关于看论文

​        这也是我第一次努力实现一篇论文。论文和想象中的不太一样，原本以为论文会把详细的算法写明，只要按部就班地实现就好了。没想到现实中的论文首先是有多处对别的论文的参考，需要自己另外体会参考论文的做法，看论文花的时间比预期要长很多，甚至到最后也没能够看懂参考文献中全部的推导。其次现实中的论文并没有花很多笔墨在描述算法上，细节都需要自己定夺，甚至连数据结构都没有提及。也可能是当初选择论文的时候没有仔细考虑，只觉得这个题材很吸引我。阅读之后才发现原文提供的两个过程资料的链接都失效了，也算是自己的失误。

### 3. 关于实现

​        最后确实做得并不够好也不够完整，自己的编程水平、调试以及寻找错误的能力还不够，由于这个原因作业写得太慢了，对数据结构也没有很好的安排。全局的几个结构中大多是嵌套数组，因此被下标困扰得十分头疼。比较满意的就是边缘检测的实现，感觉运行效率不算是很差而且边缘检测模块的交互上自我评价也应该说是还过得去。期间收获大概是初次使用了d3.js的折线图功能。而meanshift部分阅读了OpenCV的代码，学习到了一些优化和简化计算的方法，比如减少乘法和除法计算的技巧。这次作业遗憾很多，如果能有更多时间就好了，但是不管是该报告的排版还是最后的界面，我都会在之后争取改到令自己满意。

<br>

# 五、个人信息

姓名：吴萌

学号：3150102426
