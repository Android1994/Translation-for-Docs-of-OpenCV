# Discrete Fourier Transform



---

## **目标**

我们将在本节教程探寻以下问题的答案：

- 傅里叶变换是什么？为什么要使用傅里叶变换？
- 如何在OpenCV中使用？
- 相关函数的用法，如：cv::copyMakeBorder() , cv::merge() , cv::dft() , cv::getOptimalDFTSize() , cv::log()以及cv::normalize() 。

## **源码**

你能从[这里](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/discrete_fourier_transform/discrete_fourier_transform.cpp)下载到源码，或从源码库以下位置找到`samples/cpp/tutorial_code/core/discrete_fourier_transform/discrete_fourier_transform.cpp`。

下面是使用函数cv::dft()的样例代码：

```
#include "opencv2/core.hpp"
#include "opencv2/imgproc.hpp"
#include "opencv2/imgcodecs.hpp"
#include "opencv2/highgui.hpp"
 
#include <iostream>
 
using namespace cv;
using namespace std;
 
static void help(char* progName)
{
  cout << endl
       <<  "This program demonstrated the use of the discrete Fourier transform (DFT). " << endl
       <<  "The dft of an image is taken and it's power spectrum is displayed."          << endl
       <<  "Usage:"                                                       << endl
       << progName << " [image_name -- default ../data/lena.jpg] "        << endl << endl;
}
 
int main(int argc, char ** argv)
{
    help(argv[0]);
 
    const char* filename = argc >=2 ? argv[1] : "../data/lena.jpg";
    
    Mat I = imread(filename, IMREAD_GRAYSCALE);
    if( I.empty())
      return -1;

    Mat padded;          //将图像扩大到最优尺寸
    int m = getOptimalDFTSize( I.rows );
    int n = getOptimalDFTSize( I.cols ); // on the border add zero values
    copyMakeBorder(I, padded, 0, m - I.rows, 0, n - I.cols, BORDER_CONSTANT, Scalar::all(0));
    
    Mat planes[] = {Mat_<float>(padded), Mat::zeros(padded.size(), CV_32F)};
    Mat complexI;
    merge(planes, 2, complexI);         // 把两个单通道的planes合成一个两通道的complexI
 
    dft(complexI, complexI);            // 结果会覆盖输入矩阵
   
    // 计算复数的模并转换到对数刻度
    // => log(1 + sqrt(Re(DFT(I))^2 + Im(DFT(I))^2))
    split(complexI, planes);                   // planes[0] = Re(DFT(I), planes[1] = Im(DFT(I))
    magnitude(planes[0], planes[1], planes[0]);// planes[0] = magnitude
    Mat magI = planes[0];
    
    magI += Scalar::all(1);                    // 转换到对数刻度
    log(magI, magI);
    
    // 假如图像的行数或列数是奇数的话，变换回原来的尺寸
    magI = magI(Rect(0, 0, magI.cols & -2, magI.rows & -2));
    
   // 重新安排一下坐标轴的象限，使得坐标原点处于图像中心
   int cx = magI.cols/2;
   int cy = magI.rows/2;
  
   Mat q0(magI, Rect(0, 0, cx, cy));   // 为每个象限创建一个ROI，这是左上方的
   Mat q1(magI, Rect(cx, 0, cx, cy));  // 右上方的
   Mat q2(magI, Rect(0, cy, cx, cy));  // 左下方的
   Mat q3(magI, Rect(cx, cy, cx, cy)); // 右下方的
   
   Mat tmp;                           // 对换两个象限 (左上和右下进行对换)
   q0.copyTo(tmp);
   q3.copyTo(q0);
   tmp.copyTo(q3);
   
   q1.copyTo(tmp);                    // 对换两个象限 (右上和左下进行对换)
   q2.copyTo(q1);
   tmp.copyTo(q2);
    
   normalize(magI, magI, 0, 1, NORM_MINMAX); // 将浮点数矩阵转换为可以显示的图像（浮点数的值在0-1之间）
    
   imshow("Input Image"       , I   );    // 显示结果
   imshow("spectrum magnitude", magI);
   waitKey();
    
   return 0;
}
```

## **解释**

傅里叶变换能够把一幅图像分解成正弦和余弦分量。也就是说能够把图像从空间域转换到频域。其思想是，任何函数都能近似地表示为无穷多个正弦和余弦函数之和。一个二维图像的傅里叶变换的数学形式如下：

。。。。。。（公式）

其中，f代表图像在空间域的像素值，而F则是在频率域。变换的结果是复数。想要显示结果的话，可以通过实数图和复数图或者通过幅度、相位图。但是，由于幅度图已经包含了我们所需的图像几何结构信息，所以，大部分图像处理算法只关心幅度图。尽管如此，假如你想在频域对图像做些修改然后再反变换回去，那么你需要保存所有的信息。

在这个例子中，我们会演示如何计算并显示图像经傅里叶变换之后的幅度图。由于数字图像是离散的，也就是说其值有一个取值范围。比如，在灰度图中，像素值的范围是0-255。因此，傅里叶变换也需相应地变为离散形式，及离散傅里叶变换(DFT)。当你想从几何的角度来描述一幅图像的结构时，DFT很有用。以下是具体步骤（输入为灰度图像I）：

1.**最优化图像尺寸**。DFT算法的性能取决于图像的大小。图像的大小是2、3、5的倍数时，该算法的处理速度会达到最快。因此，为了到达最好的算法性能，对图像进行填补边界使图像大小满足以上条件不失为一个好方法。函数cv::getOptimalDFTSize()能够返回最优的大小，而函数cv::copyMakeBorder()能扩充图像的边界：

```
Mat padded;                            //最优化尺寸后的图像
int m = getOptimalDFTSize( I.rows );
int n = getOptimalDFTSize( I.cols ); // 在边界填充值为0的像素点
copyMakeBorder(I, padded, 0, m - I.rows, 0, n - I.cols, BORDER_CONSTANT, Scalar::all(0));
```

附加上的像素点全部初始化为0。

2.**为复数值和实数值开辟空间**。傅里叶变换的结果是复数，这意味着，对每一个图像像素值的变换结果都是两个值（实部和虚部）。而且，频域的范围要比对应的空间域的范围大得多，所以一般把结果存储为float类型。最后，我们创建一个两通道的float型Mat来存储复数结果：

```
Mat planes[] = {Mat_<float>(padded), Mat::zeros(padded.size(), CV_32F)};
Mat complexI;
merge(planes, 2, complexI);         // 把两个单通道的planes合成一个两通道的complexI
```

3.**执行离散傅里叶变换**。这个操作支持in-place（即输入和输出的Mat一样）：

```
dft(complexI, complexI);            // 结果会覆盖输入矩阵
```

4.**计算复数的模值**。一个复数由实部（Re）和虚部（Imaginary-Im）组成,复数的模为：

。。。。。（公式）

OpenCV相关代码如下：

```
split(complexI, planes);                   // planes[0] = Re(DFT(I), planes[1] = Im(DFT(I))
magnitude(planes[0], planes[1], planes[0]);// planes[0] = magnitude
Mat magI = planes[0];
```

5.**转换为对数刻度**。傅里叶系数的范围很大，难以进行显示。我们有可能会得到一些难以观察的很小的值或很大的值。我们可以把很大的值显示为白色像素点，很小的值显示为黑色像素点。使用灰度图来对结果进行可视化，我们可以将线性刻度转换为对数刻度：

<center>M1 = log(1+M)</center>

相关的OpenCV代码：

```
magI += Scalar::all(1); // 转换为对数刻度
log(magI,magI);
```

6.**整理结果**。还记得我们在第一步的时候把图像扩大了吗？现在可以把这些添加的值删掉了。为了使得最终的结果更好看，我们还要重新安排一下坐标轴的象限，使得坐标原点处于图像中心：

```
magI = magI(Rect(0, 0, magI.cols & -2, magI.rows & -2));
int cx = magI.cols/2;
int cy = magI.rows/2;
Mat q0(magI, Rect(0, 0, cx, cy));   // 为每个象限创建一个ROI，这是左上方的
Mat q1(magI, Rect(cx, 0, cx, cy));  // 右上方的
Mat q2(magI, Rect(0, cy, cx, cy));  // 左下方的
Mat q3(magI, Rect(cx, cy, cx, cy)); // 右下方的
Mat tmp;                           // 对换两个象限 (左上和右下进行对换)
q0.copyTo(tmp);
q3.copyTo(q0);
tmp.copyTo(q3);
q1.copyTo(tmp);                    // 对换两个象限 (右上和左下进行对换)
q2.copyTo(q1);
tmp.copyTo(q2);
```

7.**归一化**。这一步仍是为了改善可视化的效果。虽然我们已经获得了复数的幅值，但是仍超过了图像的显示范围（0-1），可使用函数cv::normalize()将幅值缩放到0-1的范围。

```
normalize(magI, magI, 0, 1, NORM_MINMAX); // 将浮点数矩阵转换为可以显示的图像（浮点数的值在0-1之间）
```

## **结果**

具体的应用是看图像中的几何方向，比如，查看一些文本是否是水平的？观察文本你会发现有些字母的线条是水平的，有些字母的线条是垂直的，这两种文本片段的主要成分可以通过傅里叶变换观察到。我们可以使用一张水平的文字图片和经过旋转之后的图片进行比较。

水平的文字图片经傅里叶变换之后：

<center>![result_normal](http://docs.opencv.org/master/result_normal.jpg)</center>

经旋转之后的图片变换结果：

<center>![result_rotated](http://docs.opencv.org/master/result_rotated.jpg)</center>

可以看到，频域中的主要成分（明亮的部分）会随着图像中物体的几何旋转而旋转。通过这种方法，我们可以计算出图像的偏移，从而旋转图像进行校准。







