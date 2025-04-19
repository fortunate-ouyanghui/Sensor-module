- 工作电压
![MQ-2(1)](https://github.com/user-attachments/assets/edd5d948-f12d-451a-90e0-f1dcead13e93)
- 引脚说明
1. AO引脚输出模拟信号
2. D0引脚输出高低电平，超过设定阈值时，输出低电平；低于设定阈值时，输出高电平
- 注意：传感器通电后，需要先预热约60s后测量的数据才稳定。通电后传感器会出现正常的轻度发热现象，因为内部有电热丝。
## 代码编写
- 获取AO的模拟量
```C
//ADC校准写在setup（）中
//获取模拟量
uint16_t MQ2_ADC_Read()
{
	HAL_ADC_Start(&hadc1);
	if(HAL_ADC_PollForConversion(&hadc1,HAL_MAX_DELAY)==HAL_OK)
	{
		HAL_ADC_Stop(&hadc1);
		return HAL_ADC_GetValue(&hadc1);
	}
}

//每10次数据求平均值
uint16_t MQ2_GetData()
{
	uint32_t tempData=0;
	for(int i=0;i<MQ2_READ_TIMES;i++)//MQ2_READ_TIMES为10，表示每10个数据取平均值
	{
		tempData+=MQ2_ADC_Read();
		HAL_Delay(5);
	}
	tempData/=MQ2_READ_TIMES;
	return tempData;
}
```
- 气体浓度ppm的计算
```C
//将ADC原始值 转换为 气体浓度
float MQ2_GetData_PPM()
{
	float tempData=0;
	tempData=MQ2_GetData();
	
	//计算传感器电阻 RS
	float vol=(tempData*5)/4096;
	float RS=(5-vol)/(vol*0.5);
	
	//计算 R0（洁净空气中的传感器电阻）
	float R0=6.64;
	
	//通过经验公式计算ppm
	float ppm=pow(11.5428*R0/RS,0.6549f);

	return ppm;
}
```
![MQ-2(2)](https://github.com/user-attachments/assets/c7d3f764-9838-44f9-9854-30ec8bd4b24e)
![MQ-2(3)](https://github.com/user-attachments/assets/9c368e85-ae61-40c0-ae84-cc1907cf6e9a)
![MQ-2(6)](https://github.com/user-attachments/assets/e59e39c9-efb4-472e-8443-6ac35a1498e3)



