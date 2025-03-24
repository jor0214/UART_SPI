# **Electric Tin-Can Telephone**
## **Introduction**

This project demonstrates communication between two Nucleo STM using UART. The objective is to send and recieve messages between the boards and display them on the LCD screen. Additionally, LEDs are used to provide feedback of data transmission and reception.

### **Key Features:**
- **USART Communication** : Utilizes USART1 and USART2 for serial communication.
- **Message Display**: LCD display used to present recieved and sent messages.
- **Feedback** : LEDs provide feedback upon message sent and recieved.
- **Data Handling** : Intterupts and circular buffer used to manage data reception.

## **System Architecture**

The system consists of two Nucleo boards:
1. **Sender Board** : Sends messages to the reciever.
2. **Reciever Board**: Recieves messages and displays them both on the LCD.

## **Data Flow & Message Handling**
### **Sender Board**
- Reads user input from the **serial monitor (USART2)**.
- Sends the message to the receiver via **USART1**.
- Displays the sent message on the **LCD screen**.

### **Receiver Board**
- Listens for incoming data via **USART1**.
- Displays the received message on the **LCD screen**.
- Activates **LED and buzzer notifications** when a message is received.


## **Hardware & Pin Configuration**
Each **Nucleo board** has two USART interfaces:

1. **USART2** – Used for the serial monitor.  
2. **USART1** – Used for direct communication between the two Nucleo boards.

| **Board**           | **Serial Monitor (USART2)** | **Inter-Board Communication (USART1)** |
|---------------------|-----------------------------|----------------------------------------|
| **Sender Nucleo**   | TX: PA2, RX: PA15           | TX: PA9, RX: PA10                      |
| **Receiver Nucleo** | TX: PA2, RX: PA15           | TX: PA9, RX: PA10   

Note: PA15 is an built-in pin and is not included in the Nucleo pin-mapped diagram but can be found in the datasheet.

## **Code Structure**
### **Main Components**
| **File**                   | **Description** |
|----------------------------|----------------|
| `main.c`                   | Core program handling UART communication, message parsing, and LCD display. |
| `display.c / display.h`    | Functions for rendering text on the LCD screen. |
| `circular_buffer.c / circular_buffer.h` | Implements a circular buffer for UART data handling. |
| `spi.c / spi.h`            | Manages SPI communication for the LCD screen. |
| `eeng1030_lib.c / eeng1030_lib.h` | Library functions used across the project. |
| `font5x7.h`               | Defines fonts used for text rendering on the LCD. |

### **Important Functions**
| **Function**              | **Purpose** |
|---------------------------|------------|
| `setup()`                 | Initializes GPIO, UART, and peripherals. |
| `initSerial(uint32_t baudrate)` | Configures USART1 and USART2. |
| `readInputMessage()`      | Reads user input from USART2. |
| `sendMessage(char *message)` | Sends a formatted message over USART1. |
| `USART1_IRQHandler()`     | Interrupt handler for receiving messages. |
| `printMessage(int i, const char *message)` | Displays messages on the LCD. |

### USART Configuration
#### Enabling Clocks
Before configuring USART, the appropriate **clocks** must be enabled for the GPIO ports and USART peripherals.

```c
RCC->AHB2ENR |= (1 << 0); // Enable GPIOA clock
RCC->APB1ENR1 |= (1 << 17); // Enable USART2 clock
RCC->APB2ENR |= (1 << 14); // Enable USART1 clock
```
These bits can be located in the Nucleo STM32l432 reference manual:
**USART1**
![image](https://github.com/user-attachments/assets/73726d1c-21c2-4d66-8919-628dbc14e7f4)
*Figure 1: Reference manual showing USART1 RCC*

![image](https://github.com/user-attachments/assets/7efb780a-6f6e-4e9f-a9a6-0335249f2aff)
*Figure 2: Reference manual showing Bit 14 USART1*

**USART2**
![image](https://github.com/user-attachments/assets/ed30dfe9-adea-4d77-bdd7-bbf5e23c03c8)
*Figure 3: Reference manual showing USART2 RCC*

![image](https://github.com/user-attachments/assets/a960f7ff-7d25-4fe1-ba49-984219ad92d8)
*Figure 4: Reference manual showing Bit 17*

### GPIO Configuration for USART Pins
The TX and RX pins must be configured in **alternate function mode** to work with USART.

```c
pinMode(GPIOA,2,2); // Set PA2 to alternate function mode (USART2 TX)
selectAlternateFunction(GPIOA,2,7); // AF7 for USART2 TX

pinMode(GPIOA,15,2); // Set PA15 to alternate function mode (USART2 RX)
selectAlternateFunction(GPIOA,15,3); // AF3 for USART2 RX
```
**STM32l432 Datasheet: Alternative Functions**
![image](https://github.com/user-attachments/assets/cf132ca5-d7ed-490f-b5aa-5429f9672191)
*Figure 5: Reference manual showing alternative function table for USART*

#### Enabling USART
After configuration, both TX and RX must be enabled in the **control registers**.

```c
USART1->CR1 =  (1 << 3) | (1 << 2) | (1 << 0);  // Enable TX, RX, and USART
USART2->CR1 =  (1 << 3) | (1 << 2) | (1 << 0);
```
**Control Register 1 (USART1, USART2)**
![image](https://github.com/user-attachments/assets/8f106633-1200-4c1e-85f5-9b7609b31d6c)
*Figure 6: Control register 1 showing TE and RE*

#### Transmitting Data
To send a single character via USART2:

```c
void eputc(char c)
{
    while (!(USART2->ISR & (1 << 6))); // Wait for TX ready
    USART2->TDR = c;
} 
```
To send a string via USART1:

```c
void sendMessage(char *message)
{
    while (*message)
    {
        while (!(USART1->ISR & (1 << 6))); // Wait for TX ready
        USART1->TDR = *message++;
    }
}
```

#### Receiving Data
USART1 receives data via an interrupt handler:

```c
void USART1_IRQHandler(void)
{
    if (USART1->ISR & (1 << 5)) // Check RXNE flag
    {
        char c = USART1->RDR; // Read received character
        put_circ_buf(&rx_buf, c); // Store in circular buffer
    }
}
```
![image](https://github.com/user-attachments/assets/b7b2e395-f213-43a1-b5ae-f7f3a56981b5)
*Figure 7: Bit assignments for RXNE flag and TC*
## Circular Buffer
A **circular buffer** efficiently stores incoming data, especially in asynchronous situations like USART communication. It overwrites the oldest data when full, ensuring continuous data capture without manual reset.

In this project, the buffer stores data from **USART1**, keeping the order intact even if data arrives out of sequence.

### Why Use a Circular Buffer?
- **Efficient Memory**: Fixed-size, overwrites old data when full.
- **Real-Time**: Data is stored and processed asynchronously without blocking.
- **Asynchronous Handling**: Allows data reception in interrupts without halting the program.

### Circular Buffer Structure

```c
typedef struct {
    uint8_t buffer[CIRC_BUF_SIZE]; // Data storage
    uint32_t head;                 // Next write position
    uint32_t tail;                 // Next read position
    uint8_t full;                  // Buffer full flag
} circular_buffer;
```
## SPI Configuration for LCD Screen

### Overview
The **SPI (Serial Peripheral Interface)** is used to communicate with the LCD screen. The Nucleo board acts as the **SPI master**, sending data to the **LCD display (SPI slave)**. The relevant SPI signals and their assigned **PAx pins** are:

| SPI Signal | Function | STM32 Pin |
|------------|----------|------------|
| **SCK** (Clock) | Serial clock for SPI | **PA1(SCL)** |
| **MOSI** (Master Out, Slave In) | Data from Nucleo to LCD | **PA7 (SDA)** |
| **CS** (Chip Select) | Selects LCD screen | **PA4 (CS)** |
| **RES** (Reset) | Resets LCD module | **PA3 (RES)** |
| **DC** (Data/Command) | Switches between data & command | **PA8 (DC)** |

![image](https://github.com/user-attachments/assets/727740fb-2789-4435-b52b-64dca5b93227)
*Figure 8: Pin mapping for SPI in alternative function table*

## Circuit Diagram  

The Kicad Schematic below shows the wiring connections between the microcontroller, SPI LCD, and USART communication interfaces.  

![image](https://github.com/user-attachments/assets/62267e01-253a-49ae-aae5-63909b5f168c)
*Figure 9: Kicad schematic of project layout*

### Pin Connections  

| **MCU Pin** | **Connected To** | **Function** |
|------------|----------------|------------|
| PA2  | Serial Monitor (TX) | UART TX |
| PA15 | Serial Monitor (RX) | UART RX |
| PA9  | Board 2 (TX) | UART1 TX |
| PA10 | Board 2 (RX) | UART1 RX |
| PA1  | LCD SCL | SPI Clock |
| PA7  | LCD SDA | SPI Data |
| PA3  | LCD RES | Reset |
| PA4  | LCD CS  | Chip Select |
| PA8  | LCD DC  | Data/Command |

### Wiring Explanation  

- **USART Communication:**  
  - PA2 (TX) transmits data to the serial monitor (PuTTY).  
  - PA15 (RX) receives data from the serial monitor.  
  - PA9 and PA10 handle UART communication between the two boards.  

- **SPI LCD Interface:**  
  - The LCD screen communicates using SPI, with SCL (clock) and SDA (data) lines.  
  - The CS (chip select) pin enables the display, while RES (reset) and DC (data/command) manage control signals. 

## Error Handling

Initally, when messages were being sent, only some of the message was being recieved on the second board. The displayed recieved message was scrambled on the LCD display. 
To identify  the issue, a logic analyzer was used. This showed that the TX pin was sending data but the RX pin was missing some bytes. 
Following investigation, the input buffer size was deemed too small and so data was being lost. 

After increasing the size of the buffer, the logic analyzer then showed the following:
![image](https://github.com/user-attachments/assets/486a645a-621d-4aaf-a828-5479cce4fc89)
*Figure 10: Logic analyzer showing the start of the message exchange with a hello.*
![image](https://github.com/user-attachments/assets/985d579c-ad94-4195-8920-47ad5b535fe6)
*Figure 11: Logic anaylzer showing USART2 to confirm a message being sent.*
![image](https://github.com/user-attachments/assets/aef3329d-8117-438d-9f31-c67798111e59)
*Figure 12: Logic analyzer showing message being recieved.*
![image](https://github.com/user-attachments/assets/be9698be-23b6-44b9-b440-9adc1dec6ea4)
*Figure 13: Logic analyzer showing SPI displaying messages.*

## Conclusion  

This project successfully implements serial communication between two microcontrollers using USART, while also integrating an SPI-based LCD display for output visualization. The system efficiently transmits and receives messages between boards and displays relevant information on the screen.  

By utilizing **UART for inter-board communication** and **SPI for LCD interfacing**, the project demonstrates a practical approach to embedded system communication. The use of PuTTY as a serial monitor allowed easier use of the system.
Future improvements could include implementing additional error handling, message wrapping for larger strings and wireless communication methods.  

**Timers were not neccesary** as both USART and SPI handled their own timing via baud rate and clock signals. The flow of data was managed using interrupts and status flags so no real need for additional timing control was neccesary.

## References
1. **STM32 Reference Manual**  
    c:\Users\jenni\Downloads\stm32l432_reference_manual.pdf

2. **STM32 Data Sheet**  
   c:\Users\jenni\Downloads\stm32l432kc_datasheet.pdf

3. **ST7735S Data Sheet**
    (../../../../../../../Downloads/ST7735S.pdf)

4. **Lecturer's GitHub Repository**  
    https://github.com/f3dtud/EENG1030
