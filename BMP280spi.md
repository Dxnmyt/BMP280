## STM32F10x 平台软件 SPI 驱动程序 (myspi.c)

这段代码实现了一个基于 GPIO 模拟 SPI 功能的软件 SPI 驱动程序，用于 STM32F10x 系列微控制器。由于某些应用场景可能需要使用软件 SPI 或者硬件 SPI 资源已被占用，此驱动提供了一种灵活的 SPI 通信方案。该驱动程序定义了 SPI 通信中常用的 GPIO 引脚（CS, MOSI, MISO, SCK），并提供了以下关键功能：

*   **初始化 (myspi\_Init):** 配置 SPI 通信所需的 GPIO 引脚，包括 CS (片选), MOSI (主机输出/从机输入), MISO (主机输入/从机输出), SCK (时钟) 引脚的模式和速度。
*   **片选控制 (SPI\_CS\_Enable, SPI\_CS\_Disable):**  通过控制 CS 引脚的电平来使能和禁用 SPI 通信，实现与 SPI 从设备的片选。
*   **时钟控制 (SPI\_SCK\_Set):**  允许手动设置 SCK 时钟线的电平，用于控制 SPI 时序。
*   **数据线控制 (SPI\_MOSI\_Set):**  控制 MOSI 数据线的电平，用于向 SPI 从设备发送数据。
*   **数据读取 (SW\_SPI\_MISO\_Get):**  读取 MISO 数据线上的电平，用于从 SPI 从设备接收数据。
*   **字节写入 (SPI\_writebyte):**  向 SPI 总线写入一个字节的数据，数据通过 MOSI 线发送，并根据 SPI 时序产生 SCK 时钟。
*   **字节读取 (SPI\_readbyte):**  从 SPI 总线读取一个字节的数据，同时也可以发送一个字节的数据（通常为哑数据 0xFF），在 SCK 时钟的驱动下，从 MISO 线接收数据。

此软件 SPI 驱动程序为用户提供了基础的 SPI 通信功能，用户可以基于此驱动程序构建更高级别的 SPI 设备驱动，例如下文的 BMP280 驱动。

```c
#include "stm32f10x.h"                  // Device header
#include "Delay.h"

#define CS GPIO_Pin_4
#define MOSI GPIO_Pin_7
#define MISO GPIO_Pin_6
#define SCK GPIO_Pin_5

// spi模式0

void myspi_Init(void)
{
	RCC_APB2PeriphClockCmd(RCC_APB2Periph_GPIOA, ENABLE);

	GPIO_InitTypeDef gpio_InitStructure;
	gpio_InitStructure.GPIO_Mode = GPIO_Mode_Out_PP;  //推挽输出
	gpio_InitStructure.GPIO_Pin = CS | SCK |MOSI;
	gpio_InitStructure.GPIO_Speed = GPIO_Speed_50MHz;
	GPIO_Init(GPIOA, &gpio_InitStructure);

	gpio_InitStructure.GPIO_Mode = GPIO_Mode_IPU; //主机输入模式
	gpio_InitStructure.GPIO_Pin = MISO;
	GPIO_Init(GPIOA, &gpio_InitStructure);
}

void SPI_CS_Enable(void) //拉低ss开始通信
{
	GPIO_ResetBits(GPIOA, CS);
}

void SPI_CS_Disable(void) //拉高ss结束通信
{
	GPIO_SetBits(GPIOA, CS);
}

void SPI_SCK_Set(uint8_t level) //控制时钟线的高低
{
	if(level == 0)
	{
		GPIO_ResetBits(GPIOA, SCK);
	}
	else
	{
		GPIO_SetBits(GPIOA, SCK);
	}
}

void SPI_MOSI_Set(uint8_t level)  //控制MOSI的写入
{
	if(level == 0)
	{
		GPIO_ResetBits(GPIOA, MOSI);
	}
	else
	{
		GPIO_SetBits(GPIOA, MOSI);
	}
}

uint8_t SW_SPI_MISO_Get(void)  //读取MISO的数据
{
	return GPIO_ReadInputDataBit(GPIOA, MISO);
}

void SPI_writebyte(uint8_t data)  //默认MISO
{
	uint8_t current;  //中间量
	for(int i = 7; i >= 0; i--)
	{
		current = (data >> i) & 0x01;  //控制MOSI的写入，右移写入 例：10110111 右移7位与为00000001 & 00000001
		SPI_MOSI_Set(current);
		SPI_SCK_Set(1);  //拉高时钟线
		Delay_us(1);  //延时交换数据
		SPI_SCK_Set(0);  //拉低
		Delay_us(1);  //延时
	}
}

uint8_t SPI_readbyte(uint8_t data)
{
	uint8_t receivedata = 0;
	for(int i = 7; i >= 0; i--)
	{
		uint8_t current = (data >> i) & 0x01;
		SPI_MOSI_Set(current);
		SPI_SCK_Set(1);  //需要给MOSI一个数据

		if(SW_SPI_MISO_Get())
		{
			receivedata |= (1 << i);  //循环读取数据，左移到最高位或运算获得数据
		}

		Delay_us(1);

		SPI_SCK_Set(0);

		Delay_us(1);
	}

	return receivedata;  //返回数据
}
```

## 基于软件 SPI 的 BMP280 驱动程序 (bmp280.c)

这段代码是基于上述 `myspi.c` 软件 SPI 驱动程序编写的 BMP280 压力和温度传感器驱动。它利用软件 SPI 与 BMP280 传感器进行通信，并提供了读取温度和压力数据的接口。该驱动程序包含了以下关键功能：

*   **初始化 (bmp280\_Init):**  调用 `myspi_Init()` 函数，初始化软件 SPI 驱动。
*   **数据读取 (redata):**  通过软件 SPI 从 BMP280 传感器读取指定长度的数据。该函数负责设置读取地址，发送读取命令，并接收传感器返回的数据。
*   **数据写入 (wridata):**  通过软件 SPI 向 BMP280 传感器写入一个字节的数据到指定的寄存器地址。
*   **读取校准数据 (BMP280\_ReadCalibration):**  从 BMP280 传感器读取校准参数，这些参数存储在传感器的特定寄存器中，用于后续的温度和压力补偿计算。
*   **配置传感器 (BMP280\_Config):**  配置 BMP280 传感器的工作模式和采样率等参数，例如设置为正常模式和过采样率。
*   **读取原始数据 (BMP280\_ReadRawData):**  从 BMP280 传感器读取未经处理的温度和压力原始数据。
*   **温度补偿 (bmp280\_T):**  使用从传感器读取的校准参数，对原始温度数据进行补偿计算，得到精确的温度值（摄氏度）。
*   **压力补偿 (bmp280\_P):**  使用校准参数和已补偿的温度值，对原始压力数据进行补偿计算，得到精确的压力值（帕斯卡）。

总而言之，这段代码提供了一个基于软件 SPI 的 BMP280 传感器驱动，依赖于 `myspi.c` 提供的软件 SPI 基础功能，实现了 BMP280 传感器的完整驱动，方便用户在 STM32F10x 平台上使用 SPI 方式读取 BMP280 的温度和压力数据。

```c
#include "stm32f10x.h"                  // Device header
#include "myspi.h"

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

void bmp280_Init(void)
{
	myspi_Init();
}

void redata(uint8_t regaddress, uint8_t len, uint8_t *data)
{
	SPI_CS_Enable();  //在这控制CS线
	SPI_SCK_Set(0);

	uint8_t readAddr = regaddress | 0x80;  //spi最高位为1的时候是读取数据，其他七位是地址不影响

	SPI_writebyte(readAddr);

	for(int i = 0; i < len; i++)
	{
		data[i] = SPI_readbyte(0xFF);  //定义len长度进行读取数据
	}

	SPI_CS_Disable();
}

void wridata(uint8_t reg, uint8_t data)
{
	SPI_CS_Enable();
	SPI_SCK_Set(0);
	SPI_writebyte(reg & 0x7F);  //最高位为0写入数据
	SPI_writebyte(data);
	SPI_CS_Disable();
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














