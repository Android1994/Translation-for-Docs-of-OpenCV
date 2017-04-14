# Random generator and text with OpenCV

---

## **目标**

通过这篇教程你将学到：

- 使用随机数生成器(cv::RNG)以及如何从一个均匀分布里随机抽取数字
- 使用cv::putText函数在OpenCV的窗口上显示文本

## **代码**

- 在之前的教程中（Basic Drawing），我们通过一些给定的参数，比如：坐标(cv::Point的形式)、颜色、线条宽度等画出了各种不同的几何形状。很明显可以看出，我们给每个参数都指定了确切的值。
- 在这篇教程中，我们想要为这些参数指定一些随机的值。同时，我们想在一幅图上画大量不同的几何形状，所以使用了循环来实现随机初始化参数。
- 你可以在OpenCV的样例代码文件夹或[此处](http://code.opencv.org/projects/opencv/repository/revisions/master/raw/samples/cpp/tutorial_code/core/Matrix/Drawing_2.cpp)找到此源码。

## **解释**

1. 我们首先来看看main函数吧，可以看到，第一件要做的事就是创建随机数生成器对象(RNG):

   `RNG rng(0xFFFFFFFF);`
   
   RNG实现了一个随机数生成器，此例中，rng是以0xFFFFFFFF初始化的一个RNG元素。

2. 然后，我们创建一个零矩阵（意味着这是一幅黑色的图像），并指定矩阵的高和宽以及数据类型：

   ```
   Mat image = Mat::zeros(windows_height,windows_width,CV_8UC3);
   imshow(window_name,image);
   ```

3. 然后，我们要开始画一些疯狂的东西了。这部分代码主要可以被分为8个部分，每部分是一个函数：

   ```
    c = Drawing_Random_Lines(image, window_name, rng);
    if( c != 0 ) return 0;
    c = Drawing_Random_Rectangles(image, window_name, rng);
    if( c != 0 ) return 0;
    c = Drawing_Random_Ellipses( image, window_name, rng );
    if( c != 0 ) return 0;
    c = Drawing_Random_Polylines( image, window_name, rng );
    if( c != 0 ) return 0;
    c = Drawing_Random_Filled_Polygons( image, window_name, rng );
    if( c != 0 ) return 0;
    c = Drawing_Random_Circles( image, window_name, rng );
    if( c != 0 ) return 0;
    c = Displaying_Random_Text( image, window_name, rng );
    if( c != 0 ) return 0;
    c = Displaying_Big_End( image, window_name, rng );
   ```
   
   所有这些函数都遵循相同的模式，所以我们只需分析一组即可。
 
4. 查看函数Drawing_Random_Lines:

   ```
   int Drawing_Random_Lines(Mat image,char* window_name,RNG rng){
       int LineType = 8;
       Point pt1,pt2;
       
       for(int i=0;i<Number;i++){
          pt1.x=rng.uniform(x_1,x_2);
          pt1.y=rng.uniform(y_1,y_2);
          pt2.x=rng.uniform(x_1,x_2);
          pt2.y=rng.uniform(y_1,y_2);
          
          line(image,pt1,pt2,randomColor(rng),rng.uniform(1,10),LineType);
          imshow(window_name,image);
          if(waitKey(Delay)>=0)
            return -1;
       }
       return 0;
   } 
   ```
   
  我们可以看出：
  - 循环将会执行NUMBER次，所以会有NUMBER条直线被画出。
  - 直线的两个端点由pt1和pt2给出，对于pt1我们可以看到：
    ```  
    pt1.x=rng.uniform(x_1,x_2);
    pt1.y=rng.uniform(y_1,y_2);
    ```
     - 我们知道rng是随机数生成器对象，上面的代码中我们调用了函数rng.uniform(a,b)。这样可以生成满足参数是a,b的均匀分布的随机数。
     - 从上面的解释我们可以看出，每条线的端点pt1和pt2将是随机的，所以直线的位置是不可预测的，由此可产生惊艳的视觉效果。
     - 我们还可以发现cv::line的参数中，控制颜色的参数我们是这样指定的：
       `randomColor(rng)`
       让我们看看具体的函数实现：
       ```
       static Scalar randomColor( RNG& rng )
       {
       int icolor = (unsigned) rng;
       return Scalar( icolor&255, (icolor>>8)&255, (icolor>>16)&255 );
       }
        ```
       可以看到，返回值是3个元素被随机初始化的Scalar向量，所以直线的颜色是随机的。
       
5. 以上解释也适用其它诸如画圆、画椭圆、画多边形等函数。其中像圆心、顶点这样的参数都是随机生成的。

6. 最后，我们一起再来看看Display_Random_Text和Displaying_Big_End这两个函数，它们都有些有趣的特性：

7. Dispay_Random_Text:

   ```
   int Displaying_Random_Text( Mat image, char* window_name, RNG rng )
   {
   int lineType = 8;
   for ( int i = 1; i < NUMBER; i++ )
   {
    Point org;
    org.x = rng.uniform(x_1, x_2);
    org.y = rng.uniform(y_1, y_2);
    putText( image, "Testing text rendering", org, rng.uniform(0,8),
             rng.uniform(0,100)*0.05+0.1, randomColor(rng), rng.uniform(1, 10), lineType);
    imshow( window_name, image );
    if( waitKey(DELAY) >= 0 )
      { return -1; }
   }
   return 0;
   }
   ```
   
   是不是一切都很熟悉，除了下面这条语句：
   
   `putText( image, "Testing text rendering", org, rng.uniform(0,8),rng.uniform(0,100)*0.05+0.1, randomColor(rng), rng.uniform(1, 10), lineType);`
   
   所以，cv::putText这个函数究竟做了什么？在上面的例子中：
   
   - 在图像image上显示了“Testing text rendering”这些文字
   - 文本的左下角便是点org
   - 字体类型是[0,8]间的随机值
   - 字体的大小由rng.uniform(0,100)*0.05+0.1指定（即范围是[0.1,5.1]）
   - 字体颜色是随机的（由randomColor(rng)指定）
   - 字体线条的粗细范围是[1,10]，由rng.uniform(1,10)指定
   
   最终，我们能得到NUMBER个文本（和其他图形一样），随机散布在图像不同的位置。

8. Displaying _Big_End

   ```
   int Displaying_Big_End( Mat image, char* window_name, RNG rng )
   {
   Size textsize = getTextSize("OpenCV forever!", FONT_HERSHEY_COMPLEX, 3, 5, 0);
   Point org((window_width - textsize.width)/2, (window_height - textsize.height)/2);
   int lineType = 8;
   Mat image2;
   for( int i = 0; i < 255; i += 2 )
   {
    image2 = image - Scalar::all(i);
    putText( image2, "OpenCV forever!", org, FONT_HERSHEY_COMPLEX, 3,
           Scalar(i, i, 255), 5, lineType );
    imshow( window_name, image2 );
    if( waitKey(DELAY) >= 0 )
      { return -1; }
   }
   return 0;
   }
   ```
   
   除了函数getTextSize(获得text变量的大小),我们还能在for循环中看到下面这一新操作：
   
   `image2 = image - Scalar::all(i);`
   
   所以，image2由image与Scalar::all(i)作减法得到，事实上，image2的每个像素由image每个像素减去i得到（由于每个像素有R、G、B三个值，所以每个通道的值都会受影响）
   
   还要记住的是，这里的减法操作总会执行饱和运算，即计算结果总会在允许的取值范围之内（没有负值，都在0-255的范围）。
   
## **实验结果**

正如你在代码部分看到的，程序将会依次执行不同的绘图函数，能够产生：

1. 首先，NUMBER条随机的直线将会出现在屏幕上：
<center>![Drawing_2_Tutorial_Result_0](http://docs.opencv.org/master/Drawing_2_Tutorial_Result_0.jpg)</center>

2. 然后是矩形。

3. 接着是各种椭圆：
<center>![Drawing_2_Tutorial_Result_2](http://docs.opencv.org/master/Drawing_2_Tutorial_Result_2.jpg)</center>

4. 然后是3段线。
<center>![Drawing_2_Tutorial_Result_3](http://docs.opencv.org/master/Drawing_2_Tutorial_Result_3.jpg)</center>

5. 填充多边形（此例中是三角形）紧随其后。

6. 最后的几何形状：各种圆！
<center>![Drawing_2_Tutorial_Result_5](http://docs.opencv.org/master/Drawing_2_Tutorial_Result_5.jpg)</center>

7. 快结束了，各种字体、颜色、大小的"Testing Text Redering"出现了。

8. 最后“OpenCV forever”压轴登场（同时也是真理（^_^））：
<center>![Drawing_2_Tutorial_Result_big](http://docs.opencv.org/master/Drawing_2_Tutorial_Result_big.jpg)</center>




