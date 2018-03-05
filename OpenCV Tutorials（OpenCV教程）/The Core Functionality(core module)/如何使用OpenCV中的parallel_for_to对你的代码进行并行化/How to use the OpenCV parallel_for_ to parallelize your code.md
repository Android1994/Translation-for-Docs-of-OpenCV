# How to use the OpenCV parallel_for_to parallelize your code（如何使用OpenCV中的parallel_for_to对你的代码进行并行化）


---

## **目标**
这篇教程的目的是演示如何使用OpenCV的parallel_for_框架对代码进行并行化。这里，我们将编写一个画出曼德布洛特集合的程序，并尽可能的让cpu满负荷运行。完整的代码可在[此处](https://github.com/opencv/opencv/blob/master/samples/cpp/tutorial_code/core/how_to_use_OpenCV_parallel_for_/how_to_use_OpenCV_parallel_for_.cpp)找到。假如你想要获得更多有关多线程的资料，需要额外翻阅参考书或学习相关课程，因为本篇教程只是简单的示例。

## **前提**
首要的前提是保证OpenCV和某个并行框架都已安装并配置好。对于OpenCV3.2来说，以下并行框架都可使用：

1. Intel Threading Building Blocks （第三方库，必须确保确实可用）
2. C= Parallel C/C++ Programming Language Extension（第三方库，必须确保确实可用）
3. OpenMP（集成到编译器，必须确保确实可用）
4. APPLE GCD（全系统可用，自动调用(只能在APPLE系统上使用)）
5. Windows RT concurrency（全系统可用，自动调用（只能在Windows RT上使用））
6. Windows concurrency（运行时执行，自动调用（只能在Windows上使用并且需求MSVC++ >= 10））
7. Pthreads（假如可用的话）

可以看到，有好几个并行框架都可以和OpenCV库混合使用。有些并行库是第三方的，需要保证使用CMake安装好并确实可用（比如 TBB,C=），其它还有些是在特定平台自动可用的（比如APPLE GCD），但是保险起见，你需要确认可以直接访问到这些并行库或者用CMake重新设置相关选项并重新编译生成这些库。

第二个前提和你的目标任务相关，因为并不是所有计算都适合或者说能够并行执行。为了保证简单，能够被分解成多个基本操作并且这些基本操作间没有存储依赖（不会竞争存储资源）的任务较容易进行并行化。计算机视觉的程序一般是较容易进行并行化的，因为大多数时候，对单个像素的处理不会依赖于其他像素。

## **简单的例子：画一个Mandelbrot set（曼德布洛特集合）**
我们将使用一个画曼德布洛特集合的例子来展示如何将一个串行运行的代码改为并行执行。

### **理论**
Mandelbrot set（曼德布洛特集合）的名字是由数学家Adrien Douady起的，为了纪念数学家Benoit Mandelbrot。它在其它领域也非常出名，因为它是一个对一类分形进行图像表示的例子，是一个在每个尺度以相同的模式呈现的数学集合（更甚者，Mandelbrot set是自相似的，整个形状在不同的尺度重复出现）。更详细的介绍可以参看相关的[维基百科](https://en.wikipedia.org/wiki/Mandelbrot_set)。

> Mandelbrot set的定义，参看[这里](https://docs.opencv.org/master/d7/dff/tutorial_how_to_use_OpenCV_parallel_for_.html)。

### **伪代码**
一个生成Mandelbrot set表征的简单算法叫做["escape time algorithm"（逃逸时间算法）](https://en.wikipedia.org/wiki/Mandelbrot_set#Escape_time_algorithm)。对目标图像的每一个像素，假如复数是有界限的并且不超过最大迭代次数的话我们就尝试使用循环关系。我们假设经过一个固定的迭代次数后，不属于Mandelbrot set的像素将会很快地“逃逸”，而剩下的像素即属于Mandelbrot set。

```
For each pixel (Px, Py) on the screen, do:
{
  x0 = scaled x coordinate of pixel (scaled to lie in the Mandelbrot X scale (-2, 1))
  y0 = scaled y coordinate of pixel (scaled to lie in the Mandelbrot Y scale (-1, 1))
  x = 0.0
  y = 0.0
  iteration = 0
  max_iteration = 1000
  while (x*x + y*y < 2*2  AND  iteration < max_iteration) {
    xtemp = x*x - y*y + x0
    y = 2*x*y + y0
    x = xtemp
    iteration = iteration + 1
  }
  color = palette[iteration]
  plot(Px, Py, color)
}
```

下列公式联系了伪代码和理论：
[一些公式](https://docs.opencv.org/master/d7/dff/tutorial_how_to_use_OpenCV_parallel_for_.html)。。。

<center>![how_to_use_OpenCV_parallel_for_640px-Mandelset_hires.png](https://docs.opencv.org/master/how_to_use_OpenCV_parallel_for_640px-Mandelset_hires.png)</center>

在这张图上，x轴对应复数的实部，y轴对应复数第虚部。可以发现，如果对特定部位进行缩放的话，整个形状是重复出现的。

### **实现**
### **逃逸时间算法的实现**

```
int mandelbrot(const complex<float> &z0, const int max)
{
    complex<float> z = z0;
    for (int t = 0; t < max; t++)
    {
        if (z.real()*z.real() + z.imag()*z.imag() > 4.0f) return t;
        z = z*z + z0;
    }
    return max;
}
```

我们在这里使用了`std::complex`模板类来表示一个复数。该函数的作用是某个像素是否在集合内，并返回“逃逸”的迭代。

### **线性曼德布洛特实现**

```
void sequentialMandelbrot(Mat &img, const float x1, const float y1, const float scaleX, const float scaleY)
{
    for (int i = 0; i < img.rows; i++)
    {
        for (int j = 0; j < img.cols; j++)
        {
            float x0 = j / scaleX + x1;
            float y0 = i / scaleY + y1;
            complex<float> z0(x0, y0);
            uchar value = (uchar) mandelbrotFormula(z0);
            img.ptr<uchar>(i)[j] = value;
        }
    }
}
```

在该实现中，我们线性迭代图像中的每一个像素，测试该像素是否有可能属于曼德布洛特集合。

还需做的是，通过以下代码将像素第坐标转换到曼德布洛特集空间中：

```
Mat mandelbrotImg(4800, 5400, CV_8U);
float x1 = -2.1f, x2 = 0.6f;
float y1 = -1.2f, y2 = 1.2f;
float scaleX = mandelbrotImg.cols / (x2 - x1);
float scaleY = mandelbrotImg.rows / (y2 - y1);
```

最后，我们根据以下规则确定像素的灰度值：
- 假如像素达到最大迭代次数，那么给它赋予黑色（假设该像素属于曼德布洛特集合）。
- 否则，根据逃逸的迭代次数给像素赋值，并将值缩放到灰度等级范围内。

```
int mandelbrotFormula(const complex<float> &z0, const int maxIter=500) {
    int value = mandelbrot(z0, maxIter);
    if(maxIter - value == 0)
    {
        return 0;
    }
    return cvRound(sqrt(value / (float) maxIter) * 255);
}
```

使用线性变换不足以凸显出灰度的变化。为此，这里使用平方根缩放变换（源自Jeremy D. Frens的[博客](http://www.programming-during-recess.net/2016/06/26/color-schemes-for-mandelbrot-sets/)）：$f(x)=\sqrt{\frac{x}{maxIter}\times255}$

<center>![how_to_use_OpenCV_parallel_for_sqrt_scale_transformation.png](https://docs.opencv.org/master/how_to_use_OpenCV_parallel_for_sqrt_scale_transformation.png)</center>

图中绿线代表简单的线性缩放变换，蓝线代表平方根缩放变换，可以看到低值处有更快的增长速率。

### **并行实现**

从以上串行实现中，可以发现每个像素的计算都是独立的。为了优化计算过程，我们可以并行地对多个像素进行计算，以充分利用当今处理器的多核架构。为了更方便地实现此目的，这里使用了OpenCV的[cv::parallel\_for\_](https://docs.opencv.org/master/db/de0/group__core__utils.html#gaa42ec9937b847cb52a97c613fc894c4a)框架.

```
class ParallelMandelbrot : public ParallelLoopBody
{
public:
    ParallelMandelbrot (Mat &img, const float x1, const float y1, const float scaleX, const float scaleY)
        : m_img(img), m_x1(x1), m_y1(y1), m_scaleX(scaleX), m_scaleY(scaleY)
    {
    }
    virtual void operator ()(const Range& range) const
    {
        for (int r = range.start; r < range.end; r++)
        {
            int i = r / m_img.cols;
            int j = r % m_img.cols;
            float x0 = j / m_scaleX + m_x1;
            float y0 = i / m_scaleY + m_y1;
            complex<float> z0(x0, y0);
            uchar value = (uchar) mandelbrotFormula(z0);
            m_img.ptr<uchar>(i)[j] = value;
        }
    }
    ParallelMandelbrot& operator=(const ParallelMandelbrot &) {
        return *this;
    };
private:
    Mat &m_img;
    float m_x1;
    float m_y1;
    float m_scaleX;
    float m_scaleY;
};
```

这里，首先声明了一个继承自[cv::ParallelLoopBody](https://docs.opencv.org/master/d2/d74/classcv_1_1ParallelLoopBody.html)的自定义类，并且重写了`virtual void operator ()(const cv::Range& range) const`这个函数。

`operator()`中的range代表被单独线程处理的像素的集合。任务会自动、平等地进行分配。我们需要将像素的索引坐标转换成2D的`[row,col]`坐标。并且要注意使用mat图像的引用以修改图像。

并行执行语句如下：
```
ParallelMandelbrot parallelMandelbrot(mandelbrotImg, x1, y1, scaleX, scaleY);
parallel_for_(Range(0, mandelbrotImg.rows*mandelbrotImg.cols), parallelMandelbrot);
```

