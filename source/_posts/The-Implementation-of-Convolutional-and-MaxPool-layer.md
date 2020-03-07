---
title: YOLO - From Configuration File to Convolutional Layers
date: 2019-05-08 22:19:05
categories: 
    - Neural Network 
tags:
    - YOLO 
---

 Let's firstly see how `Darknet` construct a neural network. See at `detector.c` `test_detector` function, it construct a network by parsing  the `xxx.cfg` file and `xxx.weights` file. In my case, they are yolo3-tiny.cfg and yolo3-tiny.weights 
 <!-- more --> 


I am trying to understand [Darknet source code](https://pjreddie.com/darknet/yolov2/) that implements YOLO algorithm. First, I run the detector.

```
./darknet detect cfg/yolov3-tiny.cfg yolov3-tiny.weights data/dog.jpg
```
## Parse the argumenets 

In `main` function, it goes to function `test_detector` according to the first argument`detect`. 

```c
if (0 == strcmp(argv[1], "detect")){
   float thresh = find_float_arg(argc, argv, "-thresh", .5);
   char *filename = (argc > 4) ? argv[4]: 0;
   char *outfile = find_char_arg(argc, argv, "-out", 0);
   int fullscreen = find_arg(argc, argv, "-fullscreen");
   test_detector("cfg/coco.data", argv[2], argv[3], filename, thresh, .5, outfile, fullscreen);
   }   
```

Here is the architecture of neural network defined by yolov3-tiny.cfg

```

layer     filters    size              input                output
    0 conv     16  3 x 3 / 1   416 x 416 x   3   ->   416 x 416 x  16  0.150 BFLOPs
    1 max          2 x 2 / 2   416 x 416 x  16   ->   208 x 208 x  16
    2 conv     32  3 x 3 / 1   208 x 208 x  16   ->   208 x 208 x  32  0.399 BFLOPs
    3 max          2 x 2 / 2   208 x 208 x  32   ->   104 x 104 x  32
    4 conv     64  3 x 3 / 1   104 x 104 x  32   ->   104 x 104 x  64  0.399 BFLOPs
    5 max          2 x 2 / 2   104 x 104 x  64   ->    52 x  52 x  64
    6 conv    128  3 x 3 / 1    52 x  52 x  64   ->    52 x  52 x 128  0.399 BFLOPs
    7 max          2 x 2 / 2    52 x  52 x 128   ->    26 x  26 x 128
    8 conv    256  3 x 3 / 1    26 x  26 x 128   ->    26 x  26 x 256  0.399 BFLOPs
    9 max          2 x 2 / 2    26 x  26 x 256   ->    13 x  13 x 256
   10 conv    512  3 x 3 / 1    13 x  13 x 256   ->    13 x  13 x 512  0.399 BFLOPs
   11 max          2 x 2 / 1    13 x  13 x 512   ->    13 x  13 x 512
   12 conv   1024  3 x 3 / 1    13 x  13 x 512   ->    13 x  13 x1024  1.595 BFLOPs
   13 conv    256  1 x 1 / 1    13 x  13 x1024   ->    13 x  13 x 256  0.089 BFLOPs
   14 conv    512  3 x 3 / 1    13 x  13 x 256   ->    13 x  13 x 512  0.399 BFLOPs
   15 conv    255  1 x 1 / 1    13 x  13 x 512   ->    13 x  13 x 255  0.044 BFLOPs
   16 yolo
   17 route  13
   18 conv    128  1 x 1 / 1    13 x  13 x 256   ->    13 x  13 x 128  0.011 BFLOPs
   19 upsample            2x    13 x  13 x 128   ->    26 x  26 x 128
   20 route  19 8
   21 conv    256  3 x 3 / 1    26 x  26 x 384   ->    26 x  26 x 256  1.196 BFLOPs
   22 conv    255  1 x 1 / 1    26 x  26 x 256   ->    26 x  26 x 255  0.088 BFLOPs
   23 yolo
```
Now, let's see how Darknet construct a nerual network. See  `detector.c` `test_detector` function, it construct a network by parsing  the `xxx.cfg` file and `xxx.weights` file. In my case, they are yolo3-tiny.cfg and yolo3-tiny.weights 


## Parse the configuration file

### Sections in the file 

The code to parse the yolo.cfg file is here:

```c
list *read_cfg(char *filename)
{
    FILE *file = fopen(filename, "r");
    if(file == 0) file_error(filename);
    char *line;
    int nu = 0;
    list *options = make_list();
    section *current = 0;
    while((line=fgetl(file)) != 0){
        ++ nu;
        strip(line);
        switch(line[0]){
            case '[':
                current = malloc(sizeof(section));
                list_insert(options, current);
                current->options = make_list();
                current->type = line;
                break;
            case '\0':
            case '#':
            case ';':
                free(line);
                break;
            default:
                if(!read_option(line, current->options)){
                    fprintf(stderr, "Config file error line %d, could parse: %s\n", nu, line);
                    free(line);
                }
                break;
        }
    }
    fclose(file);
    return options;
}
```


for exmaple: 

```js
[net]              // '[' is a tag for a section, the type of current setion is '[net]'
# Testing          // ignore
batch=1         
subdivisions=1
# Training
# batch=64
# subdivisions=2
width=416
height=416
channels=3
momentum=0.9
decay=0.0005
angle=0
saturation = 1.5
exposure = 1.5
hue=.1
                 // ignore
learning_rate=0.001
...

[convolutional]    // [convoltional] 
...

[maxpool]      // [maxpool] 

...
[yolo]
...

[route]
...

```

#### `[net]`

In section '[net]', `batch=1` is a option stored in `kvp`(option_list.c line 70) structure. Its key is batch, value is 1. Then this kvp object will be inserted into a node list (see it at option_list.c line76 & list.c line 40).
After parsing the yolo3-tiny.cfg file, We will get a section list; its size is 25. Because there are 25 \'[\' tags in yolo3-tiny.cfg


In `parse_network_cfg` function, it parses the `[net]` section to get the params for the whole network. 

```c
network *parse_network_cfg(char *filename)
{
    list *sections = read_cfg(filename);
    node *n = sections->front;
    if(!n) error("Config file has no sections");
    network *net = make_network(sections->size - 1);
    // other codes ...
    
}
```

#### `[convolutional]`

Then parse the different sections. 

```c
        s = (section *)n->val;
        options = s->options;
        layer l = {0};
        LAYER_TYPE lt = string_to_layer_type(s->type);
        if(lt == CONVOLUTIONAL){
            l = parse_convolutional(options, params);
        }else if(lt == DECONVOLUTIONAL){
            l = parse_deconvolutional(options, params);
        }else if(lt == LOCAL){
            l = parse_local(options, params);
        }else if(lt == ACTIVE){
            l = parse_activation(options, params);
        // other code here ...
       
```


For the first `[convolutional]` section in the yolo3-tiny.cfg as follow, the darknet will construct a `convolutional_layer` using thess params (see function `parse_convolutional` in parse.c and  function `make_convolutional_layer` in convolutional_layer.c)

```js
[convolutional]
batch_normalize=1
filters=16
size=3
stride=1
pad=1
activation=leaky
```

In this layer, there are 16 filters; the size of each filter is 3X3Xnum_channel; what is num_channel? well, `the number of channels in a filter must match the number of channels in input volume`, so here num_channel is equal to 3. The stride value for filters is 1, padding value is 1. 


Let's see how darknet calculate the output size of convolutional_layer by the input size(`l.h`) and filter params (`l.size`, `l.pad`, `l.stride`). There is a formula that shows how size of input volume relates to the one of output volume
 

```c

int convolutional_out_height(convolutional_layer l)
{
    return (l.h + 2*l.pad - l.size) / l.stride + 1;
}

int convolutional_out_width(convolutional_layer l)
{
    return (l.w + 2*l.pad - l.size) / l.stride + 1;
}

```

 As for yolo3-tiny.cfg, for this first convolutional_layer, its input size is 416 x 416 and channel is 3. So its ouput height is (416+2x1 - 3)/1 + 1 = 416, its output width is 416 too. `What about its output channel? It equals to the number of filters (16)`. 
 
 ```c
 
  l.out_c = n    // in func make_convolutional_layer
  
 ```
 So its output volume size is 416 X 416 X 16.
 
 
 ![02b028d9.png](/img/995676c2-24ed-4165-8224-0bc01148242a/9b76325f.png)
 
 For a beginner, I strongly recommend these courses: [Strided Convolutions - Foundations of Convolutional Neural Networks \| Coursera](https://www.coursera.org/lecture/convolutional-neural-networks/strided-convolutions-wfUhx) and  [One Layer of a Convolutional Network - Foundations of Convolutional Neural Networks \| Coursera](https://www.coursera.org/lecture/convolutional-neural-networks/one-layer-of-a-convolutional-network-nsiuW)
  
Now, we have 16 filters that are 3X3X3 in this layer, `how many parameters does this layer have`?  Each filter is a 3X3X3 volume, so it's 27 numbers tp be learned, and then plus the bias, so that was the b parameters. it's 28 parameters. There are 16 filters so that would be 448 parameters to be learned in this layer. 
 
```c

    // c: the number of channels; n: the number of filters; 
    // size: the number of filter width or height; groups: default is 1 
    l.weights = calloc(c/groups*n*size*size, sizeof(float));
    l.weight_updates = calloc(c/groups*n*size*size, sizeof(float));

    l.biases = calloc(n, sizeof(float));
    l.bias_updates = calloc(n, sizeof(float));

    l.nweights = c/groups*n*size*size;
    l.nbiases = n;

```
 
 
#### Activation 
 
 In this convolution layer, it choose leaky ReLU as activation function. The function is defined as follow  where α is a small constant.
 
 $$
f(x)=\begin{cases}
αx,\quad x\leq 0 \\\\ 
x,\quad x>0
\end{cases}
$$

 
 Still, I recommend this course for a beginner. [Activation functions - Shallow neural networks \| Coursera](https://www.coursera.org/lecture/neural-networks-deep-learning/activation-functions-4dDC1)

 
There are `forward_activation_layer` and `backward_activation_layer` in Darknet. Both of them handle batch inputs. 

For forward activation layer, leaky_activate is to computes f(x)

 ```c
 static inline float leaky_activate(float x){return (x>0) ? x : .1*x;}
 ```
 For backward activation layer, leaky_gradient returns the slop of the function 

```c
static inline float leaky_gradient(float x){return (x>0) ? 1 : .1;}
```


 
 
#### [maxpool]
 
 Maxpool layer is used to reduce the size of representation to speed up computation as well as to make some of the features it detects a bit more robust. Look at the `tiny-yolo3.cfg`
 
 ```js
 [maxpool]
size=2
stride=2

 ```
 ```c
 maxpool_layer make_maxpool_layer(int batch, int h, int w, int c, int size, int stride, int padding)
{
    maxpool_layer l = {0};
    l.type = MAXPOOL;
    l.batch = batch;
    l.h = h;
    l.w = w;
    l.c = c;         // output channel equals to input one 
    l.pad = padding;  // default value is size - 1
    l.out_w = (w + padding - size)/stride + 1;
    l.out_h = (h + padding - size)/stride + 1;
    l.out_c = c;
    l.outputs = l.out_h * l.out_w * l.out_c;
    // other codes ...
    return l;
}

 
 ```
 This `[maxpool]` sections comes after the `[convolutional]` section. Its input size(416 x 416 x 16) equal to the output size of the former layer (416 x 416 x  16). The filter size is 2 x 2, stride is 2. Each time, the filter would move 2 steps, for a 4x4x1 input volume, its output is 2x2x1 volume. 
![e65fb56d.png](/img/995676c2-24ed-4165-8224-0bc01148242a/e65fb56d.png)
```
9 == max(1, 3, 2, 9)
2 == max(2, 1, 1, 1)
6 == max(1, 3, 5, 6)
3 == max(2, 3, 1, 2)
```
So in this layer, its ouput width equals to (int)((416+ 1 - 2)/2 + 1), 208. And the number of its output channels equals to the number of input channels. Now, we know its output volume size is 208 X 208 X 16. There is no parameter to be learned. 

**input volume size**: 

$$ n_H . n_W . n_c$$

  $n_c$ : the number of channels

**output volume size**: 

$$(\frac{n_H + padding-f}{stride} + 1) . (\frac{n_W + padding-f}{stride} +1) . n_c$$
 
$f$: the width or height of a filter
 
 [Pooling Layers - Foundations of Convolutional Neural Networks \| Coursera](https://www.coursera.org/lecture/convolutional-neural-networks/pooling-layers-hELHk)
 
 #### Why does 1 x 1 convolution do? 
 
 [Networks in Networks and 1x1 Convolutions - Deep convolutional models: case studies \| Coursera](https://www.coursera.org/lecture/convolutional-neural-networks/networks-in-networks-and-1x1-convolutions-ZTb8x)
 
 
 For example, in this picture, the number of input volume channels ,192, has gotten too big, we can shrink it to a 28x28x32 dimension volume using 32 filters that are 1x1x192. So this is a way to shrink the number of channels .
 
 ![a085e0e4.png](/img/995676c2-24ed-4165-8224-0bc01148242a/a085e0e4.png)
 
 
 In YOLO, it implements fully connected layer by two convolutional layer. 
 
 ```
[convolutional]
batch_normalize=1
filters=256
size=3
stride=1
pad=1
activation=leaky

[convolutional]
size=1
stride=1
pad=1
filters=255
activation=linear
 ```
 
 [Convolutional Implementation of Sliding Windows - Object detection \| Coursera](https://www.coursera.org/lecture/convolutional-neural-networks/convolutional-implementation-of-sliding-windows-6UnU4)
 
