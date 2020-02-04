---
title: Gaussian Filter
date: 2019-07-12 16:09:24
tags:
 - CV 
categories: 
 - Algorithm
---

高斯函数在学术领域运用的非常广泛。 写工程产品的时候，经常用它来去除图片或者视频的噪音，平滑图片, Blur处理。我们今天来看看高斯滤波, Gaussian Filter。 
**1D的高斯函数**
一维的高斯函数（或者叫正态分布）方程跟图形如下: 
$$G(x) = \frac{1}{\sqrt{2\pi\sigma^2}} e^{-\frac{x^2}{2\sigma^2}}$$
![image.png](/img/Gaussian-Filter/gaussian.png)

$\mu$是均值；$\sigma$ 是标准方差。它有个重要特点是 -$\sigma$ 到+$\sigma$ 之间的G(x)与x轴围成的面积占全部面积的68.2%.  -2$\sigma$ 到+2$\sigma$之间的面积占95%。-3$\sigma$ 到+3$\sigma$之间的面积占99.7%。
如果我们给-3$\sigma$ 到+3$\sigma$区间, 它几乎包括了所有可能的点。这个特性对Filter kernel的生成很重要。


**2D的高斯函数**
$$G(x, y) = \frac{1}{\sqrt{2\pi\sigma^2}} e^{-\frac{x^2 + y^2}{2\sigma^2}}$$

![image.png](/img/Gaussian-Filter/2d-gaussian.png)


**所谓高斯滤波操作，其实就是用高斯函数对image做卷积计算**。但一般图像在计算机中一般是离散的3D矩阵，而高斯函数是连续函数，所以我们要从连续高斯函数中采样生成离散的2D矩阵，即Gaussian Filter Kernel。 我们可以控制Kernal的size，让它的点都落在-3$\sigma$ 到+3$\sigma$区间内。
### 生成高斯kernel 

```c++
// Function to create Gaussian filter; sigma is standard deviation
Matrix getGaussian(int height, int width, double sigma)
{
    Matrix kernel(height, Array(width));
    // sum is for normalization 
    double sum=0.0;
    int i,j;
    
   // generating the kernel 
    for (i=0 ; i<height ; i++) {
        for (j=0 ; j<width ; j++) {
            // using gaussian function to generate gaussian filter 
            kernel[i][j] = exp(-(i*i+j*j)/(2*sigma*sigma))/(2*M_PI*sigma*sigma);
            sum += kernel[i][j];
        }
    }
   
   // normalising the Kernel 
    for (i=0 ; i<height ; i++) {
        for (j=0 ; j<width ; j++) {
            kernel[i][j] /= sum;
        }
    }

    return kernel;
}
```
[代码来源](https://gist.github.com/OmarAflak/aca9d0dc8d583ff5a5dc16ca5cdda86a)
比如，我们用高斯函数生成了一个5x5， $\sigma$是1的高斯核2D矩阵:
![image.png](/img/Gaussian-Filter/gaussian_matrix.png)
它有几个特点： 
1. 最中间的值最大，值向周围递减
2. $\sigma$越大，高斯函数的峰越宽，临接的数值差越大

### 对图片应用高斯Filter
对某个像素点image[i][j]，Fitler对原图对应的像素点做点乘，相加。 生成新的值。
![https://www.youtube.com/watch?v=C_zFhWdM4ic](/img/Gaussian-Filter/convoltion.gif)
[材料来源](https://www.youtube.com/watch?v=C_zFhWdM4ic)
```c++
Image applyFilter(Image &image, Matrix &filter){
    assert(image.size()==3 && filter.size()!=0);

    int height = image[0].size();
    int width = image[0][0].size();
    int filterHeight = filter.size();
    int filterWidth = filter[0].size();
    int newImageHeight = height-filterHeight+1;
    int newImageWidth = width-filterWidth+1;
    int d,i,j,h,w;

    Image newImage(3, Matrix(newImageHeight, Array(newImageWidth)));
    
   // iter the image pixel
    for (d=0 ; d<3 ; d++) {
        for (i=0 ; i<newImageHeight ; i++) {
            for (j=0 ; j<newImageWidth ; j++) {
                // using filter convolute the image matrix
                for (h=i ; h<i+filterHeight ; h++) {
                    for (w=j ; w<j+filterWidth ; w++) {
                        newImage[d][i][j] += filter[h-i][w-j]*image[d][h][w];
                    }
                }
            }
        }
    }

    return newImage;
}
```
如下图，图片被平滑处理了。
![image.png](/img/Gaussian-Filter/blur_image.png)

#### More: 
[https://gist.github.com/OmarAflak/aca9d0dc8d583ff5a5dc16ca5cdda86a](https://gist.github.com/OmarAflak/aca9d0dc8d583ff5a5dc16ca5cdda86a)

