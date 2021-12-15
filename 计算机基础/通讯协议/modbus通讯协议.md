# Modbus通讯协议

## 简介

- Modbus 协议是应用于电子控制器上的一种通用语言

- Modbus 协议定义了一个控制器能认识使用的消息结构,而不管它们是经过何种网络进行通信的。它描述了一控制器请求访问其它设备的过程，如果回应来自其它设备的请求，以及怎样侦测错误并记录。它制定了消息域格局和内容的**公共格式**

- modbus是一个 请求/应答协议


## 示例
<table>
    <tr>
        <td colspan= "15">ADU 应用数据单元</td>
    </tr>
    <tr>
        <td colspan= "3">地址域</td>
        <td colspan= "3">功能码</td>
        <td colspan= "3">数据地址/长度</td>
        <td colspan= "3">数据</td>
        <td colspan= "3">差错校验</td>
    </tr>
    <tr>
        <td colspan= "3"></td>
        <td colspan= "9">PDU 协议数据单元</td>
        <td colspan= "3"></td>
    </tr>
    <tr>
        <td colspan= "3">01</td>
        <td colspan= "3">03</td>
        <td colspan= "3">02</td>
        <td colspan= "3">01 68</td>
        <td colspan= "3">7A 6D</td>
    </tr>
</table>

简单分析一下上表中最后一行

- 从机`01`通过响应`03`功能码，把数据长度为`02`的数据 `01 68`发送出去
- 上面的这条数据可以称为一个数据帧或一个报文
- CRC校验，防止在传输过程中数据错误

