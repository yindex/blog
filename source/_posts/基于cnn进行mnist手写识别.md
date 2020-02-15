---
title: 基于CNN的MNIST手写识别尝试
date: 2017-07-05 10:01:09
categories: 深度学习
mathjax: true
tags: ['cnn', 'mnist']
---

<!-- toc -->

# 基于CNN的MNIST手写识别尝试

## 0x01 前言
MNIST手写识别基本上是深度学习领域的Hello world了，本文主要是使用tensorflow炮筒以下流程，随便添加了几个网络层进行手写识别.

## 0x02 CNN网络结构配置

### 0x0201 数据预处理

因为mnist输入数据是一个784维的向量( $28\times28$ ),所以首先通过`tf.reshape`方法将输入数据转换成(28, 28, 1)的矩阵(width=28，height=28,单通道)

### 0x0202 网络结构

实验目的，网络结构比较简单，主要配置了两个卷基层，两个池化层，一个全连接层.
>
    - (3, 3, 1, 1) 卷积
    - relu6激活函数
    - max_pool 最大池化
    - (3, 3, 1, 4) 卷积
    - relu
    - max_pool 最大池化
    - fc 全连接层
    - dropout 抑制过拟合
    - softmax 输出 

卷积层conv2d_1，使用$3\times3$大小的卷积核(vgg就是$3\times3$)进行卷积操作，卷积输出通过relu6激活函数进行激活.这一层输出为(28, 28, 1)
对卷积层conv2d_1 输出进行最大池化输出为(14,14,1)
卷积层conv2d_2 使用$3\times3$大小卷积核，feature map为4进行卷积操作， 并通过relu函数进行激活， 输出为(14,14, 4)
对conv2d_2 进行最大池化，输出为（7， 7, 4)
最后对conv2d_2输出进行全连接操作，全连接输出为1024维的特征向量，对该特征向量dropout正则化抑制过拟合. 最后通过softmax产生最终输出.

## 0x03 Tensorflow实现
```python
#coding:utf-8
import tensorflow as tf
from tensorflow.examples.tutorials.mnist import input_data
import time

"""
权重初始化
初始化为一个接近0的很小的正数
"""
def weight_variable(shape):
    initial = tf.truncated_normal(shape, stddev = 0.1)
    return tf.Variable(initial)

def bias_variable(shape):
    initial = tf.constant(0.1, shape = shape)
    return tf.Variable(initial)

def conv2d(x, W):
    return tf.nn.conv2d(x, W, strides=[1, 1, 1, 1], padding = 'SAME')

def max_pool_2x2(x):
    return tf.nn.max_pool(x, ksize = [1, 2, 2, 1],
                          strides = [1, 2, 2, 1], padding = 'SAME')
    # tf.nn.max_pool(value, ksize, strides, padding, data_format='NHWC', name=None)
    # x(value)              : [batch, height, width, channels]
    # ksize(pool大小)        : A list of ints that has length >= 4. The size of the window for each dimension of the input tensor.
    # strides(pool滑动大小)   : A list of ints that has length >= 4. The stride of the sliding window for each dimension of the input tensor.

start = time.clock() #计算开始时间
mnist = input_data.read_data_sets("MNIST_data/", one_hot=True) #MNIST数据输入


x = tf.placeholder(tf.float32,[None, 784])
x_image = tf.reshape(x, [-1, 28, 28, 1]) #最后一维代表通道数目，如果是rgb则为3
W_conv1 = weight_variable([3, 3, 1, 1])
b_conv1 = bias_variable([1])

h_conv1 = tf.nn.relu6(conv2d(x_image, W_conv1) + b_conv1)
# x_image -> [batch, in_height, in_width, in_channels]
#            [batch, 28, 28, 1]
# W_conv1 -> [filter_height, filter_width, in_channels, out_channels]
#            [5, 5, 1, 32]
# output  -> [batch, out_height, out_width, out_channels]
#            [batch, 28, 28, 32]
h_pool1 = max_pool_2x2(h_conv1)
# h_conv1 -> [batch, in_height, in_weight, in_channels]
#            [batch, 28, 28, 32]
# output  -> [batch, out_height, out_weight, out_channels]
#            [batch, 14, 14, 32]

"""
第二层 卷积层

h_pool1(batch, 14, 14, 32) -> h_pool2(batch, 7, 7, 64)
"""
W_conv2 = weight_variable([3, 3, 1, 4])
b_conv2 = bias_variable([1])

h_conv2 = tf.nn.relu(conv2d(h_pool1, W_conv2) + b_conv2)
# h_pool1 -> [batch, 14, 14, 32]
# W_conv2 -> [5, 5, 32, 64]
# output  -> [batch, 14, 14, 64]
h_pool2 = max_pool_2x2(h_conv2)
# h_conv2 -> [batch, 14, 14, 64]
# output  -> [batch, 7, 7, 64]

"""
第三层 全连接层

h_pool2(batch, 7, 7, 64) -> h_fc1(1, 1024)
"""
W_fc1 = weight_variable([7 * 7 * 4, 1024])
b_fc1 = bias_variable([1024])

h_pool2_flat = tf.reshape(h_pool2, [-1, 7 * 7 * 4])
h_fc1 = tf.nn.relu(tf.matmul(h_pool2_flat, W_fc1) + b_fc1)

"""
Dropout

h_fc1 -> h_fc1_drop, 训练中启用，测试中关闭
"""
keep_prob = tf.placeholder("float")
h_fc1_drop = tf.nn.dropout(h_fc1, keep_prob)

"""
第四层 Softmax输出层
"""
W_fc2 = weight_variable([1024, 10])
b_fc2 = bias_variable([10])


y_conv = tf.nn.softmax(tf.matmul(h_fc1_drop, W_fc2) + b_fc2)

"""
训练和评估模型

ADAM优化器来做梯度最速下降,feed_dict中加入参数keep_prob控制dropout比例
"""
y_ = tf.placeholder("float", [None, 10])
cross_entropy = -tf.reduce_sum(y_ * tf.log(y_conv)) #计算交叉熵
with tf.name_scope("train"):

    train_step = tf.train.AdamOptimizer(1e-4).minimize(cross_entropy) #使用adam优化器来以0.0001的学习率来进行微调
correct_prediction = tf.equal(tf.argmax(y_conv,1), tf.argmax(y_,1)) #判断预测标签和实际标签是否匹配
accuracy = tf.reduce_mean(tf.cast(correct_prediction,"float"))

sess = tf.Session() #启动创建的模型
tf.summary.FileWriter("logs/", sess.graph)
sess.run(tf.initialize_all_variables()) #旧版本

```
