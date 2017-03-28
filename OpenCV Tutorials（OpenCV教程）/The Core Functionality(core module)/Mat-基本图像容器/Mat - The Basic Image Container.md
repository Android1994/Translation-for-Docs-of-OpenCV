# Mat - The Basic Image Container

---

## **GOAL**

从现实世界获取数字图像的方法有很多：数码相机、扫描仪、计算机断层扫描(CT)、磁共振成像(MRI)等。 不管以哪种方法，我们看到的都是图像的形式。然而，实际上，图像上每一个点都有一个数值，将图像存储到数码设备中即是储存这些数值。

<center>![MatBasicImageForComputer](http://docs.opencv.org/master/MatBasicImageForComputer.jpg)</center>

如上图所示，这幅车的图像只是一个包含像素值的密集矩阵。获取和存储像素值的方式应需求而不同，但最终都是把图像变为一个数值矩阵，同时还要存储描述这个矩阵的信息。OpenCV库主要目标就是操作和处理这些信息。因此，你需要知道的第一件事就是OpenCV是如何存储并处理图像的。

## **Mat**
OpenCV诞生于2001年。一开始，OpenCV用的是c风格的接口，使用名为IPlImage的c结构体存储图像数据。你会在以前的教程里经常看到这个结构体，伴其而来的是c语言的各种缺点。最大的问题是复杂的人为内存管理，彼时的OpenCV是建立在默认用户需对内存的分配和管理负责的基础上的。这对小程序来说，不是问题，然而一旦你的代码库持续增长，处理这些问题就会变得很困难，不如投入更多精力专注于你的目标。

幸运的是，我们还有c++，其中类概念的提出以及自动内存管理为用户提供了很大的方便。另一个好消息是c++完全兼容c，所以这一改变不会带来兼容问题。因此，OpenCV2.0引入了新的c++接口，使用户不需再疲于应对内存管理，并且可以让你的代码更精简（写得更少，实现得更多）。c++接口的主要缺点是现在很多嵌入式系统只支持c。所以，除非你专注于嵌入式平台的开发，没有其它理由让你继续使用老接口了（除非你是受虐狂）。

关于Mat你需要知道的第一件事就是再也不需要手动为其分配或释放内存了。虽然有时候可能仍需要手动管理内存，但是大多数的OpenCV函数都会自动为输出数据分配空间。一个比较好的附带效应是，如果你传递一个已经分配了空间的Mat对象，那么它会被重用。换句话说，我们总是尽可能高效地利用内存空间来达到我们的目的。

Mat类主要有两个数据成员：矩阵头（包含诸如矩阵大小、存储方式、矩阵存储地址等信息）和指向矩阵的指针（矩阵维数取决于像素存储方式）。矩阵头的大小是常数，而矩阵本身的大小就因图像而异了。

OpenCV是图像处理库，收集了大量图像处理方面的函数。为了解决问题，你会用到各种各样的函数，因此，传递图像数据给函数是经常发生的。我们不该忘了图像处理函数的计算复杂度是很高的。我们最不希望看到的就是由于不必要的大量图像数据拷贝而减慢程序运行的速度。

为了解决这个问题，OpenCV使用了引用计数系统。其思想是每一个Mat对象都有自己的矩阵头，但是这个矩阵对象能被它的两个实例共享，即让实例的指针都指向该矩阵地址。或者说，复制操作只会复制矩阵头以及指针，而不会复制数据本身。

    Mat A, C;                          // 只是创建了矩阵头
    A = imread(argv[1], IMREAD_COLOR); // 我们知道什么方法被调用了(为矩阵分配空间)
    Mat B(A);                                 // 使用复制构造函数
    C = A;  
    
以上所有Mat对象都指向同一个矩阵。但是，它们的矩阵头不太一样，而且对其中任一对象进行修改都会影响其它对象。事实上，不同的对象只是对同一数据不同的访问方法。尽管如此，它们的矩阵头部分是不一样的。更有意思的地方是你可以为引用一个数据的子集而创建矩阵头。例如，在创建感兴趣区域（ROI）的时候，你只是以新的矩阵尺寸创建了一个新的矩阵头：

    Mat D (A, Rect(10, 10, 100, 100) ); // 使用了矩形对象
    Mat E = A(Range::all(), Range(1,3)); // 使用行数和列数作为界限

现在，你可能会问，当矩阵不再使用时，是否是被多个矩阵对象引用的矩阵本身负责释放空间。答案很简单：最后使用这个矩阵的对象来做这件事。通过使用引用计数机制来实现这一功能。每当有新的对象指向这块矩阵数据，引用计数器就加一，一旦有对象被清理，计数器就减一。当计数器为0时，这块数据就被释放。有时候你可能会想要真正拷贝一份矩阵数据，对此，OpenCV提供了cv::Mat::clone()和cv::Mat::copyTo()函数：

    Mat F = A.clone();
    Mat G;
    A.copyTo(G);
    
现在，修改F或G都不会再影响原始数据了。总之，你需要记住的就是：

 - OpenCV中的函数会为输出的图像数据自动分配空间（除一些特别情况）；
 - 使用OpenCV的c++接口，你不需过多考虑内存管理；
 - 赋值操作和复制构造函数只复制矩阵头；
 - 要真正获得图像数据的独立拷贝，可使用cv::Mat::clone()和cv::Mat::copyTo()函数。
 
## **存储方式**
这和你存储像素值的方式有关，你可以选择不同的颜色空间和数据类型。颜色空间指的是我们对颜色进行编码的方式。最简单的颜色空间是灰度空间，我们处理的颜色是黑和白。不同的组合产生很多灰度层级。

存储彩色信息的方式有很多。每一种方式基本上都是由三到四个基元组成，我们可以用这些基元组合成其它元素。最常用的是RGB颜色空间，主要是因为这和人眼处理颜色的机制相似。RGB的基本颜色是红、绿、蓝。为了编码透明颜色，添加了第四个元素，alpha(A)。

不同的颜色空间各有优势：

 - RGB颜色空间和人眼的感觉最相近，但是请记住，OpenCV标准显示系统用的是BGR（红色通道和蓝色通道交换了位置）。
 - HSV和HLS空间将颜色分解成色调、饱和度和亮度，是一种更自然地描述色彩的方式。使用这种颜色空间，你可以通过忽视最后一个元素（亮度）来让算法对输入图像的光照条件不敏感。
 - YCrCb颜色空间用在常见的JPEG格式图像中。
 - CIE L*a*b*是具有感知一致性的颜色空间。假如你想衡量给定颜色的距离，用这个空间会很方便。
 
每个不同的元素都有自己的取值域，这影响着数据类型的选择。如何存储元素取决于它的取值范围。最小的数据类型应该是char类型，长度是一字节或者说是8比特。可以是无符号（取值范围0-255）或有符号（取值范围-127-127）。尽管对于含有三个元素的颜色空间来说，这样已经有一千六百万种可能的组合（256^3），使用float或double仍可以很好地hold住每个像素。尽管如此，你需要记住，增加元素的尺寸，存储图像所需的空间也会增加。

## **准确地创建一个Mat对象
在“Load, Modify, and Save an Image ”这一节教程中，你已经学会如何使用cv::imwrite()函数将矩阵数据写入图像文件了。出于调试的目的，能够看到真实数值会更方便一些。为此，你可以对Mat使用<<操作符。注意，这只适用于二维矩阵。


Mat作为图像数据的容器确实做得很好，它同时也是一个矩阵类。因此，通过Mat创建和操作多维矩阵也是可能的。你能通过很多方法创建Mat对象：

- cv::Mat::Mat构造函数

  例如：

    `Mat M(2,2, CV_8UC3, Scalar(0,0,255));
    cout << "M = " << endl << " " << M << endl << endl;`
<center>![MatBasicContainerOut1](http://docs.opencv.org/master/MatBasicContainerOut1.png)</center>

  对于两维多通道图像，我们先定义它们的尺寸：依次为行数和列数。
  
  接下来，我们要定义存储元素的数据类型以及每个矩阵元素的通道数，规则如下：
  

    `CV_[每个通道的比特数][有符号或无符号][类型前缀]C[通道数]`
   
  比如，CV_8UC3表示每个像素有3个通道，每个通道是8比特长的无符号字符型，这是被预先定义好的。cv::Scalar是含有4个元素的短向量，你可以通过它初始化每个像素的值。如果你想要创建更多维数的矩阵，你可以使用上面的类型宏，把通道数放在圆括号里，如下所示。
  
- 在构造函数里使用c/c++的数组进行初始化
  `int sz[3] = {2,2,2};
  Mat L(3,sz, CV_8UC(1), Scalar::all(0));`

  上面的例子展示了如何创建超过两位的矩阵，先传递维数，再传递一个包含每一维大小的指针，其它和前面一样。
  
- cv::Mat::create()函数
  `M.create(4,4,CV_8UC(2));
   cout<<"M="<<endl<<" "<<M<<endl<<endl`

<center>![MatBasicContainerOut2](http://docs.opencv.org/master/MatBasicContainerOut2.png)</center>

   通过这种方法创建Mat，你不能初始化元素的值。当新矩阵大小与旧矩阵不匹配时，它会重新分配数据内存空间。
   
- Matlab风格的初始化: cv::Mat::zeros,cv::Mat::ones,cv::Mat::eye。指定矩阵大小和数据类型:
  `Mat E = Mat::eye(4, 4, CV_64F);
    cout << "E = " << endl << " " << E << endl << endl;
    Mat O = Mat::ones(2, 2, CV_32F);
    cout << "O = " << endl << " " << O << endl << endl;
    Mat Z = Mat::zeros(3,3, CV_8UC1);
    cout << "Z = " << endl << " " << Z << endl << endl;`

<center>![MatBasicContainerOut3](http://docs.opencv.org/master/MatBasicContainerOut3.png)</center>

- 对于小矩阵，你可以使用逗号分隔进行初始化：
  ` Mat C = (Mat_<double>(3,3) << 0, -1, 0, -1, 5, -1, 0, -1, 0);
    cout << "C = " << endl << " " << C << endl << endl;`
<center>![MatBasicContainerOut6](http://docs.opencv.org/master/MatBasicContainerOut6.png)</center>

- 对已存在的矩阵对象创建新的矩阵头，使用cv::Mat::clone和cv::Mat::copyTo复制数据：
  <center>![MatBasicContainerOut7](http://docs.opencv.org/master/MatBasicContainerOut7.png)</center>

> Note:
       你可以使用cv::randu()函数对矩阵元素赋随机值，你需要给出上下界
       `Mat R = Mat(3, 2, CV_8UC3);
        randu(R, Scalar::all(0), Scalar::all(255));`
        
        
## **输出格式**
在上面的例子中，你可以看到默认的格式选项。OpenCV允许你对输出矩阵进行格式化：

- 默认:
  `cout << "R (default) = " << endl <<        R           << endl << endl;`
  <center>![MatBasicContainerOut8](http://docs.opencv.org/master/MatBasicContainerOut8.png)</center>

- Python:
  `cout << "R (python)  = " << endl << format(R, Formatter::FMT_PYTHON) << endl << endl;`
  <center>![MatBasicContainerOut16](http://docs.opencv.org/master/MatBasicContainerOut16.png)</center>

- 逗号分隔值(CSV):
  `cout << "R (csv)     = " << endl << format(R, Formatter::FMT_CSV   ) << endl << endl;`
  <center>![MatBasicContainerOut10](http://docs.opencv.org/master/MatBasicContainerOut10.png)</center>

- Numpy:
  ` cout << "R (numpy)   = " << endl << format(R, Formatter::FMT_NUMPY ) << endl << endl;`
  <center>![MatBasicContainerOut9](http://docs.opencv.org/master/MatBasicContainerOut9.png)</center>

- C:
  `cout << "R (c)       = " << endl << format(R, Formatter::FMT_C     ) << endl << endl;`
  <center>![MatBasicContainerOut11](http://docs.opencv.org/master/MatBasicContainerOut11.png)</center>

## **其它数据的输出**

OpenCV支持使用<<操作符来输出其它常见的OpenCV数据结构：

- 2D Point
  `Point2f P(5, 1);
    cout << "Point (2D) = " << P << endl << endl;`
  <center>![MatBasicContainerOut12](http://docs.opencv.org/master/MatBasicContainerOut12.png)</center>

- 3D Point
  `Point3f P3f(2, 6, 7);
    cout << "Point (3D) = " << P3f << endl << endl;`
  <center>![MatBasicContainerOut13](http://docs.opencv.org/master/MatBasicContainerOut13.png)</center>

- 通过cv::Mat输出std::vector
  ` vector<float> v;
    v.push_back( (float)CV_PI);   v.push_back(2);    v.push_back(3.01f);
    cout << "Vector of floats via Mat = " << Mat(v) << endl << endl;`
<center>![MatBasicContainerOut14](http://docs.opencv.org/master/MatBasicContainerOut14.png)</center>
  
- 输出像素点向量
  `  vector<Point2f> vPoints(20);
    for (size_t i = 0; i < vPoints.size(); ++i)
        vPoints[i] = Point2f((float)(i * 5), (float)(i % 7));
    cout << "A vector of 2D Points = " << vPoints << endl << endl;`
<center>![MatBasicContainerOut15](http://docs.opencv.org/master/MatBasicContainerOut15.png)</center>

以上大部分例子都在一个小型的控制台应用里，你能从[此](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/mat_the_basic_image_container/mat_the_basic_image_container.cpp)下载或从cpp sample文件夹里的core部分找到。
你也能从[YouTube](https://www.youtube.com/watch?v=1tibU7vGWpk)找到一个小型的演示demo。

  

 