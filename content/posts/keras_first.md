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
## 安装Keras
Keras 2 是一个较旧的版本（最新主版本为 Keras 3，但许多用户仍在使用 Keras 2.x，尤其是配合 TensorFlow 1.x 或 2.x 早期版本）。  
装TensorFlow 2.15的时候，Keras 2自动就带上了。
```
pip install tensorflow==2.15.0
```

## 一个demo
```Python
import tensorflow as tf
import keras

print(f"TensorFlow版本: {tf.__version__}")
print(f"Keras版本: {keras.__version__}")

# 创建一个简单的模型
model = keras.Sequential([
    keras.layers.Dense(128, activation='relu', input_shape=(784,)),
    keras.layers.Dense(10, activation='softmax')
])

print("模型创建成功！")
```
[视频课程](https://www.bilibili.com/video/BV1Bp4y1D7YL/?p=8)  

## 实现线性回归
```python
import keras
import numpy as np
import matplotlib
matplotlib.use('Qt5Agg')
import matplotlib.pyplot as plt
# Sequential按顺序构成的模型
from keras.models import Sequential
# Dense全连接层
from keras.layers import Dense

# 使用numpy生成100个随机点
x_data = np.random.rand(100)
noise = np.random.normal(0, 0.01, x_data.shape)
y_data = x_data*0.1 + 0.2 + noise


# 构建一个顺序模型
model= Sequential()
# 在模型中添加一个全连接层
model.add(Dense(units=1, input_dim=1))
# sgd：Stochasticgradientdescent，随机梯度下降法
# mse：Mean Squared Error，均方误差
model.compile(optimizer='sgd', loss='mse')

# 训练3001个批次
for step in range(3001):
    # 每次训练一个批次
    cost = model.train_on_batch(x_data, y_data)
    # 每500个batch打印一次cost值
    if step % 500 == 0:
        print('cost:', cost)
# 打印权值和偏置值
W, b = model. layers[0].get_weights()
print('w:', W, 'b:', b)
# x_data输入网络中，得到预测值y_pred
y_pred = model.predict(x_data)

# 显示随机点
plt.scatter(x_data, y_data)
# 显示预测结果
plt.plot(x_data, y_pred, 'r-', lw=3) # y_pred的shape是(100, 1)，但plot() 函数可以自动处理这种形状差异
plt.show()
```

## 非线性回归
```Python
import keras
import numpy as np
import matplotlib
matplotlib.use('Qt5Agg')
import matplotlib.pyplot as plt
# Sequential按顺序构成的模型
from keras.models import Sequential
# Dense全连接层
from keras.layers import Dense, Activation
from keras.optimizers import SGD, Adam

#使用numpy生成200个随机点
x_data = np.linspace(-0.5, 0.5, 200)
noise = np.random.normal(0, 0.02, x_data.shape)
y_data = np.square(x_data) + noise

#构硬一个顺序模型
model=Sequential()
#在模型中添加一个全连接层
# 构建模型 - 修正版本
model = Sequential()
# 隐藏层使用tanh或relu都可以，但需要合适的初始化
model.add(Dense(units=64, input_dim=1, activation='relu', kernel_initializer='he_normal'))
model.add(Dense(units=32, activation='relu', kernel_initializer='he_normal'))
# 重要：输出层不要使用激活函数（或用线性激活）用于回归问题
model.add(Dense(units=1))  # 默认就是线性激活

# 使用更稳定的优化器，学习率调小
optimizer = Adam(learning_rate=0.01)  # 或者 SGD(learning_rate=0.01, momentum=0.9)
model.compile(optimizer=optimizer, loss='mse')

#训练3001个批次
for step in range(3001):
    #每次训练一个批次
    cost = model.train_on_batch(x_data, y_data)
    #每500个batch打印一次cost值
    if step % 500==0:
        print('cost:', cost)


#model.fit(x_data, y_data, epochs=300, batch_size=32, verbose=1)
        
#x_data输入网络中，得到预测值y_pred
y_pred = model.predict(x_data)
#显示随机点
plt.scatter(x_data, y_data)
#显示预测结果
plt.plot(x_data, y_pred, 'r-', lw=3)
plt.show()
```
### 🔧 其他改进建议

1. **学习率太高**：`SGD(lr=0.3)` 对于这种小规模问题可能太大，容易震荡。建议用 `0.01` 或使用 `Adam` 优化器（自适应学习率）。
2. **增加训练轮数或使用 `fit()`**：`train_on_batch` 虽然灵活，但不如 `model.fit(x_data, y_data, epochs=3000, verbose=1)` 方便，且默认会做更好的内部处理。
3. **隐藏层激活函数没问题**：`relu` 在隐藏层是可以的，也可以试试 `tanh`（对称，适合拟合 x² 这种对称函数）。


## 数字识别
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
