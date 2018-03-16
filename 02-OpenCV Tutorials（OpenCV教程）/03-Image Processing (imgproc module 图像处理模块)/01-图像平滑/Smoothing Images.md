# Smoothing Images（图像平滑）


---

## **目标**

在本篇教程当中，你将学习如何通过OpenCV的函数来利用各种不同的线性滤波器实现对图像的平滑，这些函数主要有：

- [blur()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#ga8c45db9afe636703801b0b2e440fce37)
- [GaussianBlur()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#gaabe8c836e97159a9193fb0b11ac52cf1)
- [medianBlur()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#ga564869aa33e58769b4469101aac458f9)
- [bilateralFilter()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#ga9d7064d478c95d60003cf839430737ed)

## **理论**

>注意： 下面的解释来源于Richard Szeliski所著的 [Computer Vision: Algorithms and Applications](http://szeliski.org/Book/) 以及 *LearningOpenCV*

- 平滑，同时也被称为模糊，是图像处理中经常使用的一种简单操作。
- 对图像进行平滑的目的有很多。在本篇教程中，我们专注于使用平滑操作来去除噪声（其他用法可以在其他教程中见到）。
- 为了进行平滑操作，我们会使用一个滤波器作用于图像上。最常用的滤波器是线性的，每个像素的输出值($g(i,j)$)取决于所有输入像素值的加权和($f(i+k,j+l)$):
$$
g(i,j) = \sum_{k,l}f(i+k,j+l)h(k,l)
$$
$h(k,l)$被称为*核*，就是滤波器的系数。  
可以将滤波器想象成是一个装满系数的窗口，滑过整幅图像。
- 滤波器的种类有很多，下面我们介绍一些最常用的：

### **归一化箱式滤波器**
- 这个滤波器是最简单的。每个像素的输出就是其邻域像素值的平均值（每个邻域像素贡献的权重相等）
- 核如下所示：
$$
K = \frac{1}{K_{width} \cdot K_{height}}
\begin{bmatrix}
1 & 1 & 1 & \cdots & 1\\ 
1 & 1 & 1 & \cdots & 1\\ 
\cdot & \cdot & \cdot & \cdots & 1\\ 
\cdot & \cdot & \cdot & \cdots & 1\\ 
1 & 1 & 1 & \cdots & 1
\end{bmatrix}
$$

### **高斯滤波器**
- 也许是最有用的滤波器（尽管不是最快的）。高斯滤波使用一个高斯核对输入数组的每一个元素作卷积并把它们加起来作为输出值。
- 为了解释得更清楚，还记得1D的高斯核像什么样吗？  
<center>![Smoothing_Tutorial_theory_gaussian_0](https://docs.opencv.org/master/Smoothing_Tutorial_theory_gaussian_0.jpg)</center>
将图像想象成是一维的话，你会发现落在中间的像素有最大的权重。周围的像素离中心越远，权重越小。

> 注意：  
2维的高斯函数可以写成：
$$
G_{0}(x,y) = Ae^{\frac{-(x-\mu_x)^2}{2\sigma_x^2} + \frac{-(y-\mu_y)^2}{2\sigma_y^2}}
$$  
其中$\mu$是均值（峰值），$\sigma^2$是方差（分别对应变量$x$和$y$）

### **中值滤波器**

中值滤波器遍历信号（此处为图像）的每一个元素，并将像素值替换为其周围邻域像素值的中值（邻域像素分布在一个正方形的邻域内）。

### **双边滤波器**

- 至此，我们已经对一些主要目的是平滑一张图像的滤波器进行了解释。但是，有时候滤波器不仅会平滑噪音，同时也会将边缘也平滑掉。为了避免这种情况（至少在一定程度上避免），我们使用双边滤波器。
- 和高斯滤波器类似，双边滤波器也着重考虑分配给邻域像素的权重。这些权重包括两种成分，一种是和高斯滤波一样的权重，一种则考虑邻域像素值和中心像素值之间的差异程度。
- 更详细的解释可以查看[此链接](http://homepages.inf.ed.ac.uk/rbf/CVonline/LOCAL_COPIES/MANDUCHI1/Bilateral_Filtering.html)。

## **代码**
- 这个程序的功能是什么？
    - 读取一张图像
    - 分别使用四种不同的滤波器（在理论部分解释的）并依次显示滤波结果
- 代码下载：点击[这里](https://raw.githubusercontent.com/opencv/opencv/master/samples/cpp/tutorial_code/ImgProc/Smoothing/Smoothing.cpp)
- 代码展示：
```
#include <iostream>
#include "opencv2/imgproc.hpp"
#include "opencv2/imgcodecs.hpp"
#include "opencv2/highgui.hpp"
using namespace std;
using namespace cv;
int DELAY_CAPTION = 1500;
int DELAY_BLUR = 100;
int MAX_KERNEL_LENGTH = 31;
Mat src; Mat dst;
char window_name[] = "Smoothing Demo";
int display_caption( const char* caption );
int display_dst( int delay );
int main( int argc, char ** argv )
{
    namedWindow( window_name, WINDOW_AUTOSIZE );
    const char* filename = argc >=2 ? argv[1] : "../data/lena.jpg";
    src = imread( filename, IMREAD_COLOR );
    if(src.empty()){
        printf(" Error opening image\n");
        printf(" Usage: ./Smoothing [image_name -- default ../data/lena.jpg] \n");
        return -1;
    }
    if( display_caption( "Original Image" ) != 0 ) { return 0; }
    dst = src.clone();
    if( display_dst( DELAY_CAPTION ) != 0 ) { return 0; }
    if( display_caption( "Homogeneous Blur" ) != 0 ) { return 0; }
    for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
    { blur( src, dst, Size( i, i ), Point(-1,-1) );
        if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
    if( display_caption( "Gaussian Blur" ) != 0 ) { return 0; }
    for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
    { GaussianBlur( src, dst, Size( i, i ), 0, 0 );
        if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
    if( display_caption( "Median Blur" ) != 0 ) { return 0; }
    for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
    { medianBlur ( src, dst, i );
        if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
    if( display_caption( "Bilateral Blur" ) != 0 ) { return 0; }
    for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
    { bilateralFilter ( src, dst, i, i*2, i/2 );
        if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
    display_caption( "Done!" );
    return 0;
}
int display_caption( const char* caption )
{
    dst = Mat::zeros( src.size(), src.type() );
    putText( dst, caption,
             Point( src.cols/4, src.rows/2),
             FONT_HERSHEY_COMPLEX, 1, Scalar(255, 255, 255) );
    return display_dst(DELAY_CAPTION);
}
int display_dst( int delay )
{
    imshow( window_name, dst );
    int c = waitKey ( delay );
    if( c >= 0 ) { return -1; }
    return 0;
}
```

## **代码解释**
让我们看看上面OpenCV函数中涉及到平滑的部分。

### **归一化箱式滤波器：**
- OpenCV提供了[blur()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#ga8c45db9afe636703801b0b2e440fce37)函数来使用该滤波器执行平滑操作。我们需要指定4个参数（更多细节请查阅资料）：
    - *src*:原图像
    - *dst*:目标图像
    - *Size(w,h)*:指定了核的大小（w像素宽，h像素高）
    - *Point(-1,-1)*:说明锚点（待计算的像素）相较于邻域像素的位置。假如是负值，则核的中心位置即是锚点。
```
for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
    { blur( src, dst, Size( i, i ), Point(-1,-1) );
        if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
```

### **高斯滤波器**
- 通过函数[GaussianBlur()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#gaabe8c836e97159a9193fb0b11ac52cf1)实现：这里使用了4个参数（更多细节，请查看引用）：
    - *src*:原图像
    - *dst*:目标图像
    - *Size(w,h)*:指定了核的大小。w和h必须是奇数，否则核的大小将使用$\sigma_x$和$\sigma_y$来计算得到。
    - $\sigma_x$:x变量的标准差。输入0代表使用前面指定的核大小参数来计算$\sigma_x$。
    - $\sigma_y$:y变量的标准差。输入0代表使用前面指定的核大小参数来计算$\sigma_y$。
```
for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
{ GaussianBlur( src, dst, Size( i, i ), 0, 0 );
    if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
```

### **中值滤波器**
- 该滤波器功能由函数[medianBlur](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#ga564869aa33e58769b4469101aac458f9)提供：使用三个参数：
    - *src*:原图像
    - *dst*:目标图像，必须和原图像类型一致
    - *i*:指定了核的大小（由于我们使用正方形的邻域，一个参数就够了）。必须是奇数。
```
 for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
{ medianBlur ( src, dst, i );
    if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
```
 
 ### **双边滤波器**
- 功能由OpenCV函数[bilateralFilter()](https://docs.opencv.org/master/d4/d86/group__imgproc__filter.html#ga9d7064d478c95d60003cf839430737ed)提供，使用5个参数：
    - *src*:原图像
    - *dst*:目标图像
    - *d*:每个像素邻域的直径。
    - $\sigma_{Color}$:像素颜色空间的标准差。
    - $\sigma_{Space}$:像素坐标空间的标准差。
```
for ( int i = 1; i < MAX_KERNEL_LENGTH; i = i + 2 )
{ bilateralFilter ( src, dst, i, i*2, i/2 );
    if( display_dst( DELAY_BLUR ) != 0 ) { return 0; } }
```

## **结果**
- 程序打开一张图片（这里使用[lena.jpg](https://raw.githubusercontent.com/opencv/opencv/master/samples/data/lena.jpg)）并显示4种滤波器的作用效果
- 此处截取了使用*medianBlur（中值滤波器）*平滑图像的结果：
<center>![Smoothing_Tutorial_Result_Median_Filter](https://docs.opencv.org/master/Smoothing_Tutorial_Result_Median_Filter.jpg)</center>