# Adding (blending) two images using OpenCV


---

## **目标**

通过这篇教程，你会学到：

- 什么是线性叠加，以及它的用处;
- 如何使用cv::addWeighted对两个图像进行相加操作

## **原理**

>Note:
 以下解释出自Richard Szeliski的书[Computer Vision: Algorithms and Applications](http://szeliski.org/Book/)(《计算机视觉：算法与应用》)。
 
前面的教程中，我们已了解了一些像素相关的操作。一个非常有意思的双目（两个输入）运算符是线性叠加操作：

<center>g(x) = (1-α)f0(x) + αf1(x)</center>

其中α的取值为0至1，这一操作可产生两个图像或视频相互交合的效果，就像在某些幻灯片或电影里能看到的那样。（很酷，不是吗？）

## **源码**
可从[此](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/AddingImages/AddingImages.cpp)下载源码。

    #include "opencv2/imgcodecs.hpp"
    #include "opencv2/highgui.hpp"
    #include <iostream>
    using namespace cv;
    using namespace std;
    int main( void )
    {
       double alpha = 0.5; double beta; double input;
       Mat src1, src2, dst;
       cout << " Simple Linear Blender " << endl;
       cout << "-----------------------" << endl;
       cout << "* Enter alpha [0-1]: ";
       cin >> input;
       // 我们使用用户提供的α，其取值范围为0至1
       if( input >= 0 && input <= 1 )
         { alpha = input; }
       src1 = imread( "../data/LinuxLogo.jpg" );
       src2 = imread( "../data/WindowsLogo.jpg" );
       if( src1.empty() ) { cout << "Error loading src1" << endl; return -1; }
       if( src2.empty() ) { cout << "Error loading src2" << endl; return -1; }
       beta = ( 1.0 - alpha );
       addWeighted( src1, alpha, src2, beta, 0.0, dst);
       imshow( "Linear Blend", dst );
       waitKey(0);
       return 0;
    }
    
## **解释**

 1. 由于我们要实现：

    <center>g(x) = (1-α)f0(x) + αf1(x)</center>

   我们需要两张原始图像（f0(x)和f1(x)），所以我们先读取两张图片：

    `src1 = imread( "../data/LinuxLogo.jpg" );
     src2 = imread( "../data/WindowsLogo.jpg" );`

   **warning**
   
   由于我们想要叠加两张图片src1和src2，所以这两张图片必须要同样大小、同样类型。
   

 2. 现在要生成g(x)图像，对此OpenCV有个很好用的函数cv::addWeighted:
 

    `beta = ( 1.0 - alpha );
     addWeighted( src1, alpha, src2, beta, 0.0, dst);`
     
     由于cv::addWeighted产生:
     
     <center>dst = α·src1 + β·src2 + γ</center>
     
     在上面的代码中，γ就是值为0.0的参数。
     

 3. 创建窗口，显示图像并等待用户结束程序。
 

   ` imshow( "Linear Blend", dst );
     waitKey(0);`
     
## **结果**
 
结果如下：

<center>![Adding_Images_Tutorial_Result_Big](http://docs.opencv.org/master/Adding_Images_Tutorial_Result_Big.jpg)</center>