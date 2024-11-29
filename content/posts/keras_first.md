---
title: Keras 简单尝试
author: pigLoveRabbit
tags:
  - Keras
categories:
  - 深度学习
date: 2024-10-25 19:00:00
---
![Keras](/images/my-keras.png)
<!-- more -->

## 一个demo
csv在这里[下载](https://github.com/npradaschnor/Pima-Indians-Diabetes-Dataset/blob/master/diabetes.csv)
```Python
import os
# Create first network with Keras
from keras.models import Sequential
from keras.layers import Dense
import numpy
os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'
# fix random seed for reproducibility
seed = 7
numpy.random.seed(seed)
# load pima indians dataset
dataset = numpy.loadtxt("pima-indians-diabetes.csv", delimiter=",", skiprows=1)
# split into input (X) and output (Y) variables
X = dataset[:,0:8]
Y = dataset[:,8]
# create model
model = Sequential()
model.add(Dense(12, input_dim=8, kernel_initializer='uniform', activation='relu')) 
model.add(Dense(8, bias_initializer='uniform', activation='relu'))
model.add(Dense(1, kernel_initializer='uniform', activation='sigmoid'))
# Compile model
model.compile(loss='binary_crossentropy', optimizer='adam', metrics=['accuracy']) # Fit the model
model.fit(X, Y, epochs=150, batch_size=10)
# evaluate the model
scores = model.evaluate(X, Y)
print("%s: %.2f%%" % (model.metrics_names[1], scores[1]*100))
```

## 
[视频课程](https://www.bilibili.com/video/BV1Bp4y1D7YL/?p=8)  
6w张图片的数据集
```
import os
from keras.datasets import mnist
from keras import utils
from keras.optimizers import SGD
from keras.models import Sequential
from keras.layers import Dense
import numpy as np

os.environ['TF_CPP_MIN_LOG_LEVEL'] = '2'
# 载入数据
(x_train, y_train), (x_test, y_test) = mnist.load_data()
# (60000,28,28)
print('x_shape:', x_train.shape)
# (60000)
print('y_shape', y_train.shape)
# (60000,28,28)->(60000,784)
x_train = x_train.reshape(x_train.shape[0], -1)/255.0
x_test = x_test.reshape(x_test.shape[0], -1)/255.0
# 换one hot格式
y_train = utils.to_categorical(y_train, num_classes=10)
y_test = utils.to_categorical(y_test, num_classes=10)

# 创建模型，输入784个神经元，输出10个神经元
model = Sequential([
    Dense(units=10, input_dim=784, bias_initializer='one', activation='softmax')
])
# 定义优化器
sgd = SGD(learning_rate=0.2)
# 设置loss function，训练过程中准确率
model.compile(optimizer = sgd, loss='mse', metrics=['accuracy'])
# 训练模型，6w张图每次拿32张训练
model.fit(x_train, y_train, batch_size=32, epochs=10)
# 评估模型
loss,accuracy = model.evaluate(x_test, y_test)
print('\ntest loss', loss)
print('accuracy', accuracy)
```
