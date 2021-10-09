title: Arduino使用DHT11测量温湿度
author: Salamander
tags:
  - arduino
  - 传感器
categories:
  - 单片机
date: 2019-08-30 10:37:00
---
## 概览
这篇文章很简单（就是一点电工知识），就是利用[DHT11](https://baike.baidu.com/item/DHT11/1206271)温湿度传感器测量温湿度值，并把结果显示在[LCD1602](https://baike.baidu.com/item/LCD1602/6014393)显示器上。

<!-- more -->

## 实验元器件列表
| 元器件    | 型号          | 数量 | 备注 |
|--------|-------------|----|----|
| 主控板    | arduino Uno   | 1     |    |
| 温湿度传感器 | DHT11      | 1   |     |
| 液晶屏    | 1602 LCD      | 1     |    |
| 电阻     | 1K电阻         | 4   |    |
| 面包板    |               | 1     |    |
| 面包板条线    |               | 若个   |    |
| 数据线    | Uno数据线      | 1   |    |

## 工具和元器件介绍
### DHT11温湿度传感器
![](https://s2.ax1x.com/2019/08/29/mLoDuF.png)
DHT11 传感器接线方法并不复杂，DHT11封装有4个引脚，各个引脚说明如下：

| Pin | 名称   | 注释             |
|-----|------|----------------|
| 1   | VDD  | 供电 3\-5\.5 VDC |
| 2   | DATA | 串行数据，单总线       |
| 3   | NC   | 空脚             |
| 4   | GND  | 接地，电源负极        |

### LCD1602
![1602图片](https://s2.ax1x.com/2019/09/04/nE8k7R.jpg)

1602字符型液晶，是一种专门用来显示字母、数字、符号等的点阵型液晶模块，能够同时显示16x02即32个字符。

LCD1602分为两种：带背光和不带背光，带背光的要后一些，引脚多2个，为16个引脚，如下：

![](https://s2.ax1x.com/2019/08/29/mLTcqg.png)

##### 引脚说明
LCD1602 通常有14条引脚或16条引脚，14与16引脚的差别在于16条引脚多了背光电源线VCC(15脚)和地线GND(16脚)，其它引脚与14脚的LCD完全一样，如下：

| 引脚 | 符号  | 功能说明                                                                   |
|----|-----|------------------------------------------------------------------------|
| 1  | VSS | 一般接地                                                                   |
| 2  | VDD | 接电源（\+5V）                                                              |
| 3  | V0  | 液晶显示器对比度调整端，接正电源时对比度最弱，接地电源时对比度最高（对比度过高时会产生“鬼影”，使用时可以通过一个10K的电位器调整对比度） |
| 4  | RS  | RS为寄存器选择，高电平1时选择数据寄存器、低电平0时选择指令寄存器                                     |
| 5  | R/W | R/W为读写信号线，高电平\(1\)时进行读操作，低电平\(0\)时进行写操作                                |
| 6  | E   | E\(或EN\)端为使能\(enable\)端，写操作时，下降沿使能；读操作时，E高电平有效                         |
| 7  | DB0 | 低4位三态、 双向数据总线 0位（最低位）                                                  |
| 8  | DB1 | 高4位三态、 双向数据总线 1位                                                       |
| 9  | DB2 | 高4位三态、 双向数据总线 2位                                                       |
| 10 | DB3 | 高4位三态、 双向数据总线 3位                                                       |
| 11 | DB4 | 高4位三态、 双向数据总线 4位                                                       |
| 12 | DB5 | 高4位三态、 双向数据总线 5位                                                       |
| 13 | DB6 | 高4位三态、 双向数据总线 6位                                                       |
| 14 | DB7 | 高4位三态、 双向数据总线 7位（busy flag）                                            |
| 15 | BLA | 背光电源正极                                                                 |
| 16 | BLK | 背光电源负极                                                                 | 


## 驱动LCD1602
### 驱动方式
Arduino驱动LCD1602可以选择直接驱动，可以有4线和8线的驱动方式，不过这样还是挺占IO口的，要接的东西多了，就不够用了。所以在这里，我们介绍IIC驱动方式，在LCD1602上得焊接一块IIC转接板（如PCF8574T），只占用2个IO口就能驱动LCD1602。  
`IIC`「Inter-Integrated Circuit 集成电路总线」是一种串行通信总线，应用于板载低速设备间的通讯。由飞利浦公司开发的这一通讯协议，其目的就是为了简化系统硬件设计，减少设备间的连线。  
IIC串行总线有两根信号线，一根是双向的数字线SDA，另一根是时钟线SCL，每个IIC设备都有自己的地址，IIC总线上多个设备间通过设备地址进行区别。

![upload successful](/images/lcd_iic.png)  
上图为本篇使用的IIC转接板，直接焊接于LCD1602。可通过跳线帽设置是否开启背光，通过蓝色电位器调节对比度。IIC设备地址可通过短路A0/A1/A2修改，默认地址用下文的方法查看。  
### 接线
| PCF8574T |    | Arduino |
|----------|----|---------|
| GND      | -> | GND     |
| VCC      | -> | 5V      |
| SDA      | -> | A4      |
| SCL      | -> | A5      |

### 扫描I2C地址
将以下代码拷贝到Arduino IDE，并执行。然后选择工具->串口监视器，把右下角的波特率改为115200，即可读出I2C地址:   


```C++
// I2C Scanner
// Written by Nick Gammon
// Date: 20th April 2011
#include <Wire.h>
void setup() { 
    Serial.begin (115200); // Leonardo: wait for serial port to connect 
    while (!Serial) { } 
    Serial.println (); 
    Serial.println ("I2C scanner. Scanning ..."); 
    byte count = 0; 
    Wire.begin(); 
    for (byte i = 8; i < 120; i++) { 
        Wire.beginTransmission (i); 
        if (Wire.endTransmission () == 0) { 
          Serial.print ("Found address: "); 
          Serial.print (i, DEC); 
          Serial.print (" (0x"); 
          Serial.print (i, HEX); 
          Serial.println (")"); 
          count++; 
          delay (1); // maybe unneeded? 
        } // end of good response 
    } // end of for loop 
    Serial.println ("Done."); 
    Serial.print ("Found "); 
    Serial.print (count, DEC); 
    Serial.println (" device(s).");
} // end of setup
void loop() {}
```




![upload successful](/images/iic_address.png)  
可以看到默认地址是**0x27**（所以不能轻易相信淘宝客服的话。。。）。 



### 安装驱动库
LCD1602的驱动库都是要额外装的。  
在Arduino IDE中点击「项目」—「加载库」—「管理库」，查找「LiquidCrystal_I2C」，选择最新版本进行安装。  
![upload successful](/images/arduino_library.png)


### 显示字符
代码挺简单的：  
```C++
// meng
#include <Wire.h> 
#include <LiquidCrystal_I2C.h> //引用I2C库
 
//设置LCD1602设备地址，这里的地址是0x3F，一般是0x20，或者0x27，具体看模块手册
LiquidCrystal_I2C lcd(0x27, 16, 2);  

void setup()
{
  lcd.init();                  // 初始化LCD
  lcd.backlight();             //设置LCD背景等亮
}
 
void loop()
{
  lcd.setCursor(0,0);                // 设置显示指针
  lcd.print("Pig Love Rabbit");     // 输出字符到LCD1602上
  lcd.setCursor(0,1);
  lcd.print("       by MH.");
  delay(1000);
  lcd.setBacklight(LOW); // 关掉背光 delay(1000);
  delay(1000);  
  lcd.setBacklight(HIGH);
}
```

最终显示效果：![](https://z3.ax1x.com/2021/10/08/5PI7e1.jpg)














参考文章：
* [LCD 1602显示屏](https://www.jianshu.com/p/eee98fb5e68f)