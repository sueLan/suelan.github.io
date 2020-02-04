---
title: The Weights of the YOLO Neural Network
date: 2019-05-12 17:39:03
tags:
 - YOLO
categories: 
 - Neural Network
---

## Load Weights

First, see this C library function:  
```c
size_t fread ( void * ptr, size_t size, size_t count, FILE * stream );
```
**Parameters**
- **ptr** − Pointer to a block of memory with a size of at least (size*count) bytes, converted to a void*.

- **size** − This is the size in bytes of each element to be read.

- **count** − This is the number of elements, each one with a size of size bytes.

- **stream** − This is the pointer to a FILE object that specifies an input stream.

Then, in the `load_weights_upto` function in `parser.c`, it begins to load weights for layers from xxx.weights. 
```c
void load_weights_upto(network *net, char *filename, int start, int cutoff) {
 ...
     // Begin to load weights for layers
    for(i = start; i < net->n && i < cutoff; ++i){
        layer l = net->layers[i];
        if (l.dontload) continue;
        if(l.type == CONVOLUTIONAL || l.type == DECONVOLUTIONAL){
            load_convolutional_weights(l, fp);
        }
  ...
}
```

### Convolutional Layer
In `load_convolutional_weights` function, it loads values of biases for filters. One bias for one filter. 

```c
    fread(l.biases, sizeof(float), l.n, fp);
```
`l.n` is the number of filter in this layer. 

Assign the values to scales, rolling_mean and rooling_variance 
```
 if (l.batch_normalize && (!l.dontloadscales)){
        fread(l.scales, sizeof(float), l.n, fp);
        fread(l.rolling_mean, sizeof(float), l.n, fp);
        fread(l.rolling_variance, sizeof(float), l.n, fp);
```
        
And load weights for filters in this layer. The default value of `l.groups` is 1.
```c
    int num = l.c/l.groups*l.n*l.size*l.size;
    fread(l.weights, sizeof(float), num, fp);
```

`l.c` is input channel. The number of input channel equal to the number of channel of a filter in this layer. Thus, `l.size` is the size of a filter.

For a 3x3x3 filter (size=3, channel=3), it has 3x3x3=27 parameters as follow  

![-w208](/img/15576478323378/15576531132966.jpg)

If a convolutional layer has 3 such filters, it will have 3 values for bias, 3x3x3x3 .Its weight layout in memory is like this: 

![-w746](/img/15576478323378/15576535368751.jpg)

## Write Weights 

`save_weights_upto` and `save_convolutional_weights`functions show how to save weights into a xxx.weights file. It's just a reverse process. 

```c
void save_convolutional_weights(layer l, FILE *fp)
{
    if(l.binary){
        //save_convolutional_weights_binary(l, fp);
        //return;
    }
    int num = l.nweights;
    fwrite(l.biases, sizeof(float), l.n, fp);
    if (l.batch_normalize){
        fwrite(l.scales, sizeof(float), l.n, fp);
        fwrite(l.rolling_mean, sizeof(float), l.n, fp);
        fwrite(l.rolling_variance, sizeof(float), l.n, fp);
    }
    fwrite(l.weights, sizeof(float), num, fp);
}
```

```
size_t fwrite ( const void * ptr, size_t size, size_t count, FILE * stream );

```

**Parameters**
- **ptr**
Pointer to the array of elements to be written, converted to a const void*.
- **size**
Size in bytes of each element to be written.
size_t is an unsigned integral type.
- **count**
 Number of elements, each one with a size of size bytes.
size_t is an unsigned integral type.
- **stream**
Pointer to a FILE object that specifies an output stream.
