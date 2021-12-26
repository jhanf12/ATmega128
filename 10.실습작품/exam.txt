/*
 * exam.c
 *
 * Created: 2021-12-08 오후 11:51:28
 * Author : sunghun0407
 */ 

#include <avr/io.h>
#include <avr/interrupt.h>

char dig_table[4] = {0x10, 0x20, 0x40, 0x80};
//                DIG1, DIG2, DIG3, DIG4
char fnd_table[10] = {0x3F, 0x06, 0x5B, 0x4F, 0x66, 0x6D, 0x7D, 0x27, 0x7F, 0x6F};
					//  0,     1,   2,     3,    4,   5,   6,    7,    8,     9

#define T1_START 61  // 1/((1/16000000)*1024*256) 1024 분주비 주기 대략61정도인듯

int sec;
int color = 2; // 0:green, 1:orange, 2:red
int cnt = 0; // 시간으로 받을거고
int tmr_cnt = 0;  // 주파수 영역 계산값

ISR(TIMER0_COMP_vect)  // 타이머 카운터 compare  1ms라고?
{
	tmr_cnt = tmr_cnt+ 1; //속도
	if(tmr_cnt > T1_START)
	{
		tmr_cnt = 0;
		cnt = cnt - 1;
	}
}

/*ISR(TIMER0_OVF_vect)  // 타이머 오버플로우
{
	TCNT0 = T1_START;
	cnt = cnt - 1;
	
	if(++tmr_cnt > T1_START)
	{
		tmr_cnt = 0;
	}
}*/

ISR(INT4_vect)  // 수정완료 12/08 무한반복으로 안걸리고 처음상태로 돌아간다.
{
	cnt = 0;  
	color = 2;
}


void Delay_us(char time_us) {
	char i;

	for(i=0;i<time_us;i++){
		asm volatile(" PUSH R0 " );
		asm volatile(" POP  R0 " );
		asm volatile(" PUSH R0 " );
		asm volatile(" POP  R0 " );
		asm volatile(" PUSH R0 " );
		asm volatile(" POP  R0 " );
	}
}
void Delay_ms(unsigned int time_ms) {
	unsigned int i;

	for(i=0; i<time_ms; i++){
		Delay_us(250);
		Delay_us(250);
		Delay_us(250);
		Delay_us(250);
	}
}

void Display_FND(int num) {   // 프린트처럼 while문 안에 작성했더니 탈출이 안됨 무한반복, Display를 만들어놓고 cnt 시간을 받아와서 출력
	PORTD = dig_table[2];
	PORTC= ~fnd_table[num/10];
	Delay_ms(1);

	PORTD = dig_table[3];
	PORTC= ~fnd_table[num%10];
	Delay_ms(1);
}

void lcd_write(char c) {
	PORTA = c;
	PORTG |= 0x04;
	Delay_us(1);
	PORTG &= 0xFB;
	Delay_us(250);
}
void cursor_off(void) {
	PORTG &= 0x7F;
	Delay_ms(200);
	lcd_write(0x0C);
	Delay_ms(100);
}
void cursor_on(void) {
	PORTG &= 0xFE;
	Delay_ms(200);
	lcd_write(0x0F);
	Delay_ms(100);
}

void lcd_clear(void) {
	PORTG &= 0xFE;
	Delay_us(1);
	lcd_write(0x01);
	Delay_ms(4);
}

void lcd_init(void) {
	PORTG &= 0xFE;
	Delay_ms(200);
	lcd_write(0x38);
	lcd_write(0x0F);
	lcd_write(0X01);
	Delay_ms(100);
}

void lcd_gotoxy(unsigned char x, unsigned char y) {
	PORTG &= 0xFE;
	Delay_us(1);

	if(y==0) lcd_write(0x80 + x);
	else lcd_write(0xC0 + x);
}

void lcd_puts(char *s) {
	PORTG |= 0x01;
	Delay_us(1);

	while(*s)
	lcd_write(*s++);
}

void lcd_putch(char c) {
	PORTG |=0x01;
	Delay_us(1);
	PORTA = c;

	PORTG |=0x04;
	Delay_us(1);
	PORTG &=0xFB;
	Delay_us(250);
}

void Init_IOport(void)
{
	DDRA = 0xFF; // LCD
	PORTA = 0xFF;

	DDRB = 0xFF; // LED
	PORTB = 0xFF;

	DDRC=0xFF; // FND
	PORTC=0xFF;

	DDRD=0xFF; // FND DIGIT
	PORTD=0x0F;

	DDRG = 0xFF; // LCD CONTROL
	PORTG = 0x00;

	lcd_init();
}

void Init_Interrupt(void)
{
	TCCR0 = 0x07; // clk / 1024 분주비 설정
	TIMSK = 0x02; // timer0 overflow interrupt 설정
	
	EICRB = 0x02; // INT4 하강엣지
	EIMSK = 0x10; // INT4 사용가능하게

	sei();  // 전체인터럽트 켜기
}

int main(void)
{

	Init_IOport();
	Init_Interrupt();

	while (1)
	{
		if(cnt < 1)
		{
			color = (color+1)%3;  // 나머지 계산해서 0 1 2 만들어주기

			if(color == 0) // green
			{
				lcd_clear();
				lcd_gotoxy(0,0);
				lcd_puts("GREEN LED");
				lcd_gotoxy(0,1);
				lcd_puts("GO");

				PORTB = ~(0xE0); // 0x80 0x40 0x20 더한거
				cnt = 12;
			}
			else if(color == 1) // orange
			{
				lcd_clear();
				lcd_gotoxy(0,0);
				lcd_puts("ORANGE LED");
				lcd_gotoxy(0,1);
				lcd_puts("READY TO STOP");

				PORTB = ~(0x1C);  //0x10 0x08 0x04  더함 
				cnt = 3;
			}
			else if(color == 2) // red
			{
				lcd_clear();
				lcd_gotoxy(0,0);
				lcd_puts("RED LED");
				lcd_gotoxy(0,1);
				lcd_puts("STOP");

				PORTB = ~(0x03); //0x02 0x01  더함
				cnt = 10; 
			}
		} // if문 종료
		Display_FND(cnt);
	}
}
