# Eigen使用模板（ubuntu系统中）
---
```C++
#include <iostream>
#include <eigen3/Eigen/Dense>
 
int main()
{
    Eigen::MatrixXf matrix1(3,3); //定义了矩阵的大小，但是没有初始化。
    Eigen::MatrixXf matrix2(3,3);
    Eigen::Vector3f vector1;      //定义了一个3个元素的列向量

    matrix1 = Eigen::MatrixXf::Ones(3,3); //对矩阵进行初始化。
    matrix2 <<  1,2,3,
                2,1,4,
                3,4,1;
    vector1 << 6,6,6;

    std::cout << "------ matrix1 ------" << std::endl << matrix1 << std::endl;
    std::cout << "------ matrix2 ------" << std::endl << matrix2 << std::endl;
    std::cout << "------ matrix1 + matrix2 ------" << std::endl << matrix1 + matrix2 << std::endl;
    std::cout << "------ matrix1 * matrix2 ------" << std::endl << matrix1 * matrix2 << std::endl;
    std::cout << "------ matrix2 * vector1 ------" << std::endl << matrix2 * vector1 << std::endl;
    std::cout << "------ matrix2_tra ------" << std::endl << matrix2.trace() << std::endl;
    std::cout << "------ matrix2_ni ------" << std::endl << matrix2.inverse() << std::endl;
    
}
```
