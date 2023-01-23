# auto-filling

//Connect Trigger to RB2 and Echo to RB1
unsigned int T1overflow;
unsigned long T1counts;
unsigned long T1time;
unsigned long Distance;
unsigned int Dcntr;
unsigned int cup_size=0;// THE CUP SIZE
unsigned int cup_cntr=0;// cntr for every time a new fill require
unsigned int lcd_cntr=0;
unsigned char vc=0;
unsigned int x[2];// the result of converting Distance from int to string for lcd out
unsigned char y[1];
unsigned char j=0;
unsigned char i=0;
unsigned int water_level;// cup in percantage
unsigned char temparray[10];
void usDelay(unsigned int);
void msDelay(unsigned int);

unsigned char cntr=0;
void manual(void);
void init_sonar(void);
void read_sonar(void);
void full_cup(void);// full cup function
void quar_cup(void);//quartier cup function
void half_cup(void);// half cup


void interrupt(void){
 if(INTCON&0x04){// will get here every 1ms
    TMR0=248;
    Dcntr++;
    if(Dcntr==1000){//after 1000 ms
     Dcntr=0;
     read_sonar();
     if (cup_cntr==0){
      cup_size=(Distance%100)+2;// the first reading for the cup size when the cup id empty, +2 the Distance between the cup and the sensor in system
      cup_cntr=1;
      }
    }
  INTCON = INTCON & 0xFB; //clear T0IF
}
 if(PIR1&0x04){//CCP1 interrupt

 PIR1=PIR1&0xFB;
 }
 if(PIR1&0x01){//TMR1 ovwerflow

   T1overflow++;

   PIR1=PIR1&0xFE;
 }

 if(INTCON&0x02){//External Interrupt


   INTCON=INTCON&0xFD;
   }


 }
 // lcd con using mkrioE libraries all in port D
sbit LCD_RS at RD4_bit;
sbit LCD_EN at RD5_bit;
sbit LCD_D4 at RD0_bit;
sbit LCD_D5 at RD1_bit;
sbit LCD_D6 at RD2_bit;
sbit LCD_D7 at RD3_bit;
sbit LCD_RS_Direction at TRISD4_bit;
sbit LCD_EN_Direction at TRISD5_bit;
sbit LCD_D4_Direction at TRISD0_bit;
sbit LCD_D5_Direction at TRISD1_bit;
sbit LCD_D6_Direction at TRISD2_bit;
sbit LCD_D7_Direction at TRISD3_bit;
void main() {
TRISE=0X00;
ADCON1=0x06;//PORTA Digital
TRISA=0x21;
TMR0=248;
PORTA=0X00;
PORTC=0X00;
CCP1CON=0x00;// Disable CCP. Capture on rising for the first time.  Capture on Rising: 0x05, Capture on Falling: 0x04
OPTION_REG = 0x87;//Fosc/4 with 256 prescaler => incremetn every 0.5us*256=128us ==> overflow 8count*128us=1ms to overflow
INTCON=0xF0;//enable TMR0 overflow, TMR1 overflow, External interrupts and peripheral interrupts;
init_sonar();
  while(1){
     PORTB=PORTB=PORTB|0X08; // stop the water pump it work in active low
     Lcd_Init(); // Initialize LCD
     Lcd_Cmd(_LCD_CLEAR); // Clear display
     Lcd_Cmd(_LCD_CURSOR_OFF); // Cursor off
     Lcd_Out(1,5,"Welcome !");//Write text'Welcome' in first row and fith col
      msDelay(50);
      if (PORTB&0x10){ // manual mode if portb pin 4 pressed
                msDelay(100);// delay to make sure that the user is finish pressing button ,and for button debounce
                manual();
      }
     if (PORTB&0x20){ // full mode   if portb pin 5 pressed
                full_cup();
      }
      if (PORTB&0x40)  //half mode   if portb pin 6 pressed
      {
               half_cup();
      }

      if (PORTB&0x80)  //QUARTER mode  if portb pin 7 pressed
      {
              quar_cup();
      }
  }

}
void read_sonar(void){

    T1overflow=0;
    TMR1H=0;
    TMR1L=0;

    PORTB=PORTB|0X04;//Trigger the ultrasonic sensor (RB2 connected to trigger)
    usDelay(10);//keep trigger for 10uS
    PORTB=PORTB&0XFB;//Remove trigger
    while(!(PORTB&0x02));
    T1CON=0x19;//TMR1 ON,  Fosc/4 (inc 1uS) with 1:2 prescaler (TMR1 overflow after 0xFFFF counts ==65536)==> 65.536ms
    while(PORTB&0x02);
    T1CON=0x18;//TMR1 OFF,  Fosc/4 (inc 1uS) with 1:1 prescaler (TMR1 overflow after 0xFFFF counts ==65536)==> 65.536ms
    T1counts=((TMR1H<<8)|TMR1L)+(T1overflow*65536);
       T1time=T1counts;//in microseconds
       Distance=((T1time*34)/(1000))/2; //in cm, shift left twice to divide by 2
       //range=high level time(usec)*velocity(340m/sec)/2 >> range=(time*0.034cm/usec)/2
       //time is in usec and distance is in cm so 340m/sec >> 0.034cm/usec
       //divide by 2 since the travelled distance is twice that of the range from the object leaving the sensor then returning when hitting an object)
}

void init_sonar(void){
   T1overflow=0;
    T1counts=0;
    T1time=0;
    Distance=0;
    TMR1H=0;
    TMR1L=0;
    TRISB=0x02; //RB2 for trigger, RB1 for echo
    PORTB=0x00;
    INTCON=INTCON|0xC0;//GIE and PIE
    PIE1=PIE1|0x01;// Enable TMR1 Overflow interrupt

    T1CON=0x18;//TMR1 OFF,  Fosc/4 (inc 1uS) with 1:2 prescaler (TMR1 overflow after 0xFFFF counts ==65536)==> 65.536ms
}

void usDelay(unsigned int usCnt){
    unsigned int us=0;

    for(us=0;us<usCnt;us++){
      asm NOP;//0.5 uS
      asm NOP;//0.5uS
    }

}
void msDelay(unsigned int msCnt){
    unsigned int ms=0;
    unsigned int cc=0;
    for(ms=0;ms<(msCnt);ms++){
    for(cc=0;cc<155;cc++);//1ms
    }
 }
void manual(){
         cup_cntr=0; // making cup_cntr=0 to take the first read of cup where find in intterupt function so i can find the water level
  while (1){
        water_level=(100-((Distance*100)/cup_size));
        if (water_level>100)// to make our read more accurate cause
        water_level=0;
        ByteToStr(water_level,x); // converting number to string from its ascci value using this fun so we can write it in lcd
        Lcd_Init(); // Initialize LCD
        Lcd_Cmd(_LCD_CLEAR); // Clear display
        Lcd_out(2,1,x);
        Lcd_out(1,1,"Water level:");
        Lcd_out(2,4,"%");
        Lcd_Cmd(_LCD_CURSOR_OFF);
        msDelay(100);// this delay soo the water level can be notice
      if (PORTB&0x10 ){//
         PORTB=PORTB|0X08;// stop water pump
         Lcd_Init(); // Initialize LCD
         Lcd_Cmd(_LCD_CLEAR); // Clear display
         Lcd_out(2,1,x);
         Lcd_out(1,1,"Water level: OK!");
         Lcd_out(2,4,"%");
         Lcd_Cmd(_LCD_CURSOR_OFF); 
         PORTB=PORTB&0xEF;// disable pin 4 port b
         msDelay(1500); // delay so the water level can be notice from the user
         break;// break out from while and fun too
         }
       else{
         PORTB=PORTB&0xF7; // water pump on in pin 3 port b
        }
      }

   }
void full_cup(void){
     cup_cntr=0;// making cup_cntr=0 to take the first read of cup where find in intterupt function so i can find the water level
  while (1){
        water_level=(100-((Distance*100)/cup_size));
        if (water_level>100) // to make our read more accurate
        water_level=0;
        ByteToStr(water_level,x);// converting number to string from its ascci value using this fun so we can write it in lcd
        Lcd_Init(); // Initialize LCD
        Lcd_Cmd(_LCD_CLEAR); // Clear display
        Lcd_out(2,1,x);
        Lcd_out(1,1,"Water level:");
        Lcd_out(2,4,"%");
        Lcd_Cmd(_LCD_CURSOR_OFF);
        msDelay(100);// this delay soo the water level can be notice
      if ((Distance)<5 && Distance>0 ){
         PORTB=PORTB|0X08;// stop water pump
         PORTB=PORTB&0XDF;// disable pin 5 port b
         Lcd_Init(); // Initialize LCD
         Lcd_Cmd(_LCD_CLEAR); // Clear display
         Lcd_Out(1,1,"Cup is full!");
         Lcd_Cmd(_LCD_CURSOR_OFF); // Cursor off
         msDelay(3000);  // delay so the water level can be notice from the user
         break;  // break out from while and fun too
         }
       else{
         PORTB=PORTB&0xF7; // water pump on in pin 3 port b
        }
      }

}
void half_cup(void)
{
      cup_cntr=0;// making cup_cntr=0 to take the first read of cup where find in intterupt function so i can find the water level and know where to stop
      PORTB=PORTB&0XBF;//disable pin 6 port b
   while (1){
        water_level=100-((Distance*100)/(cup_size+2));
        if (water_level>100)// to make our read more accurate
        water_level=0;
        ByteToStr(water_level,x);// converting number to string from its ascci value using this fun so we can write it in lcd
        Lcd_Init(); // Initialize LCD
        Lcd_Cmd(_LCD_CLEAR); // Clear display
        Lcd_out(2,1,x);
        Lcd_out(1,1,"Water level:");
        Lcd_out(2,4,"%");
        Lcd_Cmd(_LCD_CURSOR_OFF);
        msDelay(100);// this delay soo the water level can be notice
      if (Distance < cup_size-(cup_size/2) || (Distance)<3 )
      {
         PORTB=PORTB|0X08;// stop water pump
         Lcd_Init(); // Initialize LCD
         Lcd_Cmd(_LCD_CLEAR); // Clear display
         Lcd_Cmd(_LCD_CURSOR_OFF); // Cursor off
         Lcd_Out(1,1,"Cup is HALF!");
         msDelay(3000);// delay so the water level can be notice from the user
         break;
      }
       else{
         PORTB=PORTB&0xF7;// water pump on in pin 3 port b
        }
      }

 }
void quar_cup(void){
   cup_cntr=0;// making cup_cntr=0 to take the first read of cup where find in intterupt function so i can find the water level and know where to stop
   PORTB=PORTB&0X7F;//disable pin 7 port b
   while (1){
        water_level=100-((Distance*100)/cup_size);
        if (water_level>100)// to make our read more accurate
        water_level=0;
        ByteToStr(water_level,x); // converting number to string from its ascci value using this fun so we can write it in lcd
        Lcd_Init(); // Initialize LCD
        Lcd_Cmd(_LCD_CLEAR); // Clear display
        Lcd_out(2,1,x);
        Lcd_out(1,1,"Water level:");
        Lcd_out(2,4,"%");
        Lcd_Cmd(_LCD_CURSOR_OFF);
        msDelay(100);
      if ( (Distance < cup_size-(cup_size*0.25)) || (Distance)<3 )
      {
         PORTB=PORTB|0X08;// stop water pump
         Lcd_Init(); // Initialize LCD
         Lcd_Cmd(_LCD_CLEAR); // Clear display
         Lcd_Cmd(_LCD_CURSOR_OFF); // Cursor off
         Lcd_Out(1,1,"Quarter of ");
         Lcd_Out(2,1,"the cup!");
         msDelay(3000);// delay so the water level can be notice from the user
         break;
      }
       else{
         PORTB=PORTB&0xF7;// water pump on in pin 3 port b
        }
      }

}
