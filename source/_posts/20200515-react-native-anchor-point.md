---
title: Anchor point for 3D Transform in React Native
date: 2020-05-15 11:07:07
categories: 
  - React Native
tags:
  - Animation
---

## What is Anchor Point 

Anchor point in a view is a point in the unit coordinate space, about which all geometric manipulations to the view occur. 

> Defines the anchor point of the layer's bounds rectangle. Animatable. 
> You specify the value for this property using the unit coordinate space. The default value of this property is (0.5, 0.5), which represents the center of the layer’s bounds rectangle. `All geometric manipulations to the view occur about the specified point`. For example, applying a rotation transform to a layer with the default anchor point causes the layer to rotate around its center. Changing the anchor point to a different location would cause the layer to rotate around that new point.                -- [Apple dev](https://developer.apple.com/documentation/quartzcore/calayer/1410817-anchorpoint)

By default, the anchor point for the rotation is the center of the view. That means the rotation of a view is based on its center. Inspired by this [article](https://commitocracy.com/implementing-foldview-in-react-native-e970011f98b8#.k95f793qe), which teaches us how to rotate a view based from the origin point, I got an important clue to implement the anchor point.  

### transformOrigin

> “Since the transform origin of a view is at its horizontal and vertical center by default, to rotate it in x-space along the bottom, we need to first shift our view’s origin on the y-axis by 50% of the view’s height, then apply rotation, then shift it back to the original center.” — [@jmurzy](https://commitocracy.com/implementing-foldview-in-react-native-e970011f98b8)

So @jmurze implemented a [function]
(https://gist.github.com/jmurzy/0d62c0b5ea88ca806c16b5e8a16deb6a#file-foldview-transformutil-transformorigin-js)

```js
function transformOrigin(matrix, origin) {
  const { x, y, z } = origin;

  const translate = MatrixMath.createIdentityMatrix();
  MatrixMath.reuseTranslate3dCommand(translate, x, y, z);
  MatrixMath.multiplyInto(matrix, translate, matrix);

  const untranslate = MatrixMath.createIdentityMatrix();
  MatrixMath.reuseTranslate3dCommand(untranslate, -x, -y, -z);
  MatrixMath.multiplyInto(matrix, matrix, untranslate);
}
```
`reuseTranslate3dCommand`, in the [react-native source code](https://github.com/facebook/react-native/blob/a1ac2518a364ebcd3cc024a22229cadc1791e1c4/Libraries/Utilities/MatrixMath.js#L95), replace the 12th, 13th, 14th element in the matrix by parameters, z, y, z to achieve `translation` effect. 

This function called in [processTransform.js#L79](https://github.com/facebook/react-native/blob/a1ac2518a364ebcd3cc024a22229cadc1791e1c4/Libraries/StyleSheet/processTransform.js#L79), in which, RN generates a transform matrix based on `transform object` we provide. ()[https://github.com/facebook/react-native/blob/a1ac2518a364ebcd3cc024a22229cadc1791e1c4/Libraries/StyleSheet/processTransform.js#L43]

### What `transformOrigin` does is to
```
1. translate the view by x, y, z on the x-axis, y-axis, z-axis 
2. apply rotation
3. translate the view back by -x, -y, -z on the x-axis, y-axis, z-axis 
```

### Plain Code

So, we can also use `transform` style to set the anchor point. For example, the following code sets the anchor point of the view as (0, 0.5). This will make the rotate base on the left side of the view. 

```javascript
   const transform = {
      transform: [
          {translateX: -w / 2}, 
          rotateY, 
          {translateX: w / 2}
      ]
    }
    return (   
        <Animated.View style={transform}></Animated.View>
    )
  }

```

## react-native-anchor-point

In this [package](https://github.com/sueLan/react-native-anchor-point), I provides a function `withAnchorPoint`, to inject the above code into your `transform` object. It works well with the original way you use transform and. So, you don't have to use `ref` to set the `transformOrigin` any more. This would make the `3d transform` in React Native easier to implement. 

![](./rotateZ.gif)
![](./rotateXY.gif)
![](./rotate.gif)

## Ref

- [Matrices](http://www.opengl-tutorial.org/beginners-tutorials/tutorial-3-matrices/)
- [react-native-foldview](https://github.com/jmurzy/react-native-foldview)
- [RN How to set the anchor point of a view.](https://stackoverflow.com/a/60632809/4026902)
- https://pages.mtu.edu/~shene/COURSES/cs3621/NOTES/geometry/geo-tran.html
