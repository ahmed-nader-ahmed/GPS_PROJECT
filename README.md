# int strcmp_by_me(char *a,char *b) {  
		int flag=0;  
		while(*a!='\0' && *b!='\0')  // while loop  
		{  
				if(*a!=*b)  
				{  
						flag=1;  
				}  
				a++;  
				b++;  
		}  
			
		if(flag==0)  
			return 0;  
		else  
			return 1;  
} 


void copy_string(char *target, char *source) {
    while(*source)
    {
        *target = *source;        
        source++;        
        target++;
    }    
    *target = '\0';
}


 
// Reverses a string 'str' of length 'len'
void reverse(char* str, int len)
{
    int i = 0, j = len - 1, temp;
    while (i < j) {
        temp = str[i];
        str[i] = str[j];
        str[j] = temp;
        i++;
        j--;
    }
}
 
// Converts a given integer x to string str[].
// d is the number of digits required in the output.
// If d is more than the number of digits in x,
// then 0s are added at the beginning.
int intToStr(int x, char str[], int d)
{
    int i = 0;
    while (x) {
        str[i++] = (x % 10) + '0';
        x = x / 10;
    }
 
    // If number of digits required is more, then
    // add 0s at the beginning
    while (i < d)
        str[i++] = '0';
 
    reverse(str, i);
    str[i] = '\0';
    return i;
}
 
// Converts a floating-point/double number to a string.
void ftoa(float n, char* res, int afterpoint)
{
    // Extract integer part
    int ipart = (int)n;
 
    // Extract floating part
    float fpart = n - (float)ipart;
 
    // convert integer part to string
    int i = intToStr(ipart, res, 0);
 
    // check for display option after point
    if (afterpoint != 0) {
        res[i] = '.'; // add dot
 
        // Get the value of fraction part upto given no.
        // of points after dot. The third parameter
        // is needed to handle cases like 233.007
        fpart = fpart * pow(10, afterpoint);
 
        intToStr((int)fpart, res + i + 1, afterpoint);
    }
}











int main() {
	UART5Init();
	UART0Init();
	
	
	
	while(1) {	
		
    /*
		char c = UART5_read();
		char container[100] = "";
		if (c == '$') {
			container[0] = '$';
			c = UART5_read();
			if (c == GPS_logName[1]) {
				container[1] = c;
				c = UART5_read();
					if (c == GPS_logName[2]) {
						container[2] = c ;
						c = UART5_read();
						if (c == GPS_logName[3]) {
							container[3] = c ;
							c = UART5_read();
							if (c == GPS_logName[4]) {
								container[4] = c ;
								c = UART5_read();
								if (c == GPS_logName[5]) {
									container[5] = c ;	
									
									printStr0(container);
									c = UART5_read();
									container[6] = c;
									i = 7;
									while (c != '$') {
										printStr0(&c);
										c = UART5_read();
										container[i] = c;
										i++;
									}
								}
							}
						}
					}
			}
			
			
		}
		
		*/
	
		gps_read();
		
		printStr0(rec_arr);
		printStr0(recieved_arr);
		printStr0(enter);
		
		total_dis= total_dis + GPS_getDistance(Start_Long, Start_Lat, currentLong, currentLat);
		
		ftoa(total_dis, d, 7);
		printStr0(distance);
		printStr0(d);
		printStr0(enter);
		//printStr0(&c);
		
		
		Start_Long = currentLong;
		Start_Lat = currentLat;

		
			
		
	}
}
