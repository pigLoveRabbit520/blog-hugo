---
title: Keras 简单尝试
author: pigLoveRabbit
tags:
  - Keras
categories:
  - 深度学习
date: 2024-10-25 19:00:00
---

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