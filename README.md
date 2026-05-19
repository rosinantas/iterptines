<img width="1012" height="686" alt="Screenshot 2026-05-19 010207" src="https://github.com/user-attachments/assets/0dbc278a-3d95-4b63-a8fe-33cad3c5c7d8" />
<img width="797" height="752" alt="Screenshot 2026-05-19 010258" src="https://github.com/user-attachments/assets/b9d6e9e6-a040-4ccb-b8e9-2dfdbad4deab" />
<img width="1232" height="690" alt="Screenshot 2026-05-19 010232" src="https://github.com/user-attachments/assets/3b9a8e45-7f78-4de3-8f74-2ccab2ed4266" />

```c
/* Includes ------------------------------------------------------------------*/ 
#include "main.h" 
#include "spi.h" 
#include "tim.h" 
#include "usart.h" 
#include "gpio.h" 
/* USER CODE BEGIN Includes */ 
#include "ssd1306.h"        
// OLED ekrano biblioteka 
#include "ssd1306_fonts.h"  // OLED ekrano šriftai 
#include "DHT.h"            
// DHT jutikliu biblioteka 
#include <stdio.h>          
/* USER CODE END Includes */ 
/* USER CODE BEGIN PV */ 
DHT_t dht22;                    
DHT_t dht11;                    
bool dht22_ok = false;          
bool dht11_ok = false;          
// sprintf funkcijai 
// DHT22 (AM2301) jutiklio struktura 
// DHT11 jutiklio struktura 
// ar pavyko nuskaityt sensoriu 
// ar pavyko nuskaityt sensoriu 
bool dht11_initialized = false; // ar DHT11 filtras jau inicializuotas 
bool dht22_initialized = false; // ar DHT22 filtras jau inicializuotas 
/* USER CODE END PV */ 
/* USER CODE BEGIN 0 */ 
// UART klaidos apdorojimo funkcija 
void HandleError()  
{  
uint32_t uart_err;  
uart_err = HAL_UART_GetError(&huart2); // nuskaito klaidos koda 
}  
// Išorinio pertraukimo (EXTI) callback - iškvieciamas kai keiciasi GPIO busena 
// Naudojamas DHT jutikliu duomenu nuskaitymui per laikmacio pertraukimus 
void HAL_GPIO_EXTI_Callback(uint16_t GPIO_Pin) 
{ 
if(GPIO_Pin == dht11.pin) 
DHT_pinChangeCallBack(&dht11); // perduoda pertraukima DHT11 bibliotekai 
if(GPIO_Pin == dht22.pin) 
DHT_pinChangeCallBack(&dht22); // perduoda pertraukima DHT22 bibliotekai 
} 
/* USER CODE END 0 */ 
int main(void) 
{ 
// Inicializuoja HAL biblioteka ir sistemos laikmati 
HAL_Init(); 
// Sukonfiguruoja sistemos taktini dažni  
SystemClock_Config(); 
// Inicializuoja visus sukonfiguruotus periferinius irenginius 
MX_GPIO_Init();         
MX_SPI1_Init();         
MX_TIM2_Init();         
MX_USART2_UART_Init();  
/* USER CODE BEGIN 2 */ 
// OLED ekrano maitinimo reset seka 
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_RESET); 
HAL_Delay(100); 
HAL_GPIO_WritePin(GPIOB, GPIO_PIN_0, GPIO_PIN_SET); 
HAL_Delay(100); 
// Inicializuoja ir išvalo OLED ekrana 
ssd1306_Init(); 
ssd1306_Fill(Black); 
ssd1306_UpdateScreen(); 
// Inicializuoja DHT jutiklius  
DHT_init(&dht22, DHT_Type_AM2301, &htim2, 72, GPIOB, GPIO_PIN_12); 
DHT_init(&dht11, DHT_Type_DHT11,  &htim2, 72, GPIOB, GPIO_PIN_13); 
/* USER CODE END 2 */ 
HAL_Delay(100); 
// Laikmatininkai - saugo paskutinio atnaujinimo laika (ms) 
uint32_t last_dht11 = 0; 
uint32_t last_dht22 = 0; 
uint32_t last_oled  = 0; 
uint32_t last_uart  = 0; 
// Neapdoroti jutikliu rodmenys 
float t2 = 0, h2 = 0; // DHT22 temperatura ir dregme 
float t1 = 0, h1 = 0; // DHT11 temperatura ir dregme 
// IIR (EMA) filtro išejimo kintamieji - filtruoti (vidurkinki) rodmenys 
float t2_f = 0, h2_f = 0; // DHT22 filtruoti rodmenys 
float t1_f = 0, h1_f = 0; // DHT11 filtruoti rodmenys 
// IIR filtro koeficientai  
float alpha_dht11 = 0.06f; 
float alpha_dht22 = 0.11f; 
// Laukiama kol jutikliai stabilizuojasi po ijungimo 
HAL_Delay(3000); 
while (1) 
{ 
uint32_t now = HAL_GetTick(); // dabartinis laikas milisekundemis 
//  DHT11 nuskaitymas kas 1s (pagal datasheeta)  
if(now - last_dht11 >= 1000) 
{ 
last_dht11 = now; 
dht11_ok = DHT_readData(&dht11, &t1, &h1); // nuskaito dregme ir 
temperatura 
dydžiu 
} 
if(dht11_ok) 
{ 
if(!dht11_initialized) 
{ 
// Pirma karta - inicializuoja filtra tiesiogiai nuskaitytu 
h1_f = h1; 
dht11_initialized = true; 
} 
else 
{ 
} 
} 
// IIR filtras 
h1_f = h1_f + alpha_dht11 * (h1 - h1_f); 
// DHT22 nuskaitymas kas 2s (pagal datasheeta) 
if(now - last_dht22 >= 2000) 
{ 
last_dht22 = now; 
dht22_ok = DHT_readData(&dht22, &t2, &h2); // nuskaito dregme ir 
temperatura 
dydžiu 
} 
if(dht22_ok) 
{ 
if(!dht22_initialized) 
{ 
// Pirma karta - inicializuoja filtra tiesiogiai nuskaitytu 
h2_f = h2; 
dht22_initialized = true; 
} 
else 
{ 
} 
} 
// IIR filtras 
h2_f = h2_f + alpha_dht22 * (h2 - h2_f); 
// OLED ekrano atnaujinimas kas 10s  
if(now - last_oled >= 10000) 
{ 
last_oled = now; 
ssd1306_Fill(Black); // išvalo ekrana 
// DHT22 dregmes rodymas  
if(dht22_ok) 
{ 
char hStr[32]; 
int h2_whole = (int)h2_f; 
int h2_frac  = abs((int)(h2_f * 10) % 10); 
sprintf(hStr, "H:%d.%d%%", h2_whole, h2_frac); 
ssd1306_SetCursor(0, 11); 
// Jei dregme už 30-70% ribu - rodo ispejima 
if(h2_f < 30.0f || h2_f > 70.0f) 
ssd1306_WriteString("Uz 30_70 proc ribu", Font_7x10, White); 
else 
} 
else 
{ 
ssd1306_WriteString(hStr, Font_7x10, White); 
// Jutiklio klaida 
ssd1306_SetCursor(0, 0); 
ssd1306_WriteString("Sensor Error", Font_11x18, White); 
} 
// DHT11 dregmes rodymas  
if(dht11_ok) 
{ 
char hStr[32]; 
int h1_whole = (int)h1_f; 
int h1_frac  = abs((int)(h1_f * 10) % 10); 
sprintf(hStr, "H:%d.%d%%", h1_whole, h1_frac); 
ssd1306_SetCursor(0, 33); 
// Jei dregme už 30-70% ribu - rodo ispejima 
if(h1_f < 30.0f || h1_f > 70.0f) 
ssd1306_WriteString("Uz 30_70 proc ribu", Font_7x10, White); 
else 
} 
else 
{ 
ssd1306_WriteString(hStr, Font_7x10, White); 
// Jutiklio klaida 
ssd1306_SetCursor(0, 22); 
ssd1306_WriteString("Sensor Error", Font_11x18, White); 
} 
// Kanalu skirtumo skaiciavimas ir rodymas 
if(dht11_ok && dht22_ok) 
{ 
float h_diff = h1_f - h2_f; // skirtumas tarp jutikliu 
char hvidStr[32]; 
int h_diff_whole = (int)h_diff; 
int h_diff_frac  = abs((int)(h_diff * 10) % 10); 
//neigiamas skirtumas kai sveikoji dalis = 0 
if(h_diff < 0 && h_diff_whole == 0) 
sprintf(hvidStr, "Skirt:-%d.%d%%", h_diff_whole, h_diff_frac); 
else 
sprintf(hvidStr, "Skirt:%d.%d%%", h_diff_whole, h_diff_frac); 
ssd1306_SetCursor(0, 44); 
ssd1306_WriteString(hvidStr, Font_7x10, White); 
} 
else 
{ 
} 
ssd1306_UpdateScreen();  
} 
// UART duomenu siuntimas i kompiuteri kas 1s  
if(now - last_uart >= 1000) 
{ 
last_uart = now; 
char uartBuf[80]; 
char h1_str[20], h2_str[20], diff_str[20]; 
// DHT11 duomenu formatavimas 
if(dht11_ok) 
{ 
if(h1_f < 30.0f || h1_f > 70.0f) 
sprintf(h1_str, "PERZENGTA 30-70%%"); // ispejimas už ribu 
else 
{ 
} 
} 
else 
int h1_w  = (int)h1_f; 
int h1_fr = abs((int)(h1_f * 10) % 10); 
sprintf(h1_str, "%d.%d%%", h1_w, h1_fr); 
sprintf(h1_str, "ERROR"); 
// DHT22 duomenu formatavimas 
if(dht22_ok) 
{ 
if(h2_f < 30.0f || h2_f > 70.0f) 
sprintf(h2_str, "PERZENGTA 30-70%%"); // ispejimas už ribu 
else 
{ 
} 
} 
else 
int h2_w  = (int)h2_f; 
int h2_fr = abs((int)(h2_f * 10) % 10); 
sprintf(h2_str, "%d.%d%%", h2_w, h2_fr); 
sprintf(h2_str, "ERROR"); 
// Skirtumo tarp kanalu formatavimas 
if(dht11_ok && dht22_ok) 
{ 
float diff   = h1_f - h2_f; 
int diff_w   = (int)diff; 
int diff_fr  = abs((int)(diff * 10) % 10); 
// Specialus atvejis: neigiamas skirtumas kai sveikoji dalis = 0 
if(diff < 0 && diff_w == 0) 
ssd1306_SetCursor(0, 44); 
ssd1306_WriteString("Sensor Error", Font_11x18, White); 
sprintf(diff_str, "-%d.%d%%", diff_w, diff_fr); 
else 
sprintf(diff_str, "%d.%d%%", diff_w, diff_fr); 
} 
else 
sprintf(diff_str, "ERROR"); 
// Suformuoja ir siuncia UART eilute i kompiuteri 
sprintf(uartBuf, "H1:%s H2:%s Skirt:%s\r\n", h1_str, h2_str, diff_str); 
HAL_UART_Transmit(&huart2, (uint8_t*)uartBuf, strlen(uartBuf), 100); 
} 
} 
}
```
