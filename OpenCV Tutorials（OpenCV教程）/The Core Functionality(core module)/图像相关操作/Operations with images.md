# Operations with images


---

## **输入输出**
### **图像**

从文件中读取图像：

    Mat img = imread(filename);

假如你读取一个jpg文件，会默认创建一个3通道的图像。假如你需要灰度图，请使用：

    Mat img = imread(filename, 0);
    
> Note:
  当把图像数据写入文件的时候，具体的格式由图像内容决定（前几个字节）：
  

    imwrite(filename, img);
    
> Note:
  文件的格式由扩展名决定。
  使用imencode和imdecode是从内存中读写图像，而非从文件中读写图像。
  
## **图像的基本操作**
### **获取像素的强度值**

为获取像素强度值，你必须知道图像类型和通道数，例如单通道灰度图上(8U1C)坐标为x,y的像素强度值为:

    Scalar intensity = img.at<uchar>(y, x);
    
intensity.val[0]的取值范围是0-255，注意x和y的顺序。由于OpenCV中的图像和矩阵结构相同，我们使用双方的一些惯例--行号（y轴）在前，列号(x轴)在后，你也可以使用下列方法：

    Scalar intensity = img.at<uchar>(Point(x, y));
    
考虑一个BGR三通道图像的例子（imread可自动获得图像格式）：

    Vec3b intensity = img.at<Vec3b>(y, x);
    uchar blue = intensity.val[0];
    uchar green = intensity.val[1];
    uchar red = intensity.val[2];
    
你可以对浮点图使用相同方法（你能通过在三通道图像上使用Sobel算子得到该类型图像）:

    Vec3f intensity = img.at<Vec3f>(y, x);
    float blue = intensity.val[0];
    float green = intensity.val[1];
    float red = intensity.val[2];
    
改变像素强度的方法也一样：

    img.at<uchar>(y, x) = 128;
    
OpenCV中还有些函数能从Mat中抽取一组2D或3D点，特别是在calib3d模块中，比如projectPoints。得到的矩阵只有一列，每个点代表原图的一行，相应的，得到的矩阵的类型是32FC2或32FC3。这样的矩阵可以通过std::vector轻松得到：

    vector<Point2f> points;
    //... 填充数组
    Mat pointsMat = Mat(points);
    
同样可以用Mat::at方法访问该矩阵的每个点：

    Point2f point = pointsMat.at<Point2f>(i, 0);
    
### **内存管理和引用计数**
Mat是一个包含图像/矩阵特征（行数、列数、数据类型等）和一个指向数据的指针的结构体。所以，对同一块数据，可以对应多个Mat实例。每个Mat实例都有一个引用计数器用于决定当某个Mat实例被销毁时，是否要释放内存中对应数据块。下面是不使用复制创建两个矩阵的例子：

    std::vector<Point3f> points;
    // .. fill the array
    Mat pointsMat = Mat(points).reshape(1);
    
最终，我们得到一个3列的32FC1矩阵而非1列的32FC3矩阵。pointsMat使用points的数据，并且当销毁pointsMat时并不会把数据销毁。在这个例子中，开发人员必须保证points的生命周期比pointsMat的生命周期长。假如我们要拷贝数据，可以使用cv::Mat::copyTo或cv::Mat::clone:

    Mat img = imread("image.jpg");
    Mat img1 = img.clone();
    
和c风格API中输出图像必须由开发者创建不同的是，现在空的Mat也能作为输出应用到每个函数中。通过对目标矩阵调用Mat::create实现。假如某个矩阵是空的，该方法会为其分配数据。假如矩阵不是空的，并且有正确的大小和类型，那么这个函数不会做任何事情。但是，如果矩阵的大小或类型和输入参数不一致，那么矩阵会被重新分配（数据会丢失）。比如：

    Mat img = imread("image.jpg");
    Mat sobelx;
    Sobel(img, sobelx, CV_32F, 1, 0);
    
### **基本操作**

OpenCV里有很多方便的矩阵操作。比如，通过以下方式，我们可以把一个灰度图'img'变为全黑：

    img = Scalar(0);

设置感兴趣区域（ROI）：

    Rect r(10, 10, 100, 100);
    Mat smallImg = img(r);
    
将Mat转换为c风格API中的数据结构体：

    Mat img = imread("image.jpg");
    IplImage img1 = img;
    CvMat m = img;
    
注意，以上都没有数据被拷贝的情况。
从彩色图转换为灰度图：

    Mat img = imread("image.jpg"); // 读取一个8UC3类型的图像
    Mat grey;
    cvtColor(img, grey, COLOR_BGR2GRAY);
    
将图像由8UC1转换为32FC1:

    src.convertTo(dst, CV_32F);
    
### **图像可视化**

在开发过程中，立即看到算法的效果是很有用的。OpenCV为图像显示提供了方便的方式。8位无符号的图像可以用以下方式显示：

    Mat img = imread("image.jpg");
    namedWindow("image", WINDOW_AUTOSIZE);
    imshow("image", img);
    waitKey();
    
调用waitKey()会一直循环等待在图像窗口中敲入按键，32F类型的图像需要被转换为8U类型，比如：

    Mat img = imread("image.jpg");
    Mat grey;
    cvtColor(img, grey, COLOR_BGR2GRAY);
    Mat sobelx;
    Sobel(grey, sobelx, CV_32F, 1, 0);
    double minVal, maxVal;
    minMaxLoc(sobelx, &minVal, &maxVal); //找到最小和最大值
    Mat draw;
    sobelx.convertTo(draw, CV_8U, 255.0/(maxVal - minVal), -minVal * 255.0/(maxVal - minVal));
    namedWindow("image", WINDOW_AUTOSIZE);
    imshow("image", draw);
    waitKey();



