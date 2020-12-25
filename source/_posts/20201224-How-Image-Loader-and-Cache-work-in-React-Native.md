---
title: How Image Loader and Cache work in React Native
date: 2020-12-24 21:31:27
tags:
  - Dive into React Native
categories: 
  - [iOS, React Native]
updated: 2020-12-25  
---

 In react native, it is so intuitive to display different types of images. The [Image Component](https://reactnative.dev/docs/image) can display network images, static resources, temporary local images, and images from local disk. 

```jsx
<Image style={styles.footerLastestImage} source={{ uri: latestImage }} />
```

However, there are actually lots of stories behind this simple line of code.
Today, I would like to explore image loader and image cache in react native, and talk something about how they work under the hood.

## Overview of React/Image module

Let me begin with an activity diagram about **React/Image** module.

![](ActivityReactNative.jpg)

There are several classes playing important roles in image render and cache in react native. **RCTImageView** is the iOS native implementation for **Image component**. It inherits `RCTView` and hold a `UIImageView` to render the image. `RCTImageView` is created by `RCTUIManager`. The `UIImageView` size comes from the style prop in `Image` component in JavaScript side or calculated by `Yoga`. When setting property for Image component in RN or other events triggering the image loading, `RCTImageView` calls `RCTImageLoader` to get the cached image or download image. `RCTImageLoader` is the controller for the workflow of image downloading, decoding. It depends on `RCTNetwork` to fetch the image remotely.  Before trying to download an image, `RCTImageLoader` will try to check [if there is cached image](https://github.com/facebook/react-native/blob/1ee406b9ccbecc52dff3e77d65c6d9b4837e6dab/Libraries/Image/RCTImageLoader.mm#L606) and to reuse the cached image. So here comes `RCTImageCache`, which is for Image Cache in react native iOS. 

## **RCTImageCache**

First of all, keep in mind that **RCTImageCache** uses [NSCache](https://developer.apple.com/documentation/foundation/nscache) to cache images. `NSCache` object used `RCTImageCache` has only [20MB](https://github.com/facebook/react-native/blob/00456211e591930f28a08356141fc8bec52fe3e5/Libraries/Image/RCTImageCache.m#L41) cache capacity in React Native iOS.

### How images added into NSCache? 

#### 1. expiration time 

After successfully downloading the images, **RCTImageLoader** decodes the image using [decodeImageData:size:scale:clipped:resizeMode:completionBlock](https://github.com/facebook/react-native/blob/9500eb8867d25896b1611903a64fac8d81984bf6/Libraries/Image/RCTImageLoader.mm#L935), then it [adds the decoded image to cache](https://github.com/facebook/react-native/blob/00456211e591930f28a08356141fc8bec52fe3e5/Libraries/Image/RCTImageLoader.mm#L806). In this step, it extracts the **expiration time** from the Http Response header. See more in the diagram to see how it get the expire time. The **max-age** is the top priority to decide expiration time for an image here. 

<img src="stale_time.png" style="zoom:50%;" />

#### 2. The cost of image in NSCache

Another key point is that **RCTImageCache** uses [`setObject:forKey:cost:` API](https://developer.apple.com/documentation/foundation/nscache/1416399-setobject?language=objc). 

> The `cost` value is used to compute a sum encompassing the costs of all the objects in the cache. When memory is limited or when the total cost of the cache eclipses the maximum allowed total cost, the cache could begin an eviction process to remove some of its elements.

#### 3. using Bitmap size as cost

```Objective-c

static const NSUInteger RCTMaxCachableDecodedImageSizeInBytes = 2097152; // 2 MB

- (void)addImageToCache:(UIImage *)image
                 forKey:(NSString *)cacheKey
{
  if (!image) {
    return;
  }
  // calculate the bitmap size 
  NSInteger bytes = image.reactDecodedImageBytes;
  // checks if the size of the decoded image bigger than 2MB. Any decoded image occupying more than 2MB memory won't be added into **NSCache**.  
  if (bytes <= RCTMaxCachableDecodedImageSizeInBytes) {
    [self->_decodedImageCache setObject:image
                                 forKey:cacheKey
                                   cost:bytes];
  }
}
```


Before adding an image into the **NSCache**. **RCTImageCache** firstly calculate the bitmap size using the following formula

```objective-c
static NSInteger RCTImageBytesForImage(UIImage *image)
{
  NSInteger singleImageBytes = image.size.width * image.size.height * image.scale * image.scale * 4;
  return image.images ? image.images.count * singleImageBytes : singleImageBytes;
}
```

For a `RGBA` format image, each pixel needs 4 bytes to represent the RGBA value, which contains 4 channels. We know that the value for each channel varies from 0 to 255.  So we need 8 bits, or 1 bytes to store it.

```
A   R   G   B   A   R   G   B   A   R   G   B  
| pixel 0       | pixel 1       | pixel 2  
0  233  2  100  4  155 255  7   8   9   10  11
```

Beside, Image.scale is the scale factor. Multiplying the logical size of the image (stored in the size property) by value, you get the dimensions of the image in pixels. So for the `310*165` `UIImage` object, it totally uses `310*scale*165*scale*4` pixels. 

An important point here is that the **reactDecodedImageBytes** isn't always equal to the true bitmap size in memory after decoding images. Because **RCTImageLoader** will take the size and the resize mode of the **UIImageView** into consideration. [Source code here.](https://github.com/facebook/react-native/blob/00456211e591930f28a08356141fc8bec52fe3e5/Libraries/Image/RCTImageUtils.m#L256). When decoding, it downscale the downloaded image and display the downsized image to fit the image view size. This means `reactDecodedImageBytes` used as image cost in NSCache could be bigger than the actual size of finally displaying image. 

So briefly speaking, for each image added into the `NSCache`, its cost is the bitmap size instead of the file size. `JPG` or `PNG` is kind of compressed format, which has much smaller size. Remember `NSCache` has `20MB` **totalCostLimit** in react native. In addition, bitmap size is much bigger than file size. It is so easy to exceed the totalCostLimit for `NSCache`, and then to trigger the eviction process to remove some of the images. 

#### 4. The unknown storage and eviction policy in NSCache 

In the official doc  [NSCache](https://developer.apple.com/documentation/foundation/nscache) , it says `NSCache` is to temporarily store transient key-value pairs, and doesn't mention whether it uses `disk cache` or not. I added a log to see if it get `image` from the `NSCache` when launching my App without `network`. It does get a few cached images. So far, I think we can say that `NSCache` does have `disk cache` , but the proportion of disk cache is so small that we got a few cached images and the hit rate doesn't improve a lot even when I improve the `totalCostLimit` of NSCache. 

So if your react native app is kind of image-heavy app. Sometimes, when you are in the scroll view full of displaying images. When you scroll down, then scroll back, you can see it fetching and decoding images, even if some of them were rendered just now but moved out of the screen later. Because in this case, the `NSCache` uses up its cache capacity and keep evicting images. Moreover, this eviction process isn't in a guaranteed order. Seems it doesn't simply apply `LRU` policy here. 


### How to get images from cache

The method to get images from `NSCache` looks relatively simple. [ imageForUrl:size:scale:resizeMode](https://github.com/facebook/react-native/blob/00456211e591930f28a08356141fc8bec52fe3e5/Libraries/Image/RCTImageCache.m#L79). 

1. get the key for cached image by using **url, size, scale, resizeMode.** As long as these 4 factors not changed next time, the key for the cached image won't be changed, and you can reuse this image
2. check if the cached image is expired. If it is, remove the image from the cache. 

### How to replace RCTImageCache with your own cache implementation

Let your own cache implementation conform `RCTImageCache` protocol. And then `setImageCache` in `RCTImageLoader`.

```Objective-c
/**
 * Allows developers to set their own caching implementation for
 * decoded images as long as it conforms to the RCTImageCache
 * protocol. This method should be called in bridgeDidInitializeModule.
 */
- (void)setImageCache:(id<RCTImageCache>)cache;
```


## Concurrent Loading and Decoding Tasks 

`RCTImageLoader` maintains queues to schedule image loading and decoding tasks.  The following property controls the number of concurrent tasks for image loading and decoding. 

```objective-c
/**
 * The maximum number of concurrent image loading tasks. Loading and decoding
 * images can consume a lot of memory, so setting this to a higher value may
 * cause memory to spike. If you are seeing out-of-memory crashes, try reducing
 * this value.
 */
@property (nonatomic, assign) NSUInteger maxConcurrentLoadingTasks;

/**
 * The maximum number of concurrent image decoding tasks. Decoding large
 * images can be especially CPU and memory intensive, so if your are decoding a
 * lot of large images in your app, you may wish to adjust this value.
 */
@property (nonatomic, assign) NSUInteger maxConcurrentDecodingTasks;

/**
 * Decoding large images can use a lot of memory, and potentially cause the app
 * to crash. This value allows you to throttle the amount of memory used by the
 * decoder independently of the number of concurrent threads. This means you can
 * still decode a lot of small images in parallel, without allowing the decoder
 * to try to decompress multiple huge images at once. Note that this value is
 * only a hint, and not an indicator of the total memory used by the app.
 */
@property (nonatomic, assign) NSUInteger maxConcurrentDecodingBytes;
```


