---
layout: post
title: Tensorflow utilization
date: 2020-12-04 20:34
category: Tensorflow, Python
author: 
tags: [Deep Learning]
summary: 
---

## Version updation

1. if you want to run TF1.X code in the environment with tensorflow2.0, use the following code to import tensorflow
   ```python
    import tensorflow.compat.v1 as tf
    tf.disable_v2_behavior()
   ```

2. search the Tensorflow 2.0 official website for your function

   ![image-20201204214752415](/../../media/2020-12-04-tensorflow/image-20201204214752415.png)

## Use tensorflow with GPU

1. check which gpu is used: 

   ```python
   import tensorflow as tf
   from tensorflow.python.client import device_lib
   device_lib.list_local_devices()
   ```

2. in `colab`:

   ```python
   with tf.Graph().as_default(), tf.device('/device:GPU:0'):
       
   ```

## Use tensorboard

1. use `tensorboard`: office website: https://www.tensorflow.org/tensorboard/get_started]

   ```python
   # In notebooks, use the %tensorboard line magic.
   # in colab: 
   %load_ext tensorboard
   %tensorboard --logdir your_event_directory
   
   ## if you want to have loaded once, you need to reloadd
   %reload_ext tensorboard
   %tensorboard --logdir your_event_directory
   
   # On the command line, run the same command without "%".
   tensorboard --logdir your_event_directory
   ```

   



## Use tensorflow with TPU

```python
import tensorflow as tf
tpu = tf.distribute.cluster_resolver.TPUClusterResolver()
tf.config.experimental_connect_to_cluster(tpu)
tf.tpu.experimental.initialize_tpu_system(tpu)
strategy = tf.distribute.experimental.TPUStrategy(tpu)

with strategy.scope():
    model = create_model()
    model.compile(
        optimizer=tf.keras.optimizers.Adam(learning_rate=1e-3),
        loss=tf.keras.losses.sparse_categorical_crossentropy,
        metrics=[tf.keras.metrics.sparse_categorical_accuracy])
```



