# File Input and Output using XML and YAML files（使用XML和YAML文件进行输入输出）


---

## **目标**

你能在本篇教程中找到以下问题的答案：

- 如何对文本数据进行输入输出以及OpenCV如何使用YAML或XML格式的文件。
- 如何对OpenCV其它数据类型作同样的操作。
- 如何对你自己定义的数据类型作此操作。
- 一些OpenCV数据结构如： cv::FileStorage,cv::FileNoad以及cv::FileLoadIterator的用法。

## **源码**

你能从[此处](https://github.com/opencv/opencv/tree/master/samples/cpp/tutorial_code/core/file_input_output/file_input_output.cpp)下载到以下源码，或从OpenCV库的 `samples/cpp/tutorial_code/core/file_input_output/file_input_output.cpp`路径下找到。

下面的这段代码演示了如何实现目标一节中列出的各项功能。

```
#include <opencv2/core.hpp>
#include <iostream>
#include <string>
using namespace cv;
using namespace std;
static void help(char** av)
{
    cout << endl
        << av[0] << " shows the usage of the OpenCV serialization functionality."         << endl
        << "usage: "                                                                      << endl
        <<  av[0] << " outputfile.yml.gz"                                                 << endl
        << "The output file may be either XML (xml) or YAML (yml/yaml). You can even compress it by "
        << "specifying this in its extension like xml.gz yaml.gz etc... "                  << endl
        << "With FileStorage you can serialize objects in OpenCV by using the << and >> operators" << endl
        << "For example: - create a class and have it serialized"                         << endl
        << "             - use it to read and write matrices."                            << endl;
}
class MyData
{
public:
    MyData() : A(0), X(0), id()
    {}
    explicit MyData(int) : A(97), X(CV_PI), id("mydata1234") // explicit to avoid implicit conversion
    {}
    void write(FileStorage& fs) const                        //Write serialization for this class
    {
        fs << "{" << "A" << A << "X" << X << "id" << id << "}";
    }
    void read(const FileNode& node)                          //Read serialization for this class
    {
        A = (int)node["A"];
        X = (double)node["X"];
        id = (string)node["id"];
    }
public:   // Data Members
    int A;
    double X;
    string id;
};
//These write and read functions must be defined for the serialization in FileStorage to work
static void write(FileStorage& fs, const std::string&, const MyData& x)
{
    x.write(fs);
}
static void read(const FileNode& node, MyData& x, const MyData& default_value = MyData()){
    if(node.empty())
        x = default_value;
    else
        x.read(node);
}
// This function will print our custom class to the console
static ostream& operator<<(ostream& out, const MyData& m)
{
    out << "{ id = " << m.id << ", ";
    out << "X = " << m.X << ", ";
    out << "A = " << m.A << "}";
    return out;
}
int main(int ac, char** av)
{
    if (ac != 2)
    {
        help(av);
        return 1;
    }
    string filename = av[1];
    { //write
        Mat R = Mat_<uchar>::eye(3, 3),
            T = Mat_<double>::zeros(3, 1);
        MyData m(1);
        FileStorage fs(filename, FileStorage::WRITE);
        fs << "iterationNr" << 100;
        fs << "strings" << "[";                              // text - string sequence
        fs << "image1.jpg" << "Awesomeness" << "../data/baboon.jpg";
        fs << "]";                                           // close sequence
        fs << "Mapping";                              // text - mapping
        fs << "{" << "One" << 1;
        fs <<        "Two" << 2 << "}";
        fs << "R" << R;                                      // cv::Mat
        fs << "T" << T;
        fs << "MyData" << m;                                // your own data structures
        fs.release();                                       // explicit close
        cout << "Write Done." << endl;
    }
    {//read
        cout << endl << "Reading: " << endl;
        FileStorage fs;
        fs.open(filename, FileStorage::READ);
        int itNr;
        //fs["iterationNr"] >> itNr;
        itNr = (int) fs["iterationNr"];
        cout << itNr;
        if (!fs.isOpened())
        {
            cerr << "Failed to open " << filename << endl;
            help(av);
            return 1;
        }
        FileNode n = fs["strings"];                         // Read string sequence - Get node
        if (n.type() != FileNode::SEQ)
        {
            cerr << "strings is not a sequence! FAIL" << endl;
            return 1;
        }
        FileNodeIterator it = n.begin(), it_end = n.end(); // Go through the node
        for (; it != it_end; ++it)
            cout << (string)*it << endl;
        n = fs["Mapping"];                                // Read mappings from a sequence
        cout << "Two  " << (int)(n["Two"]) << "; ";
        cout << "One  " << (int)(n["One"]) << endl << endl;
        MyData m;
        Mat R, T;
        fs["R"] >> R;                                      // Read cv::Mat
        fs["T"] >> T;
        fs["MyData"] >> m;                                 // Read your own structure_
        cout << endl
            << "R = " << R << endl;
        cout << "T = " << T << endl << endl;
        cout << "MyData = " << endl << m << endl << endl;
        //Show default behavior for non existing nodes
        cout << "Attempt to read NonExisting (should initialize the data structure with its default).";
        fs["NonExisting"] >> m;
        cout << endl << "NonExisting = " << endl << m << endl;
    }
    cout << endl
        << "Tip: Open up " << filename << " with a text editor to see the serialized data." << endl;
    return 0;
}
```

## **解释**

此处，我们只详细谈论XML和YAML格式文件的输入。文件的输出方法和输入方法基本类似。主要有两种数据结构：mapping(类似于STL中的map)和element sequence(类似于STL中的vector)。两者的不同之处在于，map中的每个元素都有唯一的名字从而可以直接访问每个元素，而对于sequence来说则需要遍历sequence来查找指定的元素。

1. ***XML/YAML 文件的打开和关闭。*** 在往文件中写入数据之前， 你需要先打开一个文件，并在写入完成后关闭该文件。OpenCV中的XML/YAML的数据结构是[cv::FileStorage](http://docs.opencv.org/master/da/d56/classcv_1_1FileStorage.html)。
   为了创建一个和硬盘中的某个文件绑定在一起的该数据结构，可以使用构造函数或[open()](http://docs.opencv.org/master/d6/dee/group__hdf5.html#gac08f6eafa0f92e1c864044e27bfc7bad)函数：
   ```
   string filename = "I.xml";
   FileStorage fs(filename, FileStorage::WRITE);
   //...
   fs.open(filename, FileStorage::READ);
   ```
   不管你使用上面两种方法的哪一种，第二个参数都是一个常量用以指定你能够对文件进行操作的类型：WRITE,READ,或者APPEND。文件的扩展名决定了输出文件的格式。 假如你使用类似于"*.xml.gz"的扩展名，输出文件还能够被压缩。
   当cv::FileStorage对象被销毁时，文件会自动关闭，你也可以使用释放函数来显示地进行该操作。
   ```
fs.release(); //显示关闭文件
   ```
   
2. ***文本和数字的输入和输出。*** 该数据结构使用和STL库相同的<<输出操作符。为了输出数据，我们需要先指定输出数据的名字。在这里，简单起见，我们仅仅使用变量的英文名:
   ```
   fs << "iterationNr" << 100;
   ```
   读入数据仅包括简单的寻址访问（通过[]操作符）和类型转换操作或者使用>>符号进行读取：
   ```
   int itNr;
   fs("iterationNr")>>itNr;
   itNr = (int)fs["iterationNr"];
   ```
   
3. ***OpenCV数据结构的输入和输出。*** 对这些数据结构的操作和上面对基本的c++类型的操作类似：
   ```
   Mat R = Mat_<uchar >::eye  (3, 3),
   T = Mat_<double>::zeros(3, 1);
   fs << "R" << R;      // 写入cv::Mat
   fs << "T" << T;
   fs["R"] >> R;        // 读取cv::Mat
   fs["T"] >> T;
   ```
   
4. ***vectors(arrays)和相关maps的输入和输出。*** 正如前面所提到的，我们也能够输出maps和sequences(array,vector)。首先，我们还是得先写变量名，然后再指定输出是sequences还是maps。
   对于sequence来说，在第一个元素前输入"["符号并在最后一个元素后输入"]"符号:
   ```
   fs << "strings" << "[";      // text -     string sequence
   fs << "image1.jpg" << "Awesomeness" << "baboon.jpg";
   fs << "]";                   // close      sequence
   ```
   对于map来说，前面的变量名还是需要的，但是改用"{"和"}"符号来作为定界符:
   ```
   fs << "Mapping";             // text - mapping
   fs << "{" << "One" << 1;
   fs <<        "Two" << 2 << "}";
   ```
   为了读取数据，我们需要使用[cv::FileNode](http://docs.opencv.org/master/de/dd9/classcv_1_1FileNode.html)和[cv::FileNodeIterator](http://docs.opencv.org/master/d7/d4e/classcv_1_1FileNodeIterator.html)数据结构。cv::FileStorage类的[]操作会返回一个cv::FileNode数据类型。假如结点是连续的，我们可以使用cv::FileNodeIterator迭代遍历元素：
   ```
   FileNode n = fs["strings"]; // Read string sequence - Get node
   if (n.type() != FileNode::SEQ)
   {
       cerr << "strings is not a sequence! FAIL" << endl;
       return 1;
   }
   FileNodeIterator it = n.begin(), it_end = n.end(); // Go through    the node
   for (; it != it_end; ++it)
   cout << (string)*it << endl;
   ```
   对于maps，你可以使用[]操作来访问给定的元素（或使用>>操作）:
   ```
   n = fs["Mapping"];   // Read mappings from a sequence
   cout << "Two  " << (int)(n["Two"]) << "; ";
   cout << "One  " << (int)(n["One"]) << endl << endl;
   ```
   
5. ***对自定义的数据结构进行输入输出。*** 假设你自己定义了如下所示的数据结构：
   ```
   class MyData
   {
    public:
       MyData() : A(0), X(0), id() {}
    public:   // Data Members
       int A;
       double X;
       string id;
   };
   ```
   可以使用OpenCV的XML/YAML I/O接口（和OpenCV的数据结构类似）在你自定义的类内部和外部添加一个专门的输入和输出函数。在类内部添加函数如下所示：
   ```
   void write(FileStorage& fs) const     //Write serialization for this class
   {
      fs << "{" << "A" << A << "X" << X << "id" << id << "}";
   }
    void read(const FileNode& node)    //Read serialization for this class
   {
      A = (int)node["A"];
      X = (double)node["X"];
      id = (string)node["id"];
}
   ```
   然后你需要在类外部添加以下函数定义：
   ```
   void write(FileStorage& fs, const std::string&, const MyData& x)
   {
      x.write(fs);
   }
   void read(const FileNode& node, MyData& x, const MyData& default_value = MyData())
   {
      if(node.empty())
       x = default_value;
      else
       x.read(node);
   }
   ```
   从上，你可以发现在read部分，我们定义了假如用户试图读取不存在的node时进行的操作。对此，我们仅仅返回默认的初始值，一个更啰嗦的方法是返回访问对象ID的负值。
   一旦添加了以上四个函数，你就可以使用>>和<<操作符进行输入输出了:
   ```
   MyData m(1);
   fs << "MyData" << m;        // 输出你自己的数据结构
   fs["MyData"] >> m;          // 读取你自己的数据结构
   ```
   或者尝试读取一个不存在的数据：
   ```
   fs["NonExisting"] >> m;   // Do not add a fs << "NonExisting" << m command for this to work
   cout << endl << "NonExisting = " << endl << m << endl;
   ```
   
## **结果**

以下结果主要是我们定义的数据，可以在控制台看到：

```
Write Done.
Reading:
100image1.jpg
Awesomeness
baboon.jpg
Two  2; One  1
R = [1, 0, 0;
  0, 1, 0;
  0, 0, 1]
T = [0; 0; 0]
MyData =
{ id = mydata1234, X = 3.14159, A = 97}
Attempt to read NonExisting (should initialize the data structure with its default).
NonExisting =
{ id = , X = 0, A = 0}
Tip: Open up output.xml with a text editor to see the serialized data.
```

尽管如此，你可以在输出的xml文件中看到更有趣的东西：

```
<?xml version="1.0"?>
<opencv_storage>
<iterationNr>100</iterationNr>
<strings>
  image1.jpg Awesomeness baboon.jpg</strings>
<Mapping>
  <One>1</One>
  <Two>2</Two></Mapping>
<R type_id="opencv-matrix">
  <rows>3</rows>
  <cols>3</cols>
  <dt>u</dt>
  <data>
    1 0 0 0 1 0 0 0 1</data></R>
<T type_id="opencv-matrix">
  <rows>3</rows>
  <cols>1</cols>
  <dt>d</dt>
  <data>
    0. 0. 0.</data></T>
<MyData>
  <A>97</A>
  <X>3.1415926535897931e+000</X>
  <id>mydata1234</id></MyData>
</opencv_storage>
```

或者YAML中的:

```
%YAML:1.0
iterationNr: 100
strings:
   - "image1.jpg"
   - Awesomeness
   - "baboon.jpg"
Mapping:
   One: 1
   Two: 2
R: !!opencv-matrix
   rows: 3
   cols: 3
   dt: u
   data: [ 1, 0, 0, 0, 1, 0, 0, 0, 1 ]
T: !!opencv-matrix
   rows: 3
   cols: 1
   dt: d
   data: [ 0., 0., 0. ]
MyData:
   A: 97
   X: 3.1415926535897931e+000
   id: mydata1234
```

你可以在[YouTube](https://www.youtube.com/watch?v=A4yqVnByMMM)上观看这一例程运行的情况。
   