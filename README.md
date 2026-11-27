# accident-alert-system-project


#include <xc.h>
#include <stdio.h>
#include <stdint.h>

#pragma config FOSC = HS, WDTE = OFF, PWRTE = ON, BOREN = ON
#pragma config LVP = OFF, CPD = OFF, WRT = OFF, CP = OFF

#define _XTAL_FREQ 20000000UL

#define LCD_RS      PORTDbits.RD0
#define LCD_EN      PORTDbits.RD1
#define BUZZER      PORTCbits.RC0
#define CANCEL_BTN  PORTBbits.RB1

#define ACC_THRESHOLD   200
#define CANCEL_SECONDS  10
#define BUZZER_TEST_ONLY  0     /* 1 = only beep forever (hardware test) */

#define LAT_STR  "12.9716"
#define LON_STR  "77.5946"
#define PHONE    "+916379885613"

volatile uint8_t impact_flag = 0;
static uint16_t base_x, base_y, base_z;

void __interrupt() isr(void)
{
    if (INTCONbits.INTF) {
        INTCONbits.INTF = 0;
        impact_flag = 1;
    }
}

/* ---------------- LCD (4-bit) ---------------- */
static void lcd_nibble(uint8_t n)
{
    PORTD = (uint8_t)((PORTD & 0x0F) | (n << 4));
    LCD_EN = 1; __delay_us(2);
    LCD_EN = 0; __delay_us(50);
}
static void lcd_cmd(uint8_t c)
{
    LCD_RS = 0;
    lcd_nibble(c >> 4);
    lcd_nibble(c & 0x0F);
    if (c <= 0x02) __delay_ms(2);
}
static void lcd_char(char c)
{
    LCD_RS = 1;
    lcd_nibble((uint8_t)c >> 4);
    lcd_nibble((uint8_t)c & 0x0F);
}
static void lcd_goto(uint8_t row, uint8_t col)
{
    lcd_cmd((row ? 0xC0 : 0x80) + col);
}
static void lcd_print(uint8_t row, const char *s)
{
    uint8_t n = 0;
    lcd_goto(row, 0);
    while (*s && n < 16) { lcd_char(*s++); n++; }
    while (n++ < 16) lcd_char(' ');
}
static void lcd_init(void)
{
    TRISD = 0x00;
    PORTD = 0x00;
    __delay_ms(30);
    LCD_RS = 0;
    lcd_nibble(0x03); __delay_ms(5);
    lcd_nibble(0x03); __delay_us(150);
    lcd_nibble(0x03);
    lcd_nibble(0x02);
    lcd_cmd(0x28);
    lcd_cmd(0x0C);
    lcd_cmd(0x06);
    lcd_cmd(0x01);
}

/* ---------------- UART ---------------- */
static void uart_init(void)
{
    SPBRG = 129;                /* 9600 baud @ 20 MHz */
    TXSTAbits.BRGH = 1;
    TXSTAbits.SYNC = 0;
    TXSTAbits.TXEN = 1;
    RCSTAbits.SPEN = 1;
    RCSTAbits.CREN = 1;
}
static void uart_putc(char c)
{
    while (!TXSTAbits.TRMT);
    TXREG = c;
}
static void uart_puts(const char *s)
{
    while (*s) uart_putc(*s++);
}

/* ---------------- ADC ---------------- */
static uint16_t adc_read(uint8_t ch)
{
    ADCON0 = (uint8_t)(0x81 | (ch << 3));
    __delay_us(25);
    ADCON0bits.GO_nDONE = 1;
    while (ADCON0bits.GO_nDONE);
    return (uint16_t)(((uint16_t)ADRESH << 8) | ADRESL);
}
static uint16_t diff(uint16_t a, uint16_t b)
{
    return (a > b) ? (a - b) : (b - a);
}
static uint16_t accel_change(void)
{
    uint16_t dx = diff(adc_read(0), base_x);
    uint16_t dy = diff(adc_read(1), base_y);
    uint16_t dz = diff(adc_read(2), base_z);
    uint16_t m = dx;
    if (dy > m) m = dy;
    if (dz > m) m = dz;
    return m;
}

/* ---------------- Buzzer ---------------- */
static void beep(uint16_t on_ms)
{
    BUZZER = 1;
    while (on_ms--) __delay_ms(1);
    BUZZER = 0;
}

/* ---------------- GSM ---------------- */
static void gsm_send_sms(uint16_t peak)
{
    char msg[96];
    uart_puts("AT\r");                    __delay_ms(1000);
    uart_puts("AT+CMGF=1\r");             __delay_ms(1000);
    uart_puts("AT+CMGS=\"" PHONE "\"\r"); __delay_ms(1500);
    sprintf(msg, "ACCIDENT ALERT! Impact level %u. Location: "
                 "http://maps.google.com/?q=%s,%s", peak, LAT_STR, LON_STR);
    uart_puts(msg);
    uart_putc(0x1A);
    __delay_ms(3000);
}

/* ---------------- Alert sequence ---------------- */
static uint8_t handle_accident(uint16_t peak)
{
    char line[20];
    uint8_t s, i;

    for (s = CANCEL_SECONDS; s > 0; s--) {
        sprintf(line, "SMS in %u sec", s);
        lcd_print(0, "ACCIDENT DETECT!");
        lcd_print(1, line);

        BUZZER = 1;
        for (i = 0; i < 50; i++) {
            if (CANCEL_BTN == 0) { BUZZER = 0; return 0; }
            __delay_ms(10);
        }
        BUZZER = 0;
        for (i = 0; i < 50; i++) {
            if (CANCEL_BTN == 0) return 0;
            __delay_ms(10);
        }
    }
    lcd_print(0, "Sending alert...");
    lcd_print(1, "Please wait");
    gsm_send_sms(peak);
    return 1;
}

/* ---------------- Main ---------------- */
void main(void)
{
    uint16_t change, peak;

    TRISA = 0xFF;
    ADCON1 = 0x80;
    TRISB = 0xFF;
    OPTION_REGbits.nRBPU = 0;
    OPTION_REGbits.INTEDG = 0;
    TRISC = 0xC0;               /* RC6/RC7 inputs for USART, RC0..RC5 outputs */
    PORTC = 0x00;

#if BUZZER_TEST_ONLY
    while (1) { beep(1000); __delay_ms(1000); }
#endif

    lcd_init();
    uart_init();
    uart_puts("SYSTEM READY\r\n");

    beep(1000);                 /* startup buzzer test */
    __delay_ms(500);
    beep(1000);

    lcd_print(0, "Accident Alert");
    lcd_print(1, "Calibrating...");
    __delay_ms(1500);
    base_x = adc_read(0);
    base_y = adc_read(1);
    base_z = adc_read(2);

    INTCONbits.INTF = 0;
    INTCONbits.INTE = 1;
    INTCONbits.GIE  = 1;

    while (1)
    {
        lcd_print(0, "System Armed");
        lcd_print(1, "Monitoring...");

        change = accel_change();

        if (impact_flag || change > ACC_THRESHOLD) {
            peak = change;
            impact_flag = 0;

            if (handle_accident(peak)) {
                lcd_print(0, "SMS SENT");
                lcd_print(1, "Help on the way");
            } else {
                lcd_print(0, "Alert cancelled");
                lcd_print(1, "False alarm");
            }
            __delay_ms(3000);
            impact_flag = 0;
        }
        __delay_ms(50);
    }
}
