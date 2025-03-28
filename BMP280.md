```markdown
## STM32F10x 平台的 BMP280 传感器驱动程序

这段代码提供了一个在 STM32F10x 微控制器上驱动 BMP280 压力和温度传感器的程序。它使用 I2C2 外设与传感器进行通信。该驱动程序包含了以下功能函数：

*   **初始化 (Initialization):**  配置 I2C2 外设以及 BMP280 传感器所需的 GPIO 引脚。
*   **读取校准数据 (Read Calibration Data):** 从 BMP280 传感器读取校准参数，这些参数用于后续的温度和压力补偿计算。
*   **配置传感器 (Configure Sensor):**  设置 BMP280 传感器的工作模式和采样率。
*   **读取原始数据 (Read Raw Data):**  从传感器读取未经处理的温度和压力数据。
*   **温度补偿 (Temperature Compensation):**  使用校准参数将原始温度数据转换为精确的温度值。
*   **压力补偿 (Pressure Compensation):** 使用校准参数和已补偿的温度值，将原始压力数据转换为精确的压力值。

总而言之，这段代码为 STM32F10x 用户提供了一个完整的 BMP280 传感器驱动，方便用户读取精确的温度和压力数据。

```c
#include "stm32f10x.h"                  // Device header


#define BMP280_ADDR 0x77
int32_t t_fine;


typedef struct
{
	uint16_t dig_T1;
	int16_t dig_T2, dig_T3;
	uint16_t dig_P1;
	int16_t dig_P2, dig_P3, dig_P4,
	dig_P5, dig_P6, dig_P7, dig_P8, dig_P9;
}
bmp280_prot;  //定义结构体

bmp280_prot port;

void myi2c_Init(void)
{
	RCC_APB1PeriphClockCmd(RCC_APB1Periph_I2C2, ENABLE);  //开启时钟
	RCC_APB1PeriphClockCmd(RCC_APB2Periph_GPIOB, ENABLE);

	GPIO_InitTypeDef GPIO_INITSTRUCTURE;
	GPIO_INITSTRUCTURE.GPIO_Mode = GPIO_Mode_AF_OD;  //i2c配置为复用开漏
	GPIO_INITSTRUCTURE.GPIO_Pin = GPIO_Pin_10 | GPIO_Pin_11;
	GPIO_INITSTRUCTURE.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOB, &GPIO_INITSTRUCTURE);


	I2C_InitTypeDef I2C_InitStructure;
	I2C_InitStructure.I2C_Ack = I2C_Ack_Enable;
	I2C_InitStructure.I2C_AcknowledgedAddress = I2C_AcknowledgedAddress_7bit;  //7位地址
	I2C_InitStructure.I2C_ClockSpeed = 50000;
	I2C_InitStructure.I2C_DutyCycle = I2C_DutyCycle_2;
	I2C_InitStructure.I2C_Mode = I2C_Mode_I2C;
	I2C_InitStructure.I2C_OwnAddress1 = 0x00;
	I2C_Init(I2C2, &I2C_InitStructure);

	I2C_Cmd(I2C2, ENABLE);
}

void redata(uint8_t regaddress, uint8_t len, uint8_t *data)  //i2c读
{
	I2C_GenerateSTART(I2C2, ENABLE); //开启
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_MODE_SELECT)); //等待EV5事件

	I2C_Send7bitAddress(I2C2, BMP280_ADDR << 1, I2C_Direction_Transmitter);  //指定地址
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));  //等待EV6事件

	I2C_SendData(I2C2, regaddress);  //发送数据
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_BYTE_TRANSMITTED)); //结束等待EV8_2事件

	I2C_GenerateSTART(I2C2, ENABLE);  //重新开始
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_MODE_SELECT));  //等待EV5事件

	I2C_Send7bitAddress(I2C2, BMP280_ADDR << 1, I2C_Direction_Receiver);  //指定地址
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_RECEIVER_MODE_SELECTED));  //变为读

	for(int i = 0; i < len; i++)  //用来读取多个字节
	{
		if(i == len - 1)
			I2C_AcknowledgeConfig(I2C2, DISABLE);  //在最后一个字节之前需要停止应答
		while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_BYTE_RECEIVED));
		data[i] = I2C_ReceiveData(I2C2);
	}

	I2C_GenerateSTOP(I2C2, ENABLE);  //然后停止
	I2C_AcknowledgeConfig(I2C2, ENABLE);  //启动应答方便下次传输
}

void wridata(uint8_t reg, uint8_t data)
{
	I2C_GenerateSTART(I2C2, ENABLE);
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_MODE_SELECT));

	I2C_Send7bitAddress(I2C2, BMP280_ADDR << 1, I2C_Direction_Transmitter);
	while(!I2C_CheckEvent(I2C2,I2C_EVENT_MASTER_TRANSMITTER_MODE_SELECTED));

	I2C_SendData(I2C2, reg);
	while(!I2C_CheckEvent(I2C2, I2C_EVENT_MASTER_BYTE_TRANSMITTING));

	I2C_SendData(I2C2, data);
	while(!I2C_CheckEvent(I2C2, I2C_EVENT_MASTER_BYTE_TRANSMITTED));

	I2C_GenerateSTOP(I2C2, ENABLE);
}


void BMP280_ReadCalibration(void)
{
	uint8_t data[24];
	redata(0x88, 24, data);

	port.dig_T1 = (data[1] << 8) | data[0];
	port.dig_T2 = (data[3] << 8) | data[2];
	port.dig_T3 = (data[5] << 8) | data[4];
	port.dig_P1 = (data[7] << 8) | data[6];
	port.dig_P2 = (data[9] << 8) | data[8];
	port.dig_P3 = (data[11] << 8) | data[10];
	port.dig_P4 = (data[13] << 8) | data[12];
	port.dig_P5 = (data[15] << 8) | data[14];
	port.dig_P6 = (data[17] << 8) | data[16];
	port.dig_P7 = (data[19] << 8) | data[18];
	port.dig_P8 = (data[21] << 8) | data[20];
	port.dig_P9 = (data[23] << 8) | data[22];

}


void BMP280_Config(void)
{
	wridata(0xF4, 0x27);  //开启正常模式以及采样率为1
	wridata(0xF5, 0x00);
}

void BMP280_ReadRawData(int32_t *temp, int32_t *pressure)
{
	uint8_t data[6];
	redata(0xF7, 6, data);

	*pressure = ((uint32_t)data[0] << 12) | ((uint32_t)data[1] << 4) | ((uint32_t)data[2] >> 4);
	*temp = ((uint32_t)data[3] << 12) | ((uint32_t)data[4] << 4) | ((uint32_t)data[5] >> 4);
}

float bmp280_T(int32_t tep)
{
	int32_t var1, var2, T;
	var1 = (((tep >> 3) - ((int32_t)port.dig_T1 << 1)) * ((int32_t)port.dig_T2)) >> 11;
	var2 = (((((tep >> 4) - ((int32_t)port.dig_T1)) * ((tep >> 4) - ((int32_t)port.dig_T1))) >> 12) * ((int32_t)port.dig_T3)) >> 14;
	t_fine = var1 + var2;
	T = (t_fine * 5 + 128) >> 8;
	return T / 100.0f;
}

float bmp280_P(int32_t pre)
{
	int64_t var1, var2, p;
	var1 = ((int64_t)t_fine) - 128000;
	var2 = var1 * var1 * (int64_t)port.dig_P6;
	var2 = var2 + ((var1 * (int64_t)port.dig_P5) << 17);
	var2 = var2 + (((int64_t)port.dig_P4) << 35);
	var1 = ((var1 * var1 * (int64_t)port.dig_P3) >> 8) + ((var1 * (int64_t)port.dig_P2) << 12);
	var1 = (((((int64_t)1) << 47) + var1)) * ((int64_t)port.dig_P1) >> 33;
	if(var1 == 0)
	{
		return 0;
	}

	p = 1048576 - pre;
	p = (((p << 31) - var2) * 3125) / var1;
	var1 = (((int64_t)port.dig_P9) * (p >> 13) * (p >> 13)) >> 25;
	var2 = (((int64_t)port.dig_P8) * p) >> 19;
	p = ((p + var1 + var2) >> 8) + (((int64_t)port.dig_P7) << 4);
	return (float)p / 256.0f;
}
```
