# Mask operations on matrices

---

矩阵的模板操作非常简单。其思想是通过一个模板矩阵（也称为核）重新计算每个像素点的值。模板的值表示邻域里每个像素的值（包括中心点现在的值）对中心点新值的影响有多大。从数学的角度看，这相当于用我们指定的权值求加权平均。

## **测试用例**

考虑一个图像对比度增强方法。我们想对图像中的每个像素进行以下计算（完整公式请看[这里](http://docs.opencv.org/master/d7/d37/tutorial_mat_mask_operations.html)）：

<center>I(i,j)=5∗I(i,j)−[I(i−1,j)+I(i+1,j)+I(i,j−1)+I(i,j+1)] </center>

<=>  I * M ,其中M = 
                   <center> i/j, -1, 0, +1</center>
                   <center> -1, 0, -1, 0</center>
                   <center> 0, -1, 5, -1</center>
                   <center> +1, 0, -1, 0</center>

第一个是计算公式，第二个是该式用模板操作的表现形式。具体做法是，把你想要计算的像素点和模板中心（上面的例子中是0,0（左上角））相对应，然后将模板的值乘上对应像素的值再求和。上面两个公式是等价的，但是在比较大的矩阵上后一个公式更直观。

现在让我们看看如何使用基本的像素访问方法或cv::filter2D函数来实现这一操作。

## **基础方法**

下面这个函数可以实现该功能：

    void Sharpen(const Mat& myImage,Mat& Result)
    {
        CV_Assert(myImage.depth() == CV_8U);  // 只接收uchar数组
        const int nChannels = myImage.channels();
        Result.create(myImage.size(),myImage.type());
        for(int j = 1 ; j < myImage.rows-1; ++j)
        {
            const uchar* previous = myImage.ptr<uchar>(j - 1);
            const uchar* current  = myImage.ptr<uchar>(j    );
            const uchar* next     = myImage.ptr<uchar>(j + 1);
            uchar* output = Result.ptr<uchar>(j);
            for(int i= nChannels;i < nChannels*(myImage.cols-1); ++i)
            {
                *output++ = saturate_cast<uchar>(5*current[i]
                             -current[i-nChannels] - current[i+nChannels] - previous[i] - next[i]);
            }
        }
        Result.row(0).setTo(Scalar(0));
        Result.row(Result.rows-1).setTo(Scalar(0));
        Result.col(0).setTo(Scalar(0));
        Result.col(Result.cols-1).setTo(Scalar(0));
    }
    
首先，我们要确保输入图像是无符号字符型。为此我们使用cv::CV_Assert函数来检测（如果不满足其中的表达式条件，它就会抛出异常）：

    CV_Assert(myImage.depth() == CV_8U);  // 只接收uchar图像
    
我们还要创建一个和输入相同大小、相同类型的输出图像。就像上一节所提到的，根据通道的数量，我们可能会有一到多个子列。

我们会通过指针迭代遍历它们，所以元素的总数量取决于通道数。

    const int nChannels = myImage.channels();
    Result.create(myImage.size(),myImage.type());
    
我们使用c中简单的[]操作符来访问像素。由于我们要同时访问多行，所以我们需要获得每行的指针（上一行，当前行（中心点所在的行），下一行）。我们还需要另一个指针指向存储计算结果的位置。然后，使用[]操作符来访问正确的元素吧。把输出指针(output)移到下一个位置的方式是在每次操作完成后对指针加一（一字节）：

    for(int j = 1 ; j < myImage.rows-1; ++j)
    {
        const uchar* previous = myImage.ptr<uchar>(j - 1);
        const uchar* current  = myImage.ptr<uchar>(j    );
        const uchar* next     = myImage.ptr<uchar>(j + 1);
        uchar* output = Result.ptr<uchar>(j);
        for(int i= nChannels;i < nChannels*(myImage.cols-1); ++i)
        {
            *output++ = saturate_cast<uchar>(5*current[i]
                         -current[i-nChannels] - current[i+nChannels] - previous[i] - next[i]);
        }
    }
    
在图像的边界，上面的操作会得到不存在的位置（比如负一减负一）。在这些点上我们的公式是没有定义的。简单的解决办法是不使用核（模板）来计算边界上的点，比如，把边界上的像素设为0：

    Result.row(0).setTo(Scalar(0));
    Result.row(Result.rows-1).setTo(Scalar(0));
    Result.col(0).setTo(Scalar(0));
    Result.col(Result.cols-1).setTo(Scalar(0));
    
## **filter2D函数**

图像处理中使用这些滤波器是很频繁的，OpenCV里存在这样的函数能够自动使用这些模板（在一些地方也称之为核）。为此，你需要先定义一个保存模板的对象：

    Mat kernel = (Mat_<char>(3,3) << 0, -1,  0,
                                    -1,  5, -1,
                                     0, -1,  0);
                                    
然后调用cv::filter2D函数并指定输入、输出以及核：

     filter2D( src, dst1, src.depth(), kernel );
     
这个函数还有第五个参数用于指定核的中心，第六个参数为被滤波的像素提供可选值，第七个参数指定在未定义区域（边界）的操作类型。

这个函数比较简洁，而且其中有优化措施，所以通常比手动敲的代码性能更好。比如，我测试的时候，第二种方法只花了13ms，而第一种方法花了31ms,确实相差很大。结果如下：

<center>![resultMatMaskFilter2D](http://docs.opencv.org/master/resultMatMaskFilter2D.png)</center>

你可以在[这里](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/mat_mask_operations/mat_mask_operations.cpp)下载源码，或在下面的文件目录里找到：

    samples/cpp/tutorial_code/core/mat_mask_operations/mat_mask_operations.cpp.
    
请在我们的[YouTuBe频道](http://www.youtube.com/watch?v=7PF1tAU9se4)查看演示视频。







