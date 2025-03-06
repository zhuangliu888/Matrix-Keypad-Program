# Matrix-Keypad-Program
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
树莓派5 - 矩阵键盘(4x4)示例
事件触发 + 简易扫描
功能：
1. 连续按下 '1','2','3','4' 视为解锁
2. 连续按下 'A','B','C','D' 视为上锁
"""

import RPi.GPIO as GPIO
import time

# -------------------------
# 1. 硬件引脚定义（需根据实际电路修改）
# -------------------------
# 行(Row)引脚：输出模式
rowPins = [4, 17, 27, 22]
# 列(Col)引脚：输入模式
colPins = [5, 6, 13, 19]

# 4x4矩阵键盘对应的按键布局
keyMap = [
    ['1', '2', '3', 'A'],
    ['4', '5', '6', 'B'],
    ['7', '8', '9', 'C'],
    ['*', '0', '#', 'D']
]

# -------------------------
# 2. 密码定义
# -------------------------
password_unlock = "1234"  # 解锁密码
password_lock   = "ABCD"  # 上锁密码

# 存储按键输入的缓冲区
input_buffer = ""

# -------------------------
# 3. 初始化GPIO
# -------------------------
def setup():
    GPIO.setmode(GPIO.BCM)  # 使用BCM编号
    # 将行Pins设置为输出，初始拉低
    for rp in rowPins:
        GPIO.setup(rp, GPIO.OUT, initial=GPIO.LOW)
    # 将列Pins设置为输入，下拉电阻，并添加事件检测
    for cp in colPins:
        GPIO.setup(cp, GPIO.IN, pull_up_down=GPIO.PUD_DOWN)
        # 当列线出现上升沿时触发回调（bouncetime做简单消抖）
        GPIO.add_event_detect(cp, GPIO.RISING, callback=column_callback, bouncetime=200)

# -------------------------
# 4. 列线事件回调函数
# -------------------------
def column_callback(channel):
    """
    当检测到某列线变为高电平时(表示有按键按下)，调用此回调。
    通过简易扫描rowPins以确定是哪一个行被按下。
    """
    # 找到是哪一个列脚触发了事件
    col_index = colPins.index(channel)

    # 简单延时消抖
    time.sleep(0.05)
    # 再次确认该列是否仍为高电平
    if GPIO.input(channel) == GPIO.HIGH:
        # 扫描行，以确定按下的是哪一行
        pressed_key = scan_key(col_index)
        if pressed_key is not None:
            handle_key(pressed_key)

# -------------------------
# 5. 简易扫描函数
# -------------------------
def scan_key(col_index):
    """
    将每个rowPins轮流拉高，判断colPins[col_index]是否为高，
    若为高，则表示该行rowIndex与colIndex对应的按键被按下。
    """
    for row_index in range(4):
        # 将该行置为高电平
        GPIO.output(rowPins[row_index], GPIO.HIGH)
        # 稍作延时，等待电平稳定
        time.sleep(0.001)
        # 判断对应列脚是否也为高
        if GPIO.input(colPins[col_index]) == GPIO.HIGH:
            # 读取到有效按键
            GPIO.output(rowPins[row_index], GPIO.LOW)
            return keyMap[row_index][col_index]
        # 将行拉低
        GPIO.output(rowPins[row_index], GPIO.LOW)
    return None

# -------------------------
# 6. 按键处理逻辑
# -------------------------
def handle_key(key):
    global input_buffer
    # 将当前按键加入缓冲区
    input_buffer += key
    print("按下的按键:", key)

    # 检查是否解锁
    if input_buffer.endswith(password_unlock):
        print("解锁成功！")
        # 在此处执行解锁操作，如点亮LED或控制继电器
        input_buffer = ""  # 清空缓冲区

    # 检查是否上锁
    if input_buffer.endswith(password_lock):
        print("上锁成功！")
        # 在此处执行上锁操作
        input_buffer = ""  # 清空缓冲区

# -------------------------
# 7. 主循环
# -------------------------
def loop():
    try:
        while True:
            # 主循环中不做轮询，等待事件触发
            time.sleep(1)
    except KeyboardInterrupt:
        print("程序结束")
    finally:
        GPIO.cleanup()

# -------------------------
# 8. 程序入口
# -------------------------
if __name__ == '__main__':
    setup()
    loop()
