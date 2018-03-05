# How to scan images, lookup tables and time measurement with OpenCV（如何遍历图像、查找表以及测量时间）

---

## **目标**

我们将在这节解答以下问题：

 - 怎样遍历图像的每一个像素？
 - OpenCV的矩阵数值是如何存储的？
 - 如何衡量算法的性能？
 - 查找表是什么以及为什么要用查找表？

## **测试用例**

考虑一个简单的颜色压缩方法。如果使用c或c++中的无符号字符型作为矩阵元素的类型的话，像素每个通道都有256种可能的取值。对于三通道图像来说，有很多种颜色组合（准确地说是一千六百万种）。要处理如此多的颜色，肯定会对我们的算法性能造成很大影响。而且，很多时候只处理一小部分就足够达到同样的效果。

这个例子中，我们将实现一个颜色压缩方法。也就是说，我们将现在颜色空间的值除以一个输入的新值从而得到压缩的颜色空间。比如，在0到9之间的值都转换为0，10到19之间的值转换为10,诸如此类。

将uchar类型（无符号字符型，取值范围0到255）除以一个int型，得到的结果也是char型。这些值只会是char类型，因此，小数部分会被舍去。利用uchar值域的这种向下取整操作可以表示为：

<center>Inew=(Iold/10)∗10</center>

一个简单的颜色压缩算法包括遍历图像的每个像素，并对每个像素应用这个公式。值得注意的是，我们使用了除法和乘法操作，这些操作的开销很大。如果可能的话，最好使用开销没那么大的加减法或简单的赋值来替代它们。我们注意到，使用uchar系统的话，我们的输入是有限的，准确地说，是256个输入。

因此，对于比较大的图片，提前计算所有可能的值，并在赋值的时候运用查找表，只执行赋值操作，这样比较明智。查找表是一个简单的数组（一维或多维），一个输入变量对应一个最终的输出值。使用它的优点在于我们不需要进行计算，只需读取结果。

我们的测试用例程序有以下步骤：从控制台变量中获取图像名和用于图像压缩的整型变量，执行颜色压缩算法。OpenCV目前有三种主要的方法来遍历图像的每个像素。为了让实验更有趣，我们会测试这三种方法并分别记录每种方法所耗时间。

你能从[这里](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/how_to_scan_images/how_to_scan_images.cpp)下载到完整源码或者在OpenCV源码目录的core部分找到该样例。它的基本用法是：

    how_to_scan_images imageName.jpg intValueToReduce [G]
    
其中最后一个参数是可选的，表示释放将读取的图像转为灰度图，否则将使用BGR颜色空间。第一件事是计算查找表：

    int divideWith = 0; // 将输入字符串转换为数字 - C++ style
    stringstream s;
    s << argv[2];
    s >> divideWith;
    if (!s || !divideWith)
    {
        cout << "Invalid number entered for dividing. " << endl;
        return -1;
    }
    uchar table[256];
    for (int i = 0; i < 256; ++i)
       table[i] = (uchar)(divideWith * (i/divideWith));
       
这里，我们首次使用了C++的stringstream类将第三个命令行变量从文本转换为整型。然后我们使用前面提到的向下取整公式计算查找表。这部分和OpenCV无关。

另一个问题是，我们如何测量时间？对此，OpenCV提供了两个简单的函数cv::getTickCount()和cv::getTickFrequency()。第一个函数返回CPU从某一事件开始（如开机）经过的时间节拍数，第二个函数返回你的CPU一秒发出几个时间节拍。所以，测量两个操作之间的时间间隔（单位是秒）简单如下：

    double t = (double)getTickCount();
    // 运行些东西 ...
    t = ((double)getTickCount() - t)/getTickFrequency();
    cout << "Times passed in seconds: " << t << endl;
    
## **图像矩阵是如何存储在内存中的**

你可能已经在Mat-基本图像容器这一章节了解到了矩阵的大小取决于使用的颜色系统。更准确地说，是取决于通道的数量。对于灰度图来说，存储矩阵就像下面这样：

<center>![tutorial_how_matrix_stored_1](http://docs.opencv.org/master/tutorial_how_matrix_stored_1.png)</center>

对多通道图像来说，矩阵每一列的子列数和通道数一样。比如，使用BGR系统的存储矩阵如下：

<center>![tutorial_how_matrix_stored_2.png](http://docs.opencv.org/master/tutorial_how_matrix_stored_2.png)</center>

注意，通道的顺序反转了：从RGB变为BGR。由于大多情况下，内存有足够大的空间，可以一行接一行地连续存储，最终结果是存储为很长的一行。这样的存储方式能够加快扫描速度。我们可以用cv::Mat::isContinues()函数来确定矩阵是否以此方式存储。

## **一种高效的访问方式**

谈到性能，那就不得不用c风格的操作符(指针)进行访问了。对颜色压缩这个任务来说，最有效率的方法如下所示：

    Mat& ScanImageAndReduceC(Mat& I, const uchar* const table)
    {
        // 只接受字符类型的矩阵
        CV_Assert(I.depth() == CV_8U);
        int channels = I.channels();
        int nRows = I.rows;
        int nCols = I.cols * channels;
        if (I.isContinuous())
        {
            nCols *= nRows;
            nRows = 1;
        }
        int i,j;
        uchar* p;
        for( i = 0; i < nRows; ++i)
        {
            p = I.ptr<uchar>(i);
            for ( j = 0; j < nCols; ++j)
            {
                p[j] = table[p[j]];
            }
        }
        return I;
    }
    
这里，我们只是简单地将指针指向每一行的开头，然后遍历每一行。当矩阵是连续存储（存储为连续的一大行）的时候，我们只需为指针分配一次地址（即只有一行），然后一路走到底。访问彩色图像时要注意：因为有三个通道，所以每行要访问的数据是图像列数的三倍。

还有另一种方法。Mat对象的data成员变量返回指向第一行第一列的指针，假如指针为空，则表示没有有效数据输入。检查data指针是检测你是否成功读入图像的最简单方式。假如矩阵是连续存储的，那么我们可以通过data指针遍历整个矩阵，例如，如果存储的是灰度图的话：

    uchar* p = I.data;
    for( unsigned int i =0; i < ncol*nrows; ++i)
        *p++ = table[*p];
        
结果一样。但是经过一段时间后，你会发现这段代码会变得难以理解。假如你想用些更高级的技术，用这种方式可能会更难理解。而且，其实这两种方式性能相似（现代的编译器可能会自动进行这种优化）。

## **使用迭代器（安全）的访问方式**

在使用前面的高效访问方式时，你需要保证循环遍历不会越过矩阵的边界，而且要注意每一行访问结束要换行。迭代器(iterator)方法会更安全些，因为它会帮你考虑这些问题。你需要做的只是定义两个变量指向矩阵的开头后结尾，然后让迭代器从开头遍历到结尾就行了。使用*操作符来获取迭代器指向的值（把*加在it前面）:

    Mat& ScanImageAndReduceIterator(Mat& I, const uchar* const table)
    {
        //只接受字符类型数组
        CV_Assert(I.depth() == CV_8U);
        const int channels = I.channels();
        switch(channels)
        {
        case 1:
            {
                MatIterator_<uchar> it, end;
                for( it = I.begin<uchar>(), end = I.end<uchar>(); it != end; ++it)
                    *it = table[*it];
                break;
            }
        case 3:
            {
                MatIterator_<Vec3b> it, end;
                for( it = I.begin<Vec3b>(), end = I.end<Vec3b>(); it != end; ++it)
                {
                    (*it)[0] = table[(*it)[0]];
                    (*it)[1] = table[(*it)[1]];
                    (*it)[2] = table[(*it)[2]];
                }
            }
        }
        return I;
    }
    
对于彩色图像来说，每列有三项。这可看成是uchar类型的短向量，OpenCV用Vec3b进行命名。我们用[]操作符来访问每列的第n个子项。要记住，OpenCV的迭代器会自动跳到下一行。同时，如果你使用uchar类型的迭代器来访问彩色图像，只能访问到蓝色通道（每列第一个通道）。

## **返回引用时的野指针**

最后一种方法并不推荐。它是用来随机访问图像任意位置的。基本用法是你需要指定要访问的是第几行第几列。之前的例子中，你可能已经注意到确定图像的类型是很重要的。这里也一样，你需要指定访问数据的类型，在下面的访问灰度图例子中你可以发现这点（cv::at()函数的用法）：

    Mat& ScanImageAndReduceRandomAccess(Mat& I, const uchar* const table)
    {
        // 只接收字符类型的数组
        CV_Assert(I.depth() == CV_8U);
        const int channels = I.channels();
        switch(channels)
        {
        case 1:
            {
                for( int i = 0; i < I.rows; ++i)
                    for( int j = 0; j < I.cols; ++j )
                        I.at<uchar>(i,j) = table[I.at<uchar>(i,j)];
                break;
            }
        case 3:
            {
             Mat_<Vec3b> _I = I;
             for( int i = 0; i < I.rows; ++i)
                for( int j = 0; j < I.cols; ++j )
                   {
                       _I(i,j)[0] = table[_I(i,j)[0]];
                       _I(i,j)[1] = table[_I(i,j)[1]];
                       _I(i,j)[2] = table[_I(i,j)[2]];
                }
             I = _I;
             break;
            }
        }
        return I;
    }
    
at()函数需要输入数据类型和坐标来计算访问地址，然后返回一个引用。当你获取数据的时候可能是个常量，当你赋值数据的时候可能是个非常量。安全起见，在debug mode only下，函数会检查坐标是否有效。假如坐标无效，你会从标准错误输出流中得到一条错误消息。和前面的高效访问方式相比，此方法的不同之处在于访问每个图像数据的时候，你都会得到一个新的行指针，然后使用c操作符[]得到列元素。

假如你需要使用多维查找表，每次都要输入数据类型和at关键字会非常麻烦。为此，OpenCV提供了cv::Mat_数据类型。它和Mat类型功能相同，只是在有些情况需要指定访问数据的类型的时候能提供方便，即能够仅使用()操作符访问数据。还有一个优点是，cv::Mat_和cv::Mat之间转换非常方便。通过上面的例子，可以很容易地看出这一点。

尽管这样更方便，你还是需要记住cv::at()函数（这两种方式效率一样）。这只是懒惰的程序员不想写那么多字而使的一个小花招。

## **核心函数**

这里有一个更加福利的方法来让图像通过查找表进行变换操作。由于在图像处理中，经常需要将一整张图像的像素都进行某种变换，这里提到方法能很轻松地完成这项工作而不需要你写一个遍历图像的算法。这便是核心模块里的LTU()函数。首先，我们创建一个Mat类型的查找表：

    Mat lookUpTable(1, 256, CV_8U);
    uchar* p = lookUpTable.ptr();
    for( int i = 0; i < 256; ++i)
        p[i] = table[i];
        
然后，调用这个函数（I是输入图像，J是输出）：

     LUT(I, lookUpTable, J);
     
## **性能对比**

要想得到最好的比较结果，请你自己编译运行以上程序。为了更好地展示这些方法的不同之处，我使用了一张非常大的图（2560*1600），而且是彩色的。为了获得更精确的值，每个函数我都调用了上百次，并取它们平均值作为结果。

| Method        | Time          |
|:-------------:|:-------------:|
| 高效方式      | 79.4717 ms    |
| 迭代器方式    | 83.7201 ms    |
| 指针返回引用  | 93.7878 ms    |
| LUT函数       | 32.5759 ms    |

我们能从中总结一些东西。尽可能使用OpenCV自带的函数（而不是自己再写一遍）。最快的方法很明显是LTU()函数。这是因为OpenCV库通过英特尔线程构建块能够实现多线程加速。但是，如果你想写一个简单的不用指针的扫描图像方法，可以使用迭代器方法，虽然确实很慢。在debug模式里，指针返回引用方法是消耗最大的，而在release模式下，有可能比迭代器方法好，但确实没有迭代器那么安全。

最后，你可以通过我们在YouTuBe的频道观看运行这个例子程序的[视频](https://www.youtube.com/watch?v=fB3AN5fjgwc)。