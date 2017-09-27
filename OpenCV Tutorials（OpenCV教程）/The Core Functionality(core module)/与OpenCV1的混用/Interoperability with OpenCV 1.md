# Interoperability with OpenCV 1

---

## **目标**

对于OpenCV的开发团队来说，保持更新很重要。我们希望这些改进能够让工作变得更容易的同时保持函数库的灵活性。新的c++接口便是为了实现这一目标而做出的改进。尽管如此，向后兼容的能力依然很重要。我们不希望看到之前基于老版本OpenCV编写的代码不能运行。因此，我们增加了一些功能来处理这种情况。下面，你将学到：

 - OpenCV2和之前版本的区别
 - 如何向图像中添加高斯噪音
 - 查找表是什么，以及如何使用它们

## **概述**

在进行转换之前，你需要首先了解新的数据结构：[Mat - The Basic Image Container](http://docs.opencv.org/master/d6/d6d/tutorial_mat_the_basic_image_container.html)，其代替了老的[CvMat](http://docs.opencv.org/master/d3/db2/structCvMat.html)和[IPLImage](http://docs.opencv.org/master/d6/d5b/structIplImage.html)。转换到新版本很容易，你只需记住少量的新事物。

OpenCV2的结构进行了重组，再也不会有把所有函数塞进一个单独的库的情况了。我们建立了许多模块。每个模块都包含了数据结构以及完成特定任务所需的函数。这样，假如你只需要用到OpenCV的某些子功能就不用再引入大量的库了。这意味着你只需添加要用到的头文件，比如：
```
#include <opencv2/core.hpp>
#include <opencv2/imgproc.hpp>
#include <opencv2/highgui.hpp>
```

所有OpenCV相关的东西都放在了cv这个命名空间里，从而避免了和其它库有命名冲突的问题。因此，你需要在使用OpenCV的东西之前加上关键字cv::，或者直接加上这条语句：
```
using namespace cv;  // 新的c++API也在这个名称空间里，直接这样导入就行.
```

这样就不用使用cv前缀了。新版c++的函数名都采用驼峰式命名规则。也就是说第一个字母是小写（除非是一个名称，如Canny），接下来的每个单词都以大写字母开头（比如copyMakeBorder）。

现在，你需要做的就是把你的应用程序和所有用到的模块进行链接，如果是windows系统并使用DLL系统的话，则需要添加库的目录。获得和Windows相关的更多信息可以访问[How to build applications with OpenCV inside the "Microsoft Visual Studio"](http://docs.opencv.org/master/d6/d8a/tutorial_windows_visual_studio_Opencv.html)，获得和Linux相关的更多信息可以访问[ Using OpenCV with Eclipse (plugin CDT)](http://docs.opencv.org/master/d7/d16/tutorial_linux_eclipse.html)。

如果想要转换Mat对象的话，可以使用[IplImage](http://docs.opencv.org/master/d6/d5b/structIplImage.html)和[CvMat](http://docs.opencv.org/master/d3/db2/structCvMat.html)操作符。现在已经不再使用指针这样的C风格接口了。在如今C++风格的接口中，大多数情况下使用的是Mat对象。这些对象可以通过很简单的命令转换成IplImage和CvMat类型。比如：

```
Mat I;
IplImage pI=I;
CvMat mI=I;
```

如果你想转换成指针，则稍微复杂一些。编译器不会自动识别你希望转换的类型，你需要明确地指明想要转换的类型。可以通过调用IplImage和CvMat操作符得到它们的指针。为了得到指针类型，我们需要使用&符：

```
Mat I;
IplImage* pI = &I.operator IplImage();
cvMat* mI = &I.operator CvMat();
```

对C风格接口最大的抱怨主要来自于需要自己管理内存空间。你需要判断何时释放指针是安全的，并确保释放指针时程序已结束运行，否则有可能造成内存泄露。为了解决该问题，OpenCV引入了一种叫智能指针的东西。这可以自动释放那些不再使用的对象。可以通过将指针声明为Ptr来实现：

```
Ptr<IplImage> piL = &I.operator IplImage();
```

将C风格的数据结构转换成Mat可以通过构造函数完成，比如：

```
Mat K(piL),L;
L=Mat(pI);
```

## **案例**
这里有一个混合使用C风格和C++风格接口的例子。为了更好地展现两种风格的不同：既有C和C++混合的例子，也有纯C++的例子。假如定义了*DEMO_MIXED_API_USE*，则是使用C和C++混合。这个程序的功能是分离彩色的平面图，然后分别做一些修改，最后再合并。

```
#include <iostream>
#include <opencv2/imgproc.hpp>
#include "opencv2/imgcodecs.hpp"
#include <opencv2/highgui.hpp>
using namespace cv;  // 新的C++接口在这个名称空间里
using namespace std;

//如果只想使用C++接口的话，就注释下面这句
#define DEMO_MIXED_API_USE
#ifdef DEMO_MIXED_API_USE
#  include <opencv2/highgui/highgui_c.h>
#  include <opencv2/imgcodecs/imgcodecs_c.h>
#endif
int main( int argc, char** argv )
{
    help(argv[0]);
    const char* imagename = argc > 1 ? argv[1] : "../data/lena.jpg";
#ifdef DEMO_MIXED_API_USE
    Ptr<IplImage> IplI(cvLoadImage(imagename));      //Ptr<T>是一个安全的引用计数指针类
    if(!IplI)
    {
        cerr << "Can not load image " <<  imagename << endl;
        return -1;
    }
    Mat I = cv::cvarrToMat(IplI); // Convert to the new style container. Only header created. Image not copied.
#else
    Mat I = imread(imagename);        // the newer cvLoadImage alternative, MATLAB-style function
    if( I.empty() )                   // same as if( !I.data )
    {
        cerr << "Can not load image " <<  imagename << endl;
        return -1;
    }
#endif
```

你可以发现使用新的结构体便没有指针的问题了，也可以使用之前的函数，并且返回的是*Mat*对象。

```
// convert image to YUV color space. The output image will be created automatically.
    Mat I_YUV;
    cvtColor(I, I_YUV, COLOR_BGR2YCrCb);
    vector<Mat> planes;    // Use the STL's vector structure to store multiple Mat objects
    split(I_YUV, planes);  // split the image into separate color planes (Y U V)
```

我们想先将图像从默认的BGR转换到YUV颜色空间，然后对图像的亮度成分做一些修改，再将图像分离成独立的平面图。程序的分离过程如下：在第一个例子中，处理每一个平面图时使用三种OpenCv中的主要扫描算法之一（C[]操作符，迭代器，单独的元素访问）。在第二部分，我们在图像中加入一些高斯噪声然后通过某种公式将分离的通道合并起来。
扫描的过程如下所示：

```
// Mat scanning
    // Method 1. process Y plane using an iterator
    MatIterator_<uchar> it = planes[0].begin<uchar>(), it_end = planes[0].end<uchar>();
    for(; it != it_end; ++it)
    {
        double v = *it * 1.7 + rand()%21 - 10;
        *it = saturate_cast<uchar>(v*v/255);
    }
    for( int y = 0; y < I_YUV.rows; y++ )
    {
        // Method 2. process the first chroma plane using pre-stored row pointer.
        uchar* Uptr = planes[1].ptr<uchar>(y);
        for( int x = 0; x < I_YUV.cols; x++ )
        {
            Uptr[x] = saturate_cast<uchar>((Uptr[x]-128)/2 + 128);
            // Method 3. process the second chroma plane using individual element access
            uchar& Vxy = planes[2].at<uchar>(y, x);
            Vxy =        saturate_cast<uchar>((Vxy-128)/2 + 128);
        }
    }
```

可以看到，我们使用了三种方法来遍历图像中的所有像素：使用迭代器，使用C指针以及直接访问单个元素。从之前的老函数名转换过来也很容易。只需移除cv前缀并且使用新的Mat数据结构。这里有一个使用加权叠加函数的例子：

```
Mat noisyI(I.size(), CV_8U);           // Create a matrix of the specified size and type
    // Fills the matrix with normally distributed random values (around number with deviation off).
    // There is also randu() for uniformly distributed random number generation
    randn(noisyI, Scalar::all(128), Scalar::all(20));
    // blur the noisyI a bit, kernel size is 3x3 and both sigma's are set to 0.5
    GaussianBlur(noisyI, noisyI, Size(3, 3), 0.5, 0.5);
    const double brightness_gain = 0;
    const double contrast_gain = 1.7;
#ifdef DEMO_MIXED_API_USE
    // To pass the new matrices to the functions that only work with IplImage or CvMat do:
    // step 1) Convert the headers (tip: data will not be copied).
    // step 2) call the function   (tip: to pass a pointer do not forget unary "&" to form pointers)
    IplImage cv_planes_0 = planes[0], cv_noise = noisyI;
    cvAddWeighted(&cv_planes_0, contrast_gain, &cv_noise, 1, -128 + brightness_gain, &cv_planes_0);
#else
    addWeighted(planes[0], contrast_gain, noisyI, 1, -128 + brightness_gain, planes[0]);
#endif
    const double color_scale = 0.5;
    // Mat::convertTo() replaces cvConvertScale.
    // One must explicitly specify the output matrix type (we keep it intact - planes[1].type())
    planes[1].convertTo(planes[1], planes[1].type(), color_scale, 128*(1-color_scale));
    // alternative form of cv::convertScale if we know the datatype at compile time ("uchar" here).
    // This expression will not create any temporary arrays ( so should be almost as fast as above)
    planes[2] = Mat_<uchar>(planes[2]*color_scale + 128*(1-color_scale));
    // Mat::mul replaces cvMul(). Again, no temporary arrays are created in case of simple expressions.
    planes[0] = planes[0].mul(planes[0], 1./255);
```

可以看到，存储平面图的变量是Mat类型。但是，从Mat转换到IplImage是很容易的，使用一个简单的声明操作符便可自动完成。

```
merge(planes, I_YUV);                // now merge the results back
    cvtColor(I_YUV, I, COLOR_YCrCb2BGR);  // and produce the output RGB image
    namedWindow("image with grain", WINDOW_AUTOSIZE);   // use this to create images
#ifdef DEMO_MIXED_API_USE
    // this is to demonstrate that I and IplI really share the data - the result of the above
    // processing is stored in I and thus in IplI too.
    cvShowImage("image with grain", IplI);
#else
    imshow("image with grain", I); // the new MATLAB style function show
#endif
```

新的*imshow*图形界面函数同时支持Mat和IplImage数据结构。编译并运行程序，如果将下面的第一张图像作为输入的话，你能得到如下输出：

<center>![outputInteropOpenCV1.jpg](http://docs.opencv.org/master/outputInteropOpenCV1.jpg)</center>

你可以在[YouTube](https://www.youtube.com/watch?v=qckm-zvo31w)上看到相关程序运行的实例，也可从[此处](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/interoperability_with_OpenCV_1/interoperability_with_OpenCV_1.cpp)下载源码。
