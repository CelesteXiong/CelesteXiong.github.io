---
layout: post
title: Introduction to our paper published on 2020 ICIP
date: 2020-09-30 11:09
category: Research
author: Shunagyu Xiong
tags: [Publication,  Deep Learning]
summary: High Resolution Demoire Network
---


# Useful information
1. paper link: (https://ieeexplore.ieee.org/document/9191255)[https://ieeexplore.ieee.org/document/9191255]
2. github link: (https://github.com/Rheaaaaayy/HRDN-DEMOIRE)[https://github.com/Rheaaaaayy/HRDN-DEMOIRE]

# Introduction

Our work in ICIP 2020 is high resolution demoire network, which from East China Normal University.
In this paper, our goal is to remove moire artifact, which always appears when people photograph LCD screens. The moire patterns can be seen in these pictures; such artifacts severely damage the image quality.

We define the problem as follows: given an image with moire pattern, model process the contaminated image and output a clean image, the output image should be as close to the corresponding clean image as possible.

![question_definition](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203120632597.png)

<!--more-->

There are several **previous methods** for demoiring; however, they mostly used a multi-scale framework to identify the complex frequencies of moire, while the relationship between different scales was ignored. **Our work** focuses on relationships between feature maps with different resolutions. 

We handle the moire pattern's multi-frequencies with High-Resolution Demoire Network(HRDN). The figure shows our model architecture. HRDN introduces several parallel branches with different resolutions. Each branch can be vertically split into subnetworks within stages, found in the middle blue part.

![network_structure](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203120739712.png)

In such parallel multi-resolution architecture, the HRDN manipulates different frequencies in different scales.

**Besides**, based on information exchange among multi-resolutions, the network can fully take advantage of multi-resolution.

To be more specific, we use Information Exchange Module(IEM) and the Final Feature Fusion Layer(FFF) to fuse features from low level to high level, and vice versa.

![IEM](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203120842255.png)

The Information exchange module(IEM) Module performs parallel convolution operation and fuse feature maps of multiple resolutions at each stage. 

As for Final Feature Fusion(FFF) , we utilize it to combine the previous modules' outputs further. As shown in Figure, FFF is implemented down-to-top. 

![FFF](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203120954873.png)

**First**, we adopt a SE block to perform channel-wise attention on each branch's outputs to mitigate channel imbalance after concatenating. **Then** we reduce the number of channels by convolution layers while the size of feature maps remains unchanged. A pixel-shuffle operation doubles the size of feature maps and changes the number of channels to 3. **At last**, the feature maps enlarged from pixel-shuffle are concatenated to higher resolution features.



When we **experiment** with the model, we met the **challenge** that the Pixel-wise loss function results in over-smoothing. To this end, we implement the Edge Enhanced Loss containing L1 Charbonnier Loss and L1 Sobel Loss, which are defined here, where $N$ represents the batch size. ![img](/../../media/2020-09-30-publication-2020ICIP-demoire/clip_image004.png) and ![img](/../../media/2020-09-30-publication-2020ICIP-demoire/clip_image006.png) denote the output demoire image from HRDN and the ground truth, respectively. The hyperparameter $\epsilon$  = 0:001, and $\alpha$ is an adjustable parameter during the training.

![Edge Enhanced Loss](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203121035497.png)We **compare** our HRDN with some demoireing methods based on deep learning and image restoration methods, such as DnCNN for image denoising, U-net, Sun's network, MopNet. 

Here are some quantitative and qualitative results. Our method is more competitive in PSNR and GFLOPs, and comparable in SSIM, even with fewer parameters.

![results](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203121251938.png)

The **visualization of results** shows that both DnCNN and Sun's network can not remove moire patterns successfully, and the limitations of MopNet are over-smoothing and inaccurate colors. Our HRDN can get clean demoired images with better visual effects.

What's more, **three ablation studies** were implemented to explore the importance of different parts of our HRDN. We remove the sobel loss function, final feature fusion, and fuse layer in each IEM, respectively. As shown in the right graph, the fuse layer plays the most important role in our model. It means that the information exchange function is helpful to handle the moire pattern. 

![ablation studies](/../../media/2020-09-30-publication-2020ICIP-demoire/image-20201203121348949.png)

Above is all our work. Our code is available on [GitHub](https://github.com/Rheaaaaayy/HRDN-DEMOIRE), if you like it, please give us a little star; thank you for watching! 