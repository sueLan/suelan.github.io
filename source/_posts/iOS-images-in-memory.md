---
title: iOS images in memory
date: 2020-05-03 08:31:45
categories: 
  - iOS 
  - Debug 
tags:
  - Memory 
  - Image
---

Memory is a finite and shared resources in mobile system, while image 
Memory use is related to the dimensions of the images, not the file size.

## The memory use of an image using sRGB space

Take the following image as an instance, its file size is 590KB, with dimension 2048 x 1536 pixel.
![83dca9cf.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/83dca9cf.png)


#### 4 byetes for each pixel in RGBA 

By this [article](https://www.objc.io/issues/3-views/moving-pixels-onto-the-screen/), each pixel in sRGB image needs 32bit, 4 bytes, in memory when it's decoded. Because in sRGB, there are Red, Green, Blue 3 channels and Alpha. The range of each channel value is from 0 to 255, which needs 8 bits to represent the value. 

```
  A   R   G   B   A   R   G   B   A   R   G   B  
 | pixel 0       | pixel 1       | pixel 2   
  0  233  2  100  4  155 255  7   8   9   10  11 
```
  

#### More memory usage when decoding a image 

By this [session](https://developer.apple.com/videos/play/wwdc2018/219), a image have load -> decode -> render these 3 phases.  

![64aaa3cd.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/ed4e1c96.png)



We only need 590KB to load a image, while we need 
`2048 pixels x 1536 pixels x 4 bytes per pixel` = 10MB when decoding 
  
 


## Image Rendering Format 

### SRGB
- 4 bytes per pixel 
- full color images 
- most common used 
![a1bf604b.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/a1bf604b.png)

### Wide format 

- 8 bytes per pixel 
- super accurate colors. Because they use 8bytes, 16 bits, for each channel. In the meantime, double the image size.  
- Only useful with wide color display. We don't want to use it when we don't need to. 
- Wide color capture cameras since iPhone 7
  

![75a1a01e.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/75a1a01e.png)

### Luminance and alpha 8 format

This image only holds grayscale value. And the image size is smaller. 

- 2 bytes per pixel 
- Single-color iamges and alpha
- Most used in Metal shaders, not very common. 
![6383b177.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/6383b177.png)
### Alpha 8 Format 

- 1 byte per pixel 
- Userful for monochrome images because it uses less memory. 
  - masks 
  - Emoji-free text 
- 75% smaller than SRGB

![753920b5.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/753920b5.png)
We can also change `image.tintColor` in this image without changing its format.  

## How do I pick the right format?

`UIGraphicsBeginImageContextWithOptions` always uses SRGB rendering-format, which use 4 bytes per pixel. 

while `UIGraphicsImageRenderer`, which introduced in iOS 10 will automatically pick the best graphic format in iOS12. It means, you will save 75% of memory by replacing `UIGraphicsBeginImageContextWithOptions` with  `UIGraphicsImageRenderer`


**Do** ✅
![2f0229d3.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/2f0229d3.png)
## Use ImageIO to downsample images

`UIImage` is expensive for sizing and to resizing
- Will decompress original image into memory 
- Internal coordinate space transforms are expensive

**Don't** ❌

 
![c25d45de.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/c25d45de.png)

Use ImageIO
- ImageIO can read image sizes and metadata information without dirtying memory.

- ImageIO can resize images at cost of resized image only.
**Do** ✅
![922ac245.png](28bfb5ac-c3fa-4575-a1b1-72fcbacc9069/922ac245.png)

The bad is that you have to specify some options. 

## Resize image effectively

This is a function of resizing image without reading it into the memory, using ImageIO.  

```swfit
func resize(url: NSURL, maxPixelSize: Int) -> CGImage? {
    let imgSource = CGImageSourceCreateWithURL(url, nil)
    guard let imageSource = imgSource else {
        return nil
    }

    var scaledImage: CGImage?
    let options: [NSString: Any] = [
            // The maximum width and height in pixels of a thumbnail.
            kCGImageSourceThumbnailMaxPixelSize: maxPixelSize,
            kCGImageSourceCreateThumbnailFromImageAlways: true,
            // Should include kCGImageSourceCreateThumbnailWithTransform: true in the options dictionary. Otherwise, the image result will appear rotated when an image is taken from camera in the portrait orientation.
            kCGImageSourceCreateThumbnailWithTransform: true
    ]
    scaledImage = CGImageSourceCreateThumbnailAtIndex(imageSource, 0, options as CFDictionary)

    return scaledImage
}


let filePath = Bundle.main.path(forResource:"large_leaves_70mp", ofType: "jpg")

let url = NSURL(fileURLWithPath: filePath ?? "")

let image = resize(url: url, maxPixelSize: 600)
```

## Optimizing when in the background

### foreground/background

The strategy is simple, when `UIApplicationDidEnterBackground`, we unload images; and when `UIApplicationWillEnterForeground`, load images. 


### on-screen/off-screen 

unload large resource when off-screen, `viewDidDisappear`; and load large resource when on-screen, `viewWillAppear`. 

## Debug 

Use memory graphs to further understand and reduce memory footprint


## Ref 

- [An great example of how to use command line tools to find root cause of an image memory issue.](https://developer.apple.com/videos/play/wwdc2018/416/)  