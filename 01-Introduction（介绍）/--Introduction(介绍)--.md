# **Introduction(介绍)**
---

OpenCV[(开源计算机视觉库)](http://opencv.org)是一个遵从BSD协议的开源库，其中包含几百种计算机视觉相关算法。这个文档是基于OpenCV 2.X版本的API，和1.X版本不同的是它是基于C++的而1.X版本是基于C的。

OpenCV的整体结构是模块化的，也就是说其中有一些共享或静态库。主要有以下几个模块：

    - 核心功能模块(Core functionality) --这一模块比较紧凑，主要包括一些基本数据结构的定义，比如密度多维数组Mat，以及被其它模块调用的基本函数。
    - 图像处理模块(image processing) --这个模块主要有线性和非线性的图像滤波函数，图像的几何变换函数（大小变换，仿射，透视，映射），颜色空间转换函数以及直方图相关函数等。
    - 视频模块(video) --视频分析模块，主要包括运动估计、背景消除、目标跟踪等算法。
    - calib3d模块 --主要和立体视觉有关，包括基本的多视图几何算法、相机校准、目标姿态估计、立体匹配算法以及三维重构相关的方法。
    - feature2d模块 --主要和特征检测、提取，计算特征描述子以及利用描述子进行匹配相关。
    - objdetect模块 --检测目标物体或预先定义的目标实例（如人脸、眼睛、汽车等）。
    - highgui模块 --提供简单易用的接口用以实现一些简单的界面功能。
    - Video I/O模块 --提供视频捕获和编解码相关功能。
    - gpu模块 --提供gpu加速相关算法。
    - ...还有一些其它的辅助模块，比如FLANN(快速最近邻算法)，Google test wrappers，Python Buildings等。
    
这个文档的其它章节会详细介绍每个模块的功能，首先，让我们熟悉一下API的概念。

## **API**

### CV Namespace（cv 命名空间）
OpenCV里所有的类以及函数都放在cv这个命名空间里，所以想要调用某个函数需要加上"cv::"标识符或"using namespace cv;"这条语句，如下：

    #include "opencv2/core.hpp"
    ...
    cv::Mat H = cv::findHomography(points1, points2, CV_RANSAC, 5);
    ...
或者：

    #include "opencv2/core.hpp"
    using namespace cv;
    ...
    Mat H = findHomography(points1, points2, CV_RANSAC, 5);
    ...

目前或未来的OpenCV库里可能有和标准模板库(STL)或其它的一些库产生冲突的外部名称，此时，应使用明确的命名空间标识符：

    Mat a(100, 100, CV_32F);
    randu(a, Scalar::all(1), Scalar::all(std::rand()));
    cv::log(a, a);
    a /= std::log(2.);
    
### Automatic Memory Management（自动内存管理）
OpenCV能够自动管理内存。

首先，std::vector、Mat以及其它的数据结构都有析构函数，用于在需要的时候释放内存缓冲区。这意味着析构函数并不总是会释放内存，还需要考虑是否有数据共享的情况。就Mat数据类型来说，它有一个引用计数器和它所占的内存相关联，当且仅当计数器为0的时候，即没有其它结构引用这块内存的数据的时候，这块内存才会被释放。同理，拷贝一个Mat实例时数据并没有真正意义上的被拷贝，只是被引用，这时，这块数据的引用计数器会增加。当然，也有函数能实现真正意义上的拷贝Mat数据，即Mat::clone方法，如：

    // 创建一个8Mb的大矩阵
    Mat A(1000, 1000, CV_64F);
    // 对同一个矩阵创建另一个header B;
    // 下面这个操作是立即完成的，和矩阵大小无关.
    Mat B = A;
    // 创建另一个header C指向A的第三行; 同样，此时没有数据没被真正拷贝
    Mat C = B.row(3);
    // 下面，创建一份真正的独立的拷贝D
    Mat D = B.clone();
    // 下面，把B的第5行拷给C,相当于把A的第5行数据拷给A的第3行
    //（B和A指向同一块数据，C指向这块数据的第三行）
    B.row(5).copyTo(C);
    // 让A和D共享一块数据，此时之前被修改的A仍被B和C引用
    A = D;
    // 让B变成空矩阵(也就是让B不引用任何内存数据),但被修改的A仍被C引用,
    // 尽管C只是指向被修改的A的一行数据
    B.release();
    //最后，让C指向一份真正的C的拷贝. 此时，大矩阵数据将被释放,
    //因为它没有再被引用了
    C = C.clone();
    
可以看出，Mat和其它基本的数据结构使用起来很简单，那么对于更高级的类或没有考虑自动内存管理的用户自定义类型又该如何处理呢？针对这种情况，OpenCV提供了和C++11中的std:shared_ptr相似的Ptr模板类，所以，比起用普通的指针：

    T* ptr = new T(...);
更推荐您使用：

    Ptr<T> ptr(new T(...));
或：

    Ptr<T> ptr = makePtr<T>(...);
Ptr<T>将指针封装进了一个T的实例，而且创建了一个和指针相关联的引用计数器。详细情况请看Ptr的定义。

### Automatic Allocation of the Output Data(输出数据的自动分配)
OpenCV能够自动释放内存，同样也能为输出数据自动分配内存。加入某个函数有一个或多个输入数组（cv::Mat实例）以及一些输出数组，那么OpenCV会自动地分配空间给输出数组。一般输出数组的类型和大小由输入数组确定，如果有需要，函数可根据额外参数来确定输出数组的属性，如：

    #include "opencv2/imgproc.hpp"
    #include "opencv2/highgui.hpp"
    using namespace cv;
    int main(int, char**)
    {
        VideoCapture cap(0);
        if(!cap.isOpened()) return -1;
        Mat frame, edges;
        namedWindow("edges",1);
        for(;;)
        {
            cap >> frame;
            cvtColor(frame, edges, COLOR_BGR2GRAY);
            GaussianBlur(edges, edges, Size(7,7), 1.5, 1.5);
            Canny(edges, edges, 0, 30, 3);
            imshow("edges", edges);
            if(waitKey(30) >= 0) break;
        }
        return 0;
    }
其中，数组"frame"通过">>"操作符自动分配空间，因为视频每一帧的分辨率和位-深度已经通过视频模块的处理事先得知。同样，数组"edges"的空间也由函数cvtColor自动分配，它的类型、位-深度和输入数组一样。"edges"的通道数是1，因为COLOR_BGR2GRAY代表从彩色图像转换为灰度图。注意，这里"frame"和"edges"只在函数第一次执行时被分配一次，因为视频每一帧都有同样的分辨率。如果你换了不同分辨率的视频，这两个数组会被重新分配空间。

这一技术实现的关键是Mat::create方法，它要获取数组的大小和类型。假如数组已经事先指定了大小和类型，那么这个函数什么也不做。否则，它会释放先前分配的数据，假如有的话（这一过程包括引用计数器减少以及与0作比较），然后根据所需的大小分配新的内存空间。大多数的函数都会为每一个输出数组调用Mat::create方法，这就实现了输出数据的自动分配。

有一些需要注意的例外，如cv::mixChannels，cv::RNG::fill以及其它少部分函数。它们不会为输出数组分配空间，调用这些方法前，你需要事先分配空间。

### Saturation Arithmetics(饱和计算)
作为计算机视觉库，OpenCV需要处理大量的图像像素，因此对像素的编码比较紧凑，一般每个通道大小为8bit或16bit，取值范围有限。有时候，对有些操作比如颜色空间转换、亮度/对比度调整、锐化、复杂插值（双三次插值、Lanczos）来说，它们有可能计算出超过取值范围的结果。这时，如果你只保留结果的低8(16)bit数据，那么就会影响接下来的图像分析工作。为了解决这一问题，OpenCV用到了饱和计算。例如，为了将操作结果r保存到8-bit编码的图像里，OpenCV会在0...255的范围内找到离r最近的值作为替代：

<center>**I(x,y)=min(max(round(r),0),255)**</center>

这个规则同样适用于有符号8-bit型、有符号16bit型以及其它无符号类型，并且在OpenCV库中应用得很广泛。在C++的代码里，使用saturation_cast<>函数可实现这一功能，这很像C++的强制类型转换操作，上例代码如下：

    I.at<uchar>(y, x) = saturate_cast<uchar>(r);
其中“uchar”是OpenCV中的8-bit无符号整型。在针对SIMD优化过的代码里，类似的SSE2指令有paddusb、packuswb等，它们实现的功能是相似的。

> Note： 
saturation操作不适用于类型为32-bit整型的数据

### Fixed Pixel Types. Limited Use of Templates(固定的像素类型。适量使用模板)

模板是C++的一种强大特性，能够保证实现功能强大、有效、安全的数据结构和算法。但是大量使用模板会导致编译时间和代码块大小的急剧增加，而且，如果只使用模板的话，会很难区分接口部分和功能实现部分。一些基本算法使用模板效果还不错，但对于一个算法常常就有几千行代码的计算机视觉库来说可就不怎么好了。由于这一原因以及为了其它没有或只有有限模板功能的语言（Python、Java、Matlab等）能够更方便地使用OpenCV来进行开发，如今的OpenCV更多的是建立在多态和运行时调度模板之上的。在运行时调度会变得很慢（如访问像素操作）、不可行（通用的Ptr<>方法实现）或者不方便（saturation_cast<>()）的地方，现在都采用小的模板类和函数。总之，现在的OpenCV版本对模板的使用更加小心翼翼。

因此，OpenCV能够操作的原始数据类型比较固定、有限。就是说，数组元素应该是以下类型之一：

 - 8-bit无符号整型(uchar)
 - 8-bit有符号整型(schar)
 - 16-bit无符号整型(ushort)
 - 16-bit有符号整型(short)
 - 32-bit有符号整型(int)
 - 32-bit单精度浮点数(float)
 - 64-bit双精度浮点数(double)
 - 含有若干元素的元组，其中的元素的类型都一样（以上类型之一）。数组元素是这样的元组的数组被称为多通道数组，和元素是标量值的单通道数组对应。最大的通道数被定义为常量“CV_CN_MAX”，现在的值是512。

以上基本类型可用下面的枚举来表示：

    enum { CV_8U=0, CV_8S=1, CV_16U=2, CV_16S=3, CV_32S=4, CV_32F=5, CV_64F=6 }
    
多通道(n通道)类型可用下面的方法表示：

 - 常量CV_8UC1...CV_64FC4（通道数从1到4）
 - 当通道数超过4或在编译时未知时，可用宏CV_8UC(n) ... CV_64FC(n) 或 CV_MAKETYPE(CV_8U, n) ... CV_MAKETYPE(CV_64F, n)

> Note: 
CV_32FC1 == CV_32F, CV_32FC2 == CV_32FC(2) == CV_MAKETYPE(CV_32F, 2), 并且 CV_MAKETYPE(depth, n) == ((depth&7) + ((n-1)<<3)`。（此处有疑问）

例子：

    Mat mtx(3, 3, CV_32F); // 创建3x3大小的单精度矩阵
    Mat cmtx(10, 1, CV_64FC2); // 创建10x1大小的2通道矩阵
    Mat img(Size(1920, 1080), CV_8UC3); // 创建1920列，1080行的三通道（彩色）图像
    Mat grayscale(image.size(), CV_MAKETYPE(image.depth(), 1)); // 创建和image同大小、同通道类型的单通道图像
    
OpenCV不能创建或处理更复杂的数据类型。或者说，每个函数只能处理所有可能的数组类型的子集。通常来说，算法越复杂，支持的类型越少。下面是这种限制的典型例子：

 - 人脸识别算法只能处理8-bit的灰度图或彩色图
 - 线性代数函数和大多数的机器学习算法只能处理浮点数组
 - 基本函数，比如cv::add支持所有类型
 - 颜色空间转换函数支持8-bit无符号类型、16-bit无符号类型和32-bit浮点数类型。

每个函数所支持的数据类型主要根据实际需求来决定的，未来也可根据用户需求进行扩展。

### InputArray 和 OutputArray
很多OpenCV的函数都会处理2维或多维的密集数值数组。通常，这些函数会使用cppMat作为参数，但有时候使用std::vector<>（比如点集）或Mat<>（比如3x3的单应矩阵）会更方便。为了避免API的重复，OpenCV使用了特殊的“代理”类，用于传递只读数组作为函数输入。派生自InputArray类的OutputArray类用来声明函数的输出数组。一般情况下，你不需要关心这些中间类型（并且不需要明确声明这些类型的变量）--它们能够自动工作（转换）。你可以认为你总是能用Mat、std::vector<>、Matx<>、Vec<> 或者 Scalar这些类型来代替InputArray/OutputArray。如果某个函数有可选的输入或输出数组，而你没有或不需要的话，可传递cv::noArray()作为参数。

### Error Handling(错误处理)
OpenCV使用异常来提示严重的错误。当输入数据格式正确，取值范围合理但是函数因为某种原因不能正常返回结果时（比如优化算法无法收敛），它会返回一个特殊的错误码（通常是布尔值）。

异常也可以是cv::Exception类的实例或派生类。同样的，cv::Exception派生自std::exception。所以，在使用了其它C++标准库的代码中，它也能很好地运行。

一般，既可以使用宏CV_Error(errcode,description)或类似printf的变种CV_Error_(errcode, printf-spec, (printf-args))进行异常抛出，也可使用宏CV_Assert(condition)检测是否满足条件，如果不满足则抛出异常。对于注重性能的代码，可使用只在Debug配置下可用的CV_DbgAssert(condition)。通过自动内存管理，所有的内存临时缓冲区在程序发生错误时都会被释放。如果需要的话，你可以使用try语句来捕获异常：

    try
    {
        ... // call OpenCV
    }
    catch( cv::Exception& e )
    {
        const char* err_msg = e.what();
        std::cout << "exception caught: " << err_msg << std::endl;
    }
    
### Multi-threading and Re-enterability(多线程和可重入性)
现在的OpenCV实现是完全可重入的。也就是说，同样的函数、一个类实例的相同不变方法、不同类实例的相同可变方法都能被不同的线程所调用。同样，同一个cv::Mat类的变量可以在不同的线程中使用，因为引用计数操作使用的是特定的原子指令。

 
