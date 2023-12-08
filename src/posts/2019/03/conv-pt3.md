---
title: "Demystifying the math and implementation of Convolutions: Part III."
date: "2019-04-03"
tags: 
- convolutions
- deep learning
- AI
---

In this part, we will discuss how to implement convolutions over 3D matrices, such as RGB images. We will also discuss how to optimize the code using im2col and vectorization.
<!-- excerpt -->

## Convolution over Volume
Convolution over volume refers to 3D input matrices, such as RGB Images. While I will not be implementing them here (not in this post at least), they are essentially easy to implement if you are able to model them as matrices and vectors. However, I will explain the process of modeling such high dimensional data.

First thing, we have seen convolutions over 2D matrices, which can also treated as 3D matrices with a depth of 1. Starting from there, let's remodel what we have done so far in terms of 3D matrices. Let M be a 3D image, and the shape of out matrix is rows*columns*depth, as depicted in figure 1.


{% asset_img "conv/conv-depth-11.png" "Fig. 1: 3D Matrix (Image) of shape 6x6x3" %}

Given an input image M: m*n*d and filter K: m'*n'*d' the convolution operation is only valid if and only if k = k', thus we will refer to the depth of the input matrix and the filter as k. 

The most important things to remember are:

- Convolution of any kind of matrix over a single filter, results in a 2D Image.
- Matrix and Filter must have the same rank (i.e Number of dimensions) and the depth must be the same
- The fact that I am going to assume a stride of 1 for the rest of this tutorial

Now, with that in mind, let's see how our convolution goes:


{% asset_img "conv/conv-depth-21-1024x537.png" "Fig. 2: Convolution of 3D matrix over 1 3D Filter" %}

Now, we wont be starting from scratch. Let's recall our good ol' Convolution formula over standard Matrices
{% asset_img "conv/png (2).png" "Eq. 1: Convolution over 2D image and a single 2D filter." %}

We transformed this problem into matrix dot product using `im2col`, as depicted in figure 3.
{% asset_img "conv/conv-im2col-1-filter-1024x488 (1).png" "Fig 3. im2col, of a 2D image by single 2D filter." %}
Convolution over volume is the pretty much the same, we split the image and filters by depth (channels), perform standard convolution of each image by its respective filters, and the final convolution value is the sum of each single convolution.
{% asset_img "conv/png (3).png" "Eq. 1: Eq. 2: Convolution over 3D images, we added a new index l that iterates through the depth d" %}

Let's see if we can straight optimize this code using im2col. One thing we can do is, when we generate the column matrix M',  is to apply im2col to every channel, and concatenate all columns together as a single column into our new matrix as in the images, below thus the final output shape of M' is (Cx*Cy, m'*n'*d) where Cx and Cy are the convolution output shape calculated using standard output formula from Part I, and m', n', d are the dimensions of the filter
    
{% asset_img "conv/im2col-3d2-1024x510.png" "" %}
{% asset_img "conv/im2col-3d-1024x510.png" "" %}

Next step, is the flatten the Filter, we do the same thing, we flatten the filter into a row matrix, (m*n*d,1) as in figure 5.
{% asset_img "conv/im2col-filter-3d-1.png" "Fig. 5: Flattening a single 3D filter" %}

Once we do this, our convolution is easily expressed as DOT product of  M' dot K'.
{% asset_img "conv/conv-im2col-1-filter-3d-1024x351.png" "Fig. 6: Convolution as DOT product, we reshape the output and obtain the final convolution value, recall that the input matrix M was 4x4x3, and K was 3x3x3 matrix with stride 1 we obtain convolution of 2x2 " %}

If we wanted to add more filters, we simple add them as columns to the flattened filter! Only when reshaping the output matrix C, we consider the number of filters as the depth of the convolution.

# SHOW ME THE CODE

Alright, the following code is based on im2col from previous part, extends it to 3 dimensional inputs and filters. It only processes, however one filter. Your task in to fork it and make it work with n filters. 

<script src="https://gist.github.com/praisethemoon/020528e79f13f48941a7bf3d1322b39b.js"></script>

In case you doubt my math and coding skills, here is a Keras version that exactly the same thing:

<script src="https://gist.github.com/praisethemoon/d1918f1219616b67daa01652a4030ede.js"></script>

## Conclusion
With this, I have finished almost everything related to convolution operator. A couple of details were not mentioned which I will discuss in another part (or a separate post) are

- Zero padding: A technique to increase the output size of a convolution 
- Activation Functions: Transform convolutions into signals/energies
- Max Pooling: Reduce the shape of an input to reduce computation time and preserve the most essential/important information in the image/convolution.

I hope this series of posts have been informative to you as it has been pleasant for me to write.