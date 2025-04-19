- 通信方式：串口单线通信
- 数据：
1. 一次完整的数据为40位，高位先出
2. 数据格式位：8位湿度整数数据+8位湿度小数数据+8位温度整数数据+8位温度小数数据+8位校验和
3. 校验和计算方式：校验和为（8bit湿度整数数据+8bit湿度小数数据+8bi温度整数数据+8bit温度小数数据）相加所得结果
- 时序图  
总时序图：
![dht11](https://github.com/user-attachments/assets/cf60ac25-074e-482d-af4d-ddf727c65442)
1.主机请求数据，从机响应时序图
![dht11-2](https://github.com/user-attachments/assets/4d6282c0-10d3-4ea2-be3d-f514ca9bf451)
2. 数据0：
![dht11-3](https://github.com/user-attachments/assets/afde1652-ad79-49da-96ad-8b62d8d1dc6a)
3. 数据1：
![dht11-4](https://github.com/user-attachments/assets/69090348-5987-4ccc-869f-a8bfa8589fe9)
## 代码编写
- 获取引脚状态
```C
uint16_t DHT_Scan()
{
	return HAL_GPIO_ReadPin(dht11_GPIO_Port,dht11_Pin);
}
```
- us微秒级延时函数实现(因为HAL库没有微妙us级别的延时函数)
1. 配置定时器
![dht11-5](https://github.com/user-attachments/assets/3b7214d7-3624-42ef-bfa2-8168de785305)
2. 代码实现
```C
void delay_us(uint16_t nus)
{
  __HAL_TIM_SetCounter(&htim1,0);
  __HAL_TIM_ENABLE(&htim1);//开启定时器
  while(__HAL_TIM_GET_COUNTER(&htim1)<nus);
  __HAL_TIM_DISABLE(&htim1);//关闭定时器
}
```
- DHT输入和输出方向实现
```C
//输出
void DHT11_OUT()
{
  GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.Pin=dht11_Pin;
	GPIO_InitStructure.Mode=GPIO_MODE_OUTPUT_PP;
	GPIO_InitStructure.Pull=GPIO_PULLUP;
	GPIO_InitStructure.Speed=GPIO_SPEED_FREQ_LOW;
	HAL_GPIO_Init(dht11_GPIO_Port,&GPIO_InitStructure);
}
```
```C
//输入
void DHT11_IN()
{
	GPIO_InitTypeDef GPIO_InitStructure;
	GPIO_InitStructure.Pin=dht11_Pin;
	GPIO_InitStructure.Mode=GPIO_MODE_INPUT;
	GPIO_InitStructure.Pull=GPIO_PULLUP;
	GPIO_InitStructure.Speed=GPIO_SPEED_FREQ_LOW;
	HAL_GPIO_Init(dht11_GPIO_Port,&GPIO_InitStructure);
}
```
- 主机请求数据：设置输出模式，主机拉低19ms，再拉高20us，最后设置为输入模式等待DHT11模块的响应
```C
void DHT_Start()
{
  DHT11_OUT();

	HAL_GPIO_WritePin(dht11_GPIO_Port,dht11_Pin,GPIO_PIN_RESET);
  HAL_Delay(19);

  HAL_GPIO_WritePin(dht11_GPIO_Port,dht11_Pin,GPIO_PIN_SET);
	delay_us(20);

  DHT11_IN();
}
```
- 读取一位数据（确定读取值为0还是1）
```C
uint16_t DHT_ReadBit()
{
  while(	while(HAL_GPIO_ReadPin(dht11_GPIO_Port,dht11_Pin)==0);//无论是数据0，还是数据1，都要先等待固定50us的低电平过去

  delay_us(40);//这里的延时40us是用来判断逻辑0还是逻辑1的关键

  //固定的50us低电平过去后，再来判断是逻辑0还是逻辑1
  if(HAL_GPIO_ReadPin(dht11_GPIO_Port,dht11_Pin)==1)
  {
    while(HAL_GPIO_ReadPin(dht11_GPIO_Port,dht11_Pin)==1);//等待这个高电平过去，再返回1，方便下次调用该函数时等待固定的50us低电平过去
    return 1；
  }
  else
    return 0;
}
```
![dht11-6](https://github.com/user-attachments/assets/7b8bce72-a39c-4371-a7d9-b20f46629088)
- 读取一个字节
```C
uint16_t DHT_ReadByte()
{
  uint16_t data=0;
  for(int i=0;i<8;i++)
  {
    data<<=1;//左移1位
    data|=DHT_ReadBit();
  }
  return data;
}
```
- 读取一次温湿度完整数据
```C
uint16_t DHT_ReadData(uint8_t buffer[5])
{
  uint16_t i=0;
  DHT_Start();//主机开始请求数据
  if(DHT_Scan()==RESET)
  {
    while(DHT_Scan()==RESET);//等待从机响应的80us的低电平过去
    while（DHT_Scan()==SET）;//等待从机响应的80us的高电平过去
    
    for(i=0;i<5;i++)
    {
      buffer[i]=DHT_ReadByte();
    }

    while(DHT_Scan()==RESET);//等待从机发送完一整个数据后的结束信号50us的低电平过去
    DHT11_OUT();
    HAL_GPIO_WritePin(dht11_GPIO_Port,dht11_Pin,GPIO_PIN_SET);
    uint8_t check=buffer[0]+buffer[1]+buffer[2]+buffer[3];
    if(check!=buffer[4])
    {
      return 0;//表示数据出错
    }
  }
  return 1；
}
```
