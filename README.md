//----------CONSTANTS----------------------
#define PI 3.14159265359
#define Earth_Radius 6.378e+6
                     
//----------INCLUDES----------------------
#include <stdio.h>
#include <stdlib.h>
#include "tm4c123gh6pm.h"
#include <math.h>
#include "stdint.h"
#include <string.h>

//----------GLOBAL VARIABLES----------------------
char container_of_gps_readings[100];
char test[100] =  "$GPRMC,110618.00,A,3003.71417,N,3120.66051,W,0.078,,030118,,,A*6A\n"; // SANEe ELS3ADA COORDINATES
char latitude[20];
char longitude[20];
char GPS_logName[]="$GPRMC,";
char c;
char STRING_DISTANCE[20];
char valid = 0;

float currentLat = 3003.83607, cuurentLong = 3116.73446; // CREDIT BUILDING
float distance = 0;
float latitude_number = 0; //3003.71417
float longitude_number = 0; //3120.66051

int number_of_comma = 0, i = 0, j = 0, print_counter = 0;
int i_uart = 0;

//----------SystemInit----------------------
void SystemInit() {}

//----------PRINTF----------------------
#define ITM_Port8(n)    (*((volatile unsigned char *)(0xE0000000+4*n)))
#define ITM_Port16(n)   (((volatile unsigned short)(0xE0000000+4*n)))
#define ITM_Port32(n)   (*((volatile unsigned long *)(0xE0000000+4*n)))
#define DEMCR           (*((volatile unsigned long *)(0xE000EDFC)))
#define TRCENA          0x01000000

struct __FILE { int handle; /* Add whatever you need here */ };
FILE __stdout;
FILE __stdin;
int fputc(int ch, FILE *f) {
  if (DEMCR & TRCENA) {
    while (ITM_Port32(0) == 0);
    ITM_Port8(0) = ch;
  }
  return(ch);
}

//----------CALCULATIONS----------------------
float ToRad(float angle){
	int degree = (int)(angle/100);
	
	float minutes = angle - (int)(angle/100)*100;
	//printf("\n ==== %f\n", ((degree+(minutes/60))));
	return ((degree+(minutes/60)) * (PI/180));
}

float GPS_getDistance(float currentLat, float cuurentLong, float destLat, float destLong){
	//Get Radian Angle
	float currentLatRad  = ToRad(currentLat);
	float cuurentLongRad = ToRad(cuurentLong);
	float destLatRad     = ToRad(destLat);
	float destLongRad    = ToRad(destLong);
	
	//Get difference
	float longdiff = destLongRad - cuurentLongRad;
	float latdiff  = destLatRad  - currentLatRad;
	
	//Calculate distance
	float a = pow(sin(latdiff/2),2) + cos(currentLatRad)*cos(destLatRad)*pow(sin(longdiff/2),2);
	double c = 2 * atan2(sqrt(a),(sqrt(1-a)));
	//printf("\n a = %f\n", a);
	//printf("\n c = %f\n", c);
	return Earth_Radius * c ;
}

//----------FLOAT to STRING----------------------
void reverse(char* str, int len) {
    int i = 0, j = len - 1, temp;
    while (i < j) {
        temp = str[i];
        str[i] = str[j];
        str[j] = temp;
        i++;
        j--;
    }
}
 
int intToStr(int x, char str[], int d) {
    int i = 0;
    while (x) {
        str[i++] = (x % 10) + '0';
        x = x / 10;
    }
 
    while (i < d)
        str[i++] = '0';
 
    reverse(str, i);
    str[i] = '\0';
    return i;
}
 
void ftoa(float n, char* res, int afterpoint) {
    int ipart = (int)n;
    float fpart = n - (float)ipart;
    int i = intToStr(ipart, res, 0);
    if (afterpoint != 0) {
        res[i] = '.'; 
        fpart = fpart * pow(10, afterpoint);
 
        intToStr((int)fpart, res + i + 1, afterpoint);
    }
}

//----------UART5----------------------
void UART5Init() {
	SYSCTL_RCGCUART_R |= SYSCTL_RCGCUART_R5;
	SYSCTL_RCGCGPIO_R |= SYSCTL_RCGCGPIO_R4;
	UART5_CTL_R &= ~UART_CTL_UARTEN;
	UART5_IBRD_R = 104;
	UART5_FBRD_R = 11;
	UART5_LCRH_R = (UART_LCRH_WLEN_8 | UART_LCRH_FEN);
	UART5_CTL_R |= (UART_CTL_UARTEN | UART_CTL_RXE | UART_CTL_TXE);
	GPIO_PORTE_AFSEL_R |= 0x30;
	GPIO_PORTE_PCTL_R = (GPIO_PORTE_PCTL_R & ~0x00ff0000) | (GPIO_PCTL_PE4_U5RX | GPIO_PCTL_PE5_U5TX);
	GPIO_PORTE_DEN_R |= 0x30;
	GPIO_PORTE_AMSEL_R = 0;
}

uint8_t UART5_ReadAvailable(void) {
	return ((UART5_FR_R & UART_FR_RXFE) == UART_FR_RXFE) ? 0:1;
}

char UART5_read() {
	while((UART5_ReadAvailable()) != 1);
	return UART5_DR_R & 0xFF;
}

void UART5_write(char c) {
	while((UART5_FR_R & UART_FR_TXFF) != 0);
	UART5_DR_R = c;
}

//----------UART0----------------------
void UART0Init() {
	// 1st give it clk
	SYSCTL_RCGCUART_R |= SYSCTL_RCGCUART_R0;
	SYSCTL_RCGCGPIO_R |= SYSCTL_RCGCGPIO_R0;
	UART0_CTL_R &= ~UART_CTL_UARTEN;
	UART0_IBRD_R = 104;
	UART0_FBRD_R = 11;
	UART0_LCRH_R = (UART_LCRH_WLEN_8 | UART_LCRH_FEN);
	UART0_CTL_R |= (UART_CTL_UARTEN | UART_CTL_RXE | UART_CTL_TXE);
	GPIO_PORTA_AFSEL_R |= 0x03;
	GPIO_PORTA_PCTL_R |= (GPIO_PORTA_PCTL_R & ~0xff) | (GPIO_PCTL_PA0_U0RX | GPIO_PCTL_PA1_U0TX);
	GPIO_PORTA_DEN_R |= 0x03;
}

uint8_t UART0_ReadAvailable(void) {
	return ((UART0_FR_R & UART_FR_RXFE) == UART_FR_RXFE)? 0:1;
}

//func read data from uart
char UART0_read() {
	while((UART0_ReadAvailable()) != 1);
	return UART5_DR_R & 0xFF;
}
void UART0_write(char c) {
	while((UART0_FR_R & UART_FR_TXFF) != 0);
	UART0_DR_R = c;
}

void printStr0(char *str) {
	uint8_t i = 0;
	while(str[i]) {
		UART0_write(*str);
		str++;
	}
}

//----------PORTF----------------------
void PortFInit() {
	SYSCTL_RCGCGPIO_R |=0x20;
	while((SYSCTL_PRGPIO_R & 0x20)==0){};
	GPIO_PORTF_LOCK_R = GPIO_LOCK_KEY;
	GPIO_PORTF_CR_R |= 0X0E;
	GPIO_PORTF_AMSEL_R &= ~0X0E;
	GPIO_PORTF_PCTL_R &= ~0X0000FFF0;
	GPIO_PORTF_AFSEL_R &= ~0X0E;
	GPIO_PORTF_DEN_R |= 0X0E;
	GPIO_PORTF_DIR_R |= 0X0E;
	GPIO_PORTF_DATA_R &=~0X0E;
}

void RGB_set(uint8_t mask) {
	mask &= 0xE;
	GPIO_PORTF_DATA_R |= mask;
}

void RGB_clear(uint8_t mask) {
	mask &= 0xE;
	GPIO_PORTF_DATA_R &= ~mask;
}

int flag_init = 1;

//----------MAIN----------------------
int main(){
	
	// INITIALLIZATIONS
	UART5Init();
	UART0Init();
	PortFInit();

	while (1) {				
		
		c = UART5_read();
		if (c == '$') {
			container_of_gps_readings[0] = '$';
			c = UART5_read();
			if (c == GPS_logName[1]) {
				container_of_gps_readings[1] = c;
				c = UART5_read();
					if (c == GPS_logName[2]) {
						container_of_gps_readings[2] = c ;
						c = UART5_read();
						if (c == GPS_logName[3]) {
							container_of_gps_readings[3] = c ;
							c = UART5_read();
							if (c == GPS_logName[4]) {
								container_of_gps_readings[4] = c ;
								c = UART5_read();
								if (c == GPS_logName[5]) {
									container_of_gps_readings[5] = c ;
									
									c = UART5_read();
									container_of_gps_readings[6] = c;
									
									i_uart = 7;
									while (c != '$') {
										c = UART5_read();
										container_of_gps_readings[i_uart] = c;
										i_uart++;
									}
									
									//-----------print-----------
									printStr0("\33[2K\r");
									printStr0("####### gps STRING\n");
									printStr0("\33[2K\r");
									printStr0(container_of_gps_readings);
									printStr0("\33[2K\r");
									printStr0("\n");
									//---------------------------
															
									number_of_comma = 0;
									i = 0;j=0;
									// get Latitude and longitude STRINGS if VALID
									while (number_of_comma != 2) {
										if (container_of_gps_readings[i] == ',') number_of_comma++;
										if (number_of_comma == 2) {
											i++;
											if (container_of_gps_readings[i] == 'A') {
												i++;
												i++;
												j = 0;
												while (container_of_gps_readings[i] != ',') {
													latitude[j] = container_of_gps_readings[i];
													j++;
													i++;
												}
												i++;
												i++;
												i++;
												j=0;
												while (container_of_gps_readings[i] != ',') {
													longitude[j] = container_of_gps_readings[i];
													j++;
													i++;
												}
												valid = 1;
											}
											else {
												//container_of_gps_readings[] = empty[];
												container_of_gps_readings[0] = '\0';
												latitude[0] = '\0';
												longitude[0] = '\0';
												valid = 0;
											}
										}
										i++;
									}	
									container_of_gps_readings[0] = '\0';
									
									//-----------print-----------
									printStr0("\33[2K\r");
									printStr0("####### valid = ");
									printStr0(&valid);
									printStr0("\n");
									printStr0("\33[2K\r");
									printStr0("\n");
									//---------------------------
									
									if (valid == 1) {			
										
										//-----------print-----------
										printStr0("\33[2K\r");
										printStr0("####### latitude = ");
										printStr0(latitude);
										printStr0("\n");
										printStr0("\33[2K\r");
										printStr0("\n");
										
										printStr0("\33[2K\r");
										printStr0("####### longitude = ");
										printStr0(longitude);
										printStr0("\n");
										printStr0("\33[2K\r");
										printStr0("\n");
										printStr0("\33[2K\r");
										//---------------------------
										
										latitude_number = atof(latitude);
										longitude_number = atof(longitude);
										
										if (flag_init == 1) {
											currentLat = latitude_number;
											cuurentLong = longitude_number;
											flag_init =0;
											//printStr0("onffdfffffffffffffffffffffffffffff\n");
										}
										
										distance += GPS_getDistance(currentLat, cuurentLong, latitude_number, longitude_number);
										
										ftoa(distance, STRING_DISTANCE, 6);
										//-----------print-----------
										printStr0("\33[2K\r");
										printStr0("####### DISTANCE = ");
										printStr0(STRING_DISTANCE);
										printStr0("\n");
										printStr0("\33[2K\r");
										printStr0("\n");
										printStr0("\33[2K\r");
										//---------------------------
										
										currentLat = latitude_number;
										cuurentLong = longitude_number;
										
										if (distance >= 9.8)  { // destination is reached
											RGB_clear(0xE);
											RGB_set(0x8); //green
										}
										else if (distance >= 5 && distance < 9.8) { // 5 meters more to reach your destination
											
											RGB_set(0xA); // blue NOT yellow, yellow = green + red
										}
										else if (distance < 5) { // destination is not reached
											RGB_set(0x2); // red
										}
										
										latitude[0] = '\0';
										longitude[0] = '\0';
											
									}
									
								}
							}
						}
					}
			}
		}
		
	}
}
