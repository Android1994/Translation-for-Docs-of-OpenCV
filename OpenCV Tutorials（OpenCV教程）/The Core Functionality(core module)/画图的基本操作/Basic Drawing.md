# Basic Drawing（画图的基本操作）

---

## **目标**

通过这篇教程，你能学会：

- 使用cv::Point来定义2维点变量
- cv::Scalar的用处和用法
- 使用cv::line来画出直线
- 使用cv::ellipse画出椭圆
- 使用cv::rectangle画出矩形
- 使用cv::circle画出圆形
- 使用cv::fillPoly画出填充多边形

## **原理**

这篇教程使用了大量的cv::Point和cv::Scalar这两个结构体：

### **Point**

一个二维的点是通过指定x轴和y轴的坐标进行定义的：

    Point pt;
    pt.x = 10;
    pt. = 8;
或者

    Point pt =  Point(10, 8);
    
### **Scalar**

- 代表一个有四个元素的向量，Scalar在OpenCV中常用来传递像素值。
- 这篇教程使用Scalar来代表BGR的值（3个参数），最后一个参数如果不用的话是不需要指定的。
- 声明一种颜色的例子如下：
  `Scalar(a,b,c)`
  然后我们这样定义一种BGR颜色：Blue=a,Green=b,Red=c。

## **代码**

 1. 下面的代码你可以在OpenCV的样例文件夹中找到，或者可以从[这里](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/Matrix/Drawing_1.cpp)找到
```
#include <opencv2/core.hpp>
#include <opencv2/imgproc.hpp>
#include <opencv2/highgui.hpp>

#define w 400
using namespace cv;
        
void MyEllipse( Mat img, double angle );
void MyFilledCircle( Mat img, Point center );
void MyPolygon( Mat img );
void MyLine( Mat img, Point start, Point end );

int main( void ){
  char atom_window[] = "Drawing 1: Atom";
  char rook_window[] = "Drawing 2: Rook";
  Mat atom_image = Mat::zeros( w, w, CV_8UC3 );
  Mat rook_image = Mat::zeros( w, w, CV_8UC3 );
  MyEllipse( atom_image, 90 );
  MyEllipse( atom_image, 0 );
  MyEllipse( atom_image, 45 );
  MyEllipse( atom_image, -45 );
  MyFilledCircle( atom_image, Point( w/2, w/2) );
  MyPolygon( rook_image );
  rectangle( rook_image,
         Point( 0, 7*w/8 ),
         Point( w, w),
         Scalar( 0, 255, 255 ),
         FILLED,
         LINE_8 );
  MyLine( rook_image, Point( 0, 15*w/16 ), Point( w, 15*w/16 ) );
  MyLine( rook_image, Point( w/4, 7*w/8 ), Point( w/4, w ) );
  MyLine( rook_image, Point( w/2, 7*w/8 ), Point( w/2, w ) );
  MyLine( rook_image, Point( 3*w/4, 7*w/8 ), Point( 3*w/4, w ) );
  imshow( atom_window, atom_image );
  moveWindow( atom_window, 0, 200 );
  imshow( rook_window, rook_image );
  moveWindow( rook_window, w, 200 );
  waitKey( 0 );
  return(0);
}

void MyEllipse( Mat img, double angle )
{
  int thickness = 2;
  int lineType = 8;
  ellipse( img,
       Point( w/2, w/2 ),
       Size( w/4, w/16 ),
       angle,
       0,
       360,
       Scalar( 255, 0, 0 ),
       thickness,
       lineType );
}

void MyFilledCircle( Mat img, Point center )
{
  circle( img,
      center,
      w/32,
      Scalar( 0, 0, 255 ),
      FILLED,
      LINE_8 );
}

void MyPolygon( Mat img )
{
  int lineType = LINE_8;
  Point rook_points[1][20];
  rook_points[0][0]  = Point(    w/4,   7*w/8 );
  rook_points[0][1]  = Point(  3*w/4,   7*w/8 );
  rook_points[0][2]  = Point(  3*w/4,  13*w/16 );
  rook_points[0][3]  = Point( 11*w/16, 13*w/16 );
  rook_points[0][4]  = Point( 19*w/32,  3*w/8 );
  rook_points[0][5]  = Point(  3*w/4,   3*w/8 );
  rook_points[0][6]  = Point(  3*w/4,     w/8 );
  rook_points[0][7]  = Point( 26*w/40,    w/8 );
  rook_points[0][8]  = Point( 26*w/40,    w/4 );
  rook_points[0][9]  = Point( 22*w/40,    w/4 );
  rook_points[0][10] = Point( 22*w/40,    w/8 );
  rook_points[0][11] = Point( 18*w/40,    w/8 );
  rook_points[0][12] = Point( 18*w/40,    w/4 );
  rook_points[0][13] = Point( 14*w/40,    w/4 );
  rook_points[0][14] = Point( 14*w/40,    w/8 );
  rook_points[0][15] = Point(    w/4,     w/8 );
  rook_points[0][16] = Point(    w/4,   3*w/8 );
  rook_points[0][17] = Point( 13*w/32,  3*w/8 );
  rook_points[0][18] = Point(  5*w/16, 13*w/16 );
  rook_points[0][19] = Point(    w/4,  13*w/16 );
  const Point* ppt[1] = { rook_points[0] };
  int npt[] = { 20 };
  fillPoly( img,
        ppt,
        npt,
        1,
        Scalar( 255, 255, 255 ),
        lineType );
}

void MyLine( Mat img, Point start, Point end )
{
  int thickness = 2;
  int lineType = LINE_8;
  line( img,
    start,
    end,
    Scalar( 0, 0, 0 ),
    thickness,
    lineType );
}
```    
        
## **解释**

1. 由于我们想要画两个图形（原子图案和国际象棋中车的图案），所以需要创建两个Mat和窗口来进行显示。
    ```
    char atom_window[] = "Drawing 1: Atom";
    char rook_window[] = "Drawing 2: Rook";
    
    Mat atom_image = Mat::zeros( w, w, CV_8UC3 );
    Mat rook_image = Mat::zeros( w, w, CV_8UC3 );
    ```
    
2. 我们为不同的几何图形创建了不同的函数。比如，为了画原子图案我们使用了两种函数MyEllipse和MyFilledCircle:
    ```
    MyEllipse( atom_image, 90 );
    MyEllipse( atom_image, 0 );
    MyEllipse( atom_image, 45 );
    MyEllipse( atom_image, -45 );`
    
    MyFilledCircle( atom_image, Point( w/2, w/2) );
    ```
    
3. 使用MyLine、rectangle和MyPolygon这三个函数画车的图案：
    ```
    MyPolygon( rook_image );
    
    rectangle( rook_image,
         Point( 0, 7*w/8 ),
         Point( w, w),
         Scalar( 0, 255, 255 ),
         FILLED,
         LINE_8 );
         
    MyLine( rook_image, Point( 0, 15*w/16 ), Point( w, 15*w/16 ) );
    MyLine( rook_image, Point( w/4, 7*w/8 ), Point( w/4, w ) );
    MyLine( rook_image, Point( w/2, 7*w/8 ), Point( w/2, w ) );
    MyLine( rook_image, Point( 3*w/4, 7*w/8 ), Point( 3*w/4, w ) );
    ```
  
4. 让我们看看每个函数是怎样实现的吧：
    
    - MyLine
    ```
      void MyLine( Mat img, Point start, Point end )
    {
      int thickness = 2;
      int lineType = LINE_8;
      line( img,
        start,
        end,
        Scalar( 0, 0, 0 ),
        thickness,
        lineType );
    }
    ```
        我们可以看到，MyLine调用了函数cv::line，cv::line有以下功能：
        - 以start为起点，end为终点画一条直线
        - 直线会显示在图像img上
        - 直线的颜色由Scalar(0,0,0)决定，在此处即为黑色
        - 直线宽度被设为thickness（此例中为2）
        - 直线类型为8邻接的（lineType = 8）

    - MyEllipse
    ```
      void MyEllipse( Mat img, double angle )
    {
      int thickness = 2;
      int lineType = 8;
      ellipse( img,
           Point( w/2, w/2 ),
           Size( w/4, w/16 ),
           angle,
           0,
           360,
           Scalar( 255, 0, 0 ),
           thickness,
           lineType );
    }
    ```
       通过上面的代码我们可以发现是函数cv::ellipse实现了画椭圆的功能：
        - 椭圆会显示在图像img上
        - 椭圆中心处于点(w/2,w/2)上，并被大小是(w/4,w/16)的包围盒围住（轴的长度）
        - 椭圆旋转角度是angle
        - 圆弧起始角和终结角的角度分别是0°和360°
        - 椭圆颜色是Scalar(255,0,0)，即蓝色
        - 弧线宽度是2

    - MyFilledCircle
      ```
      void MyFilledCircle( Mat img, Point center )
        {
          circle( img,
              center,
              w/32,
              Scalar( 0, 0, 255 ),
              FILLED,
              LINE_8 );
        }
       ```
        和ellipse函数类似，circle函数有以下参数：
        
        - 显示所画圆形的图像(img)
        - 圆的圆心center
        - 圆的半径：w/32
        - 圆的颜色：Scalar(0,0,255)，即红色
        - 由于thickness设为-1，圆会被填充
           
    - MyPolygon
        ```
        void MyPolygon( Mat img )
        {
          int lineType = LINE_8;
          
          Point rook_points[1][20];
          rook_points[0][0]  = Point(    w/4,   7*w/8 );
          rook_points[0][1]  = Point(  3*w/4,   7*w/8 );
          rook_points[0][2]  = Point(  3*w/4,  13*w/16 );
          rook_points[0][3]  = Point( 11*w/16, 13*w/16 );
          rook_points[0][4]  = Point( 19*w/32,  3*w/8 );
          rook_points[0][5]  = Point(  3*w/4,   3*w/8 );
          rook_points[0][6]  = Point(  3*w/4,     w/8 );
          rook_points[0][7]  = Point( 26*w/40,    w/8 );
          rook_points[0][8]  = Point( 26*w/40,    w/4 );
          rook_points[0][9]  = Point( 22*w/40,    w/4 );
          rook_points[0][10] = Point( 22*w/40,    w/8 );
          rook_points[0][11] = Point( 18*w/40,    w/8 );
          rook_points[0][12] = Point( 18*w/40,    w/4 );
          rook_points[0][13] = Point( 14*w/40,    w/4 );
          rook_points[0][14] = Point( 14*w/40,    w/8 );
          rook_points[0][15] = Point(    w/4,     w/8 );
          rook_points[0][16] = Point(    w/4,   3*w/8 );
          rook_points[0][17] = Point( 13*w/32,  3*w/8 );
          rook_points[0][18] = Point(  5*w/16, 13*w/16 );
          rook_points[0][19] = Point(    w/4,  13*w/16 );
          
          const Point* ppt[1] = { rook_points[0] };
          int npt[] = { 20 };
          
          fillPoly( img,
                ppt,
                npt,
                1,
                Scalar( 255, 255, 255 ),
                lineType );
        }
        ``` 
        我们使用cv::polygon函数来画一个填充的多边形，有以下几点需要注意：
        
        - 多边形会显示在图像img上
        - 多边形的顶点是点集ppt
        - 顶点的总数是npt
        - 多边形的数量是1
        - 多边形的颜色是Scalar(255,255,255)，即白色
        
    - rectangle
      ```
       rectangle( rook_image,
         Point( 0, 7*w/8 ),
         Point( w, w),
         Scalar( 0, 255, 255 ),
         FILLED,
         LINE_8 );
      ```
      最后是cv::rectangle函数：
      
        - 矩形会显示在图像rook_image上
        - 两个矩形的对角顶点是Point(0,7*w/8)和Point(w,w)
        - 矩形的颜色是Scalar(0,255,255)，即黄色
        - 由于thickness设为FILLED（-1），所以矩形会被填充
          
## **结果**
 编译并运行程序的结果如下：
 
 <center>![Drawing_1_Tutorial_Result_0](http://docs.opencv.org/master/Drawing_1_Tutorial_Result_0.png)</center>
        
    
        
