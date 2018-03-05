# Changing the contrast and brightness of an image（改变图像的亮度和对比度）

---

## **目标**

你能从这篇教程中学到：

- 访问像素值
- 初始化0矩阵
- 学习cv_saturate_cast的用处和用法
- 学习如何变换像素
- 改善图像亮度

## **原理**

> Note:
  以下解释出自Richard Szeliski的书[Computer Vision: Algorithms and Applications](http://szeliski.org/Book/)（计算机视觉：算法和应用）
  
### **图像处理**

- 通常，图像处理函数会有一个或多个输入图像，然后产生一个输出图像
- 图像的变换可被看作是：
  - 基于像素点的操作（像素变换）
  - 邻域操作（基于区域）

### **像素变换**     

- 此种变换，每个像素的输出值只取决于对应像素的输入值（也有可能和一些全局的信息或参数有关）。
- 这种操作的例子有改变亮度和对比度以及颜色的修正、变换等。

### **改变图像亮度和对比度**

- 两个常用的操作是乘法和加上一个常数：
<center>g(x)=αf(x)+β</center>

- 参数α>0并且一般称参数β为增益或偏置；这些参数分别控制图像的亮度和对比度
- 你可以把f(x)看作原图像的像素，g(x)看作输出图像的像素，这样，我们可以把公式记作：
<center>g(i,j)=αf(i,j)+β</center>
其中i和j表示像素处于第i行第j列。

## **代码**

- 下面的代码实现了g(i,j)=αf(i,j)+β：
    
    `#include "opencv2/imgcodecs.hpp"`
    `#include "opencv2/highgui.hpp"`
    `#include <iostream>`
    `using namespace std;
     using namespace cv;
    int main( int, char** argv )
    {
        double alpha = 1.0; //控制对比度
        int beta = 0;       //控制亮度
        Mat image = imread( argv[1] );
        Mat new_image = Mat::zeros( image.size(), image.type() );
        cout << " Basic Linear Transforms " << endl;
        cout << "-------------------------" << endl;
        cout << "* Enter the alpha value [1.0-3.0]: "; cin >> alpha;
        cout << "* Enter the beta value [0-100]: ";    cin >> beta;
        for( int y = 0; y < image.rows; y++ ) {
            for( int x = 0; x < image.cols; x++ ) {
                for( int c = 0; c < 3; c++ ) {
                    new_image.at<Vec3b>(y,x)[c] =
                      saturate_cast<uchar>( alpha*( image.at<Vec3b>(y,x)[c] ) + beta );
                }
            }
        }
        namedWindow("Original Image", WINDOW_AUTOSIZE);
        namedWindow("New Image", WINDOW_AUTOSIZE);
        imshow("Original Image", image);
        imshow("New Image", new_image);
        waitKey();
        return 0;
    }`

## **解释**

 1. 首先，我们创建两个变量来存储用户输入的α和β值：
    `double alpha = 1.0; //控制对比度
     int beta = 0;       //控制亮度`
 
 
 2. 使用cv::imread读取图像：
    `Mat image = imread( argv[1] );`

 3. 我们需要一个新的Mat对象来存储变换后的图像，我们希望新的Mat对象有以下特征：
       
      - 初始化像素为0；
      - 和原图像有相同的大小、类型；
        `Mat new_image = Mat::zeros( image.size(), image.type() );`
        我们可以看到Mat::zeros基于image.size()和image.type()返回了matlab风格的零元素初始化。

 4. 现在，访问每个像素实现g(i,j)=αf(i,j)+β这一操作，由于我们是对BGR图像进行操作，所以我们需要分别访问每个像素的三个通道值，下面是这部分代码：
    `for( int y = 0; y < image.rows; y++ ) {
        for( int x = 0; x < image.cols; x++ ) {
            for( int c = 0; c < 3; c++ ) {
                new_image.at<Vec3b>(y,x)[c] =
                  saturate_cast<uchar>( alpha*( image.at<Vec3b>(y,x)[c] ) + beta );
            }
        }
    }`

    有以下几点需注意：
    - 我们使用了image.at<Vec3b>(y,x)[c]这一语句来实现访问每个像素，y是行号，x是列号，c是R、G、B（0、1、2）。
    - 由于操作αf(i,j)+β有可能输出超过取值范围的值，我们使用了cv::saturate_cast来保证输出值有效。
    

 5. 最后，我们创建窗口用来显示结果：
    `namedWindow("Original Image", WINDOW_AUTOSIZE);
    namedWindow("New Image", WINDOW_AUTOSIZE);
    imshow("Original Image", image);
    imshow("New Image", new_image);
    waitKey();`

>Note:
      我们可以使用下面的命令来取代for循环：
      `image.convertTo(new_image, -1, alpha, beta);`
      其中，cv::Mat::convertTo能高效地实现 new_image = a*image + beta这一操作，上面的例子只是为了更好地演示如何访问每个像素。不管怎样，这两种方法都能达到相同的效果，但使用convertTo方法会更高效。
      
## **结果**

 - 运行代码，设置α为2.2，β为50
   `$ ./BasicLinearTransforms lena.jpg`
   `Basic Linear Transforms`
   `-------------------------`
   `* Enter the alpha value [1.0-3.0]: 2.2`
   `* Enter the beta value [0-100]: 50`

 - 我们得到：
   <center>![Basic_Linear_Transform_Tutorial_Result_big](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_Result_big.jpg)</center>

## **实例**

在本段中，我们将运用之前所学的调整图像的亮度和对比度来修正一幅曝光不足的图片。我们还会使用另一种叫做gamma校正的技术来修正图像的亮度。

### **调整亮度和对比度**

增加（或降低）β值会使得每个像素加上（或减去）一个常数。若计算结果超过[0;255]范围，将会进行饱和处理（即若值超过255（或小于0），则将值转化为255（或0））。

<center>![Basic_Linear_Transform_Tutorial_hist_beta](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_hist_beta.png)</center>

<center>**图中，颜色浅的部分代表原图像的灰度直方图，颜色深的部分代表β设为80的校正后图像的灰度直方图**</center>

这个直方图统计的是不同颜色等级的像素数量（横轴为颜色等级，竖轴为像素数量）。一幅比较暗的图片里面大多像素的颜色值都比较小，由此，峰值处于直方图的左边。当加上一个常量偏置后，由于是对所有像素都加上这一常量，所以直方图整体会往右移动。

参数α会影响直方图中条柱的分布。假如设置α小于1，直方图的条柱会更紧密，图像的对比度会更小。

<center>![Basic_Linear_Transform_Tutorial_hist_alpha](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_hist_alpha.png)</center>

<center>**图中，颜色浅的部分代表原图像的灰度直方图，颜色深的部分代表α设为<1进行校正后图像的灰度直方图**</center>

注意，这些直方图是使用色彩平衡校正软件里的工具得到的。（下一句没看懂：The brightness tool should be identical to the β bias parameters but the contrast tool seems to differ to the α gain where the output range seems to be centered with Gimp (as you can notice in the previous histogram).）

使用β偏置可以提高图像亮度，但同时会造成对比度下降从而导致图像有些模糊不清。使用α增益可以改善这一情况，但由于计算结果可能溢出，这会导致原始图像明亮区域一些细节信息的丢失。

### **Gamma校正**

[Gamma校正](https://en.wikipedia.org/wiki/Gamma_correction)可用于校正图像的亮度，它是一种非线性的变换：

<center>**O = （(I/255)^γ） * 255**</center>

由于变换是非线性的，所以每个像素的变换都是不一样的，具体取决于每个像素的原始值。

<center>![Basic_Linear_Transform_Tutorial_gamma](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_gamma.png)</center>

<center>**不同Gamma值、不同输入值对应的变化趋势**</center>

当γ值小于1时，图像灰暗部分会变得更加明亮，直方图也会往右移；当γ值大于1时则相反。

### **修正一幅曝光不足的图像**

对下面的图像进行修正：α设为1.3，β设为40:

<center>![Basic_Linear_Transform_Tutorial_linear_transform_correction](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_linear_transform_correction.jpg)</center>

<center>**由Visem拍摄[CC BY-SA 3.0],取自维基共享资源**</center>

可以看出，图像总体亮度得到改善，但是很明显，天空云彩部分由于计算结果溢出变得过于明亮（摄影术中的[高光区域裁剪](https://en.wikipedia.org/wiki/Clipping_(photography))）。

对下面的图像采用Gamma修正，γ设为0.4:

<center>![Basic_Linear_Transform_Tutorial_gamma_correction](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_gamma_correction.jpg)</center>

<center>**由Visem拍摄[CC BY-SA 3.0],取自维基共享资源**</center>

由于Gamma校正是一种非线性的变换，所以没有之前的数值溢出问题。

<center>![Basic_Linear_Transform_Tutorial_histogram_compare](http://docs.opencv.org/master/Basic_Linear_Transform_Tutorial_histogram_compare.png)</center>

<center>**左：α、β校正后的直方图；中：原图的直方图；右：Gamma校正后的直方图**</center>

上面这张图比较了三幅图的直方图（y-轴的取值范围不一样）。可以看到，原图中，大部分像素聚集在直方图的左边。在经过α、β校正后，直方图整体往右移动了并且由于数值溢出的问题，在x-轴255处（最右边）出现了很高的峰值。而在Gamma校正后，虽然直方图也是有往右移的趋势，但是原本值较低的区域移动得更多，原本值本来就比较高的区域移动得较少。

在这篇教程中，你已经了解了调整图像亮度和对比度的两种简单方法。它们是比较基础的技术，并不是用来替代位图图像编辑器的！

### **代码**

你可以在[此处](https://github.com/opencv/opencv/blob/master/samples/cpp/tutorial_code/ImgProc/changing_contrast_brightness_image/changing_contrast_brightness_image.cpp)找到本篇教程的代码，Gamma校正的代码如下：

    Mat lookUpTable(1, 256, CV_8U);
    uchar* p = lookUpTable.ptr();
    for( int i = 0; i < 256; ++i)
        p[i] = saturate_cast<uchar>(pow(i / 255.0, gamma_) * 255.0);
    Mat res = img.clone();
    LUT(img, lookUpTable, res);
    
这里使用了查找表来改善算法性能。

### **其它资料**

- [Gamma correction in graphics rendering（图形渲染中的Gamma校正）](https://learnopengl.com/#!Advanced-Lighting/Gamma-Correction)
- [Gamma correction and images displayed on CRT monitors（CRT监视器上的Gamma校正和图像显示）](http://www.graphics.cornell.edu/~westin/gamma/gamma.html)
- [Digital exposure techniques(数码曝光技术)](http://www.cambridgeincolour.com/tutorials/digital-exposure-techniques.htm)

    





