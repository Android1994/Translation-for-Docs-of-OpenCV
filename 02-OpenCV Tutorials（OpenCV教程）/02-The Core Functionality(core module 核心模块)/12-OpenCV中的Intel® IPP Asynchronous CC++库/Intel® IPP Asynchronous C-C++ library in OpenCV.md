# Intel® IPP Asynchronous C/C++ library in OpenCV（OpenCV中的Intel® IPP Asynchronous CC++库）


---

## **目标**
这篇教程是关于如何在OpenCV中使用[Intel® IPP Asynchronous C/C++](http://software.intel.com/en-us/intel-ipp-preview)库。下面的例子演示了如何利用Intel® IPP Asynchronous C/C++函数对Sobel算子的实现进行加速。在例子中，[cv::hpp::getMat](http://docs.opencv.org/master/d6/dcf/group__core__ipp.html#ga6b6a7a9c44efd50c8aaf9fb0866a363a)和[cv::hpp::getHpp](http://docs.opencv.org/master/d6/dcf/group__core__ipp.html#gae751e7f96d79f90699fdf71bdaa87a6b)函数是用来将[hppiMatrix](http://software.intel.com/en-us/node/501660)转换为Mat。

## **代码**
你也可以从[这里](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/ippasync/ippasync_sample.cpp)下载代码。

```
#include <stdio.h>
#include "opencv2/core/utility.hpp"
#include "opencv2/imgproc.hpp"
#include "opencv2/highgui.hpp"
#include "cvconfig.h"
using namespace std;
using namespace cv;
#ifdef HAVE_IPP_A
#include "opencv2/core/ippasync.hpp"
#define CHECK_STATUS(STATUS, NAME)\
    if(STATUS!=HPP_STATUS_NO_ERROR){ printf("%s error %d\n", NAME, STATUS);\
    if (virtMatrix) {hppStatus delSts = hppiDeleteVirtualMatrices(accel, virtMatrix); CHECK_DEL_STATUS(delSts,"hppiDeleteVirtualMatrices");}\
    if (accel)      {hppStatus delSts = hppDeleteInstance(accel); CHECK_DEL_STATUS(delSts, "hppDeleteInstance");}\
    return -1;}
#define CHECK_DEL_STATUS(STATUS, NAME)\
    if(STATUS!=HPP_STATUS_NO_ERROR){ printf("%s error %d\n", NAME, STATUS); return -1;}
#endif
static void help()
{
 printf("\nThis program shows how to use the conversion for IPP Async.\n"
"This example uses the Sobel filter.\n"
"You can use cv::Sobel or hppiSobel.\n"
"Usage: \n"
"./ipp_async_sobel [--camera]=<use camera,if this key is present>, \n"
"                  [--file_name]=<path to movie or image file>\n"
"                  [--accel]=<accelerator type: auto (default), cpu, gpu>\n\n");
}
const char* keys =
{
    "{c  camera   |           | use camera or not}"
    "{fn file_name|../data/baboon.jpg | image file       }"
    "{a accel     |auto       | accelerator type: auto (default), cpu, gpu}"
};
//this is a sample for hppiSobel functions
int main(int argc, const char** argv)
{
    help();
    VideoCapture cap;
    CommandLineParser parser(argc, argv, keys);
    Mat image, gray, result;
#ifdef HAVE_IPP_A
    hppiMatrix* src,* dst;
    hppAccel accel = 0;
    hppAccelType accelType;
    hppStatus sts;
    hppiVirtualMatrix * virtMatrix;
    bool useCamera = parser.has("camera");
    string file = parser.get<string>("file_name");
    string sAccel = parser.get<string>("accel");
    parser.printMessage();
    if( useCamera )
    {
        printf("used camera\n");
        cap.open(0);
    }
    else
    {
        printf("used image %s\n", file.c_str());
        cap.open(file.c_str());
    }
    if( !cap.isOpened() )
    {
        printf("can not open camera or video file\n");
        return -1;
    }
    accelType = sAccel == "cpu" ? HPP_ACCEL_TYPE_CPU:
                sAccel == "gpu" ? HPP_ACCEL_TYPE_GPU:
                                  HPP_ACCEL_TYPE_ANY;
    //Create accelerator instance
    sts = hppCreateInstance(accelType, 0, &accel);
    CHECK_STATUS(sts, "hppCreateInstance");
    accelType = hppQueryAccelType(accel);
    sAccel = accelType == HPP_ACCEL_TYPE_CPU ? "cpu":
             accelType == HPP_ACCEL_TYPE_GPU ? "gpu":
             accelType == HPP_ACCEL_TYPE_GPU_VIA_DX9 ? "gpu dx9": "?";
    printf("accelType %s\n", sAccel.c_str());
    virtMatrix = hppiCreateVirtualMatrices(accel, 1);
    for(;;)
    {
        cap >> image;
        if(image.empty())
            break;
        cvtColor( image, gray, COLOR_BGR2GRAY );
        result.create( image.rows, image.cols, CV_8U);
        double execTime = (double)getTickCount();
        //convert Mat to hppiMatrix
        src = hpp::getHpp(gray,accel);
        dst = hpp::getHpp(result,accel);
        sts = hppiSobel(accel,src, HPP_MASK_SIZE_3X3,HPP_NORM_L1,virtMatrix[0]);
        CHECK_STATUS(sts,"hppiSobel");
        sts = hppiConvert(accel, virtMatrix[0], 0, HPP_RND_MODE_NEAR, dst, HPP_DATA_TYPE_8U);
        CHECK_STATUS(sts,"hppiConvert");
        // Wait for tasks to complete
        sts = hppWait(accel, HPP_TIME_OUT_INFINITE);
        CHECK_STATUS(sts, "hppWait");
        execTime = ((double)getTickCount() - execTime)*1000./getTickFrequency();
        printf("Time : %0.3fms\n", execTime);
        imshow("image", image);
        imshow("rez", result);
        waitKey(15);
        sts =  hppiFreeMatrix(src);
        CHECK_DEL_STATUS(sts,"hppiFreeMatrix");
        sts =  hppiFreeMatrix(dst);
        CHECK_DEL_STATUS(sts,"hppiFreeMatrix");
    }
    if (!useCamera)
        waitKey(0);
    if (virtMatrix)
    {
        sts = hppiDeleteVirtualMatrices(accel, virtMatrix);
        CHECK_DEL_STATUS(sts,"hppiDeleteVirtualMatrices");
    }
    if (accel)
    {
        sts = hppDeleteInstance(accel);
        CHECK_DEL_STATUS(sts, "hppDeleteInstance");
    }
    printf("SUCCESS\n");
#else
    printf("IPP Async not supported\n");
#endif
    return 0;
}
```

## **解释**

 1. 创建OpenCV变量：
 
    ```
    VideoCapture cap;
    Mat image,gray,result;
    ```
    
    以及IPP Async变量：
    
    ```
    hppiMatrix* src,* dst;
    hppAccel accel=0;
    hppAccelType accelType;
    hppStatus sts;
    hppiVirtualMatrix* virtMatrix;
    ```

 2. 读取图像或视频。关于读取视频流，你可以查看[Video Input with OpenCV and similarity measurement](http://docs.opencv.org/master/d5/dc4/tutorial_video_input_psnr_ssim.html)。
 
    ```
    if( useCamera )
    {
       printf("used camera\n");
       cap.open(0);
    }
    else
    {
       printf("used image %s\n", file.c_str());
       cap.open(file.c_str());
    }
    if( !cap.isOpened() )
    {
       printf("can not open camera or video file\n");
       return -1;
    }
    ```
    
 3. 使用[hppiCreateInstance](http://software.intel.com/en-us/node/501686)创建加速器实例：
 
    ```
    accelType = sAccel == "cpu" ? HPP_ACCEL_TYPE_CPU:
                sAccel == "gpu" ? HPP_ACCEL_TYPE_GPU:
                                  HPP_ACCEL_TYPE_ANY;
    //Create accelerator instance
    sts = hppCreateInstance(accelType, 0, &accel);
    CHECK_STATUS(sts, "hppCreateInstance");
    ```
    
 4. 使用[hppiCreateVirtualMatrices]()函数创建一个虚矩阵的数组。
 
    ```
    virMatrix=hppiCreateVirtualMatrices(accel,1);
    ```
    
 5. 为输入输出数据创建一个矩阵:

    ```
    cap>>image;
    if(image.empty())
        break;
    
    cvtColor(image,gray,COLOR_BGR2GRAY);
    result.create(image.rows,image.cols,CV_8U);
    ```
    
 6. 使用[cv::hpp::getHpp](http://docs.opencv.org/master/d6/dcf/group__core__ipp.html#gae751e7f96d79f90699fdf71bdaa87a6b)将Mat转换为[hppiMatrix](http://software.intel.com/en-us/node/501660)并且调用[hppiSobel](http://software.intel.com/en-us/node/474701)函数。
    
    ```
    //convert Mat to hppiMatrix
    src = getHpp(gray, accel);
    dst = getHpp(result, accel);
    sts = hppiSobel(accel,src, HPP_MASK_SIZE_3X3,HPP_NORM_L1,virtMatrix[0]);
    CHECK_STATUS(sts,"hppiSobel");
    sts = hppiConvert(accel, virtMatrix[0], 0, HPP_RND_MODE_NEAR, dst, HPP_DATA_TYPE_8U);
    CHECK_STATUS(sts,"hppiConvert");
    // Wait for tasks to complete
    sts = hppWait(accel, HPP_TIME_OUT_INFINITE);
    CHECK_STATUS(sts, "hppWait");
    ```
    
    使用[hppiConvert](http://software.intel.com/en-us/node/501746)是因为[hppiSobel](http://software.intel.com/en-us/node/474701)返回的目标矩阵是HPP_DATA_TYPE_16S类型的，而原数据是HPP_DATA_TYPE_8U类型的。并且调用IPP Async函数后需要检查hppStatus的状态。
    
 7. 创建显示窗口并显示图像：
 
    ```
    imshow("image",iamge);
    imshow("rez",result);
    
    waitKey(15);
    ```
    
 8. 删除hpp矩阵：
 
    ```
    sts =  hppiFreeMatrix(src);
    CHECK_DEL_STATUS(sts,"hppiFreeMatrix");
    sts =  hppiFreeMatrix(dst);
    CHECK_DEL_STATUS(sts,"hppiFreeMatrix");
    ```
    
 9. 删除虚矩阵以及加速器实例。
 
    ```
    if (virtMatrix)
    {
       sts = hppiDeleteVirtualMatrices(accel, virtMatrix);
       CHECK_DEL_STATUS(sts,"hppiDeleteVirtualMatrices");
    }
    if (accel)
    {
       sts = hppDeleteInstance(accel);
       CHECK_DEL_STATUS(sts, "hppDeleteInstance");
    }
    ```
    
## **结果**

对以上代码进行编译后，我们可以将图像或视频路径以及加速器类型作为参数输入并执行。结果如下所示：

<center>![How_To_Use_IPPA_Result.jpg](http://docs.opencv.org/master/How_To_Use_IPPA_Result.jpg)</center>
 
