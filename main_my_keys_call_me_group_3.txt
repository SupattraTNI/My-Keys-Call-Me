#include "RC522.h"
#include "string.h"
#include <stdio.h>
#include "stm32l1xx.h"
#include "stm32l1xx_ll_system.h"
#include "stm32l1xx_ll_bus.h"
#include "stm32l1xx_ll_gpio.h"
#include "stm32l1xx_ll_pwr.h"
#include "stm32l1xx_ll_rcc.h"
#include "stm32l1xx_ll_utils.h"
#include "stm32l1xx_ll_spi.h"
#include "stm32l1xx_ll_tim.h"
#include "stm32l1xx_ll_i2c.h"

uint8_t k_status;
uint8_t data[18];

void SystemClock_Config(void);

// function buzzer
void GPIO_buzzer_Config(void);
void TIM_buzzer_Config(void);
void GPIO_peep_Config(void);
void GPIO_LED_Config(void);
void GPIO_swDoor_Config(void);
void Door_State(void); //use when put sw for lock and unlock the door



uint8_t Door_state1 = 0, cnt5 = 5, cnt10 = 10, cnt_test = 0, swd1, swd2, swd1_press, swd2_press;  // 0 is no one pass, 1 is someone pass
uint8_t inf1, inf2 ,state = 0;
int cnt_peep = 0, key_state = 0, swd1_state = 2, swd2_state = 2;

int main()
{
	SystemClock_Config(); //start clock 32Mhz
  	init_RC522(); //RFID begin working
	GPIO_buzzer_Config(); //PC0 for buzzer
	TIM_buzzer_Config(); //Set timer for buzzer
	GPIO_peep_Config(); //Set GPIO for infrared sensor which use for count people are in the room
	GPIO_swDoor_Config(); //Set GPIO for sw which use for lock and unlock the door

	LL_GPIO_ResetOutputPin(GPIOB, LL_GPIO_PIN_13); //PB13 output 0 for matrix door's sw
	LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_2); // Door is locked when start
	
  while(1) //working loop 
  {
		
    read_UID(PICC_ANTICOLL1, PICC_ARG_UID, data); //read card
    while(request_card(PICC_REQALL, data) == ERR)
		{	
			Door_State(); //put to lock and unlock the door

	// counting person
	
		// read infrared
			inf1 = !LL_GPIO_IsInputPinSet(GPIOC, LL_GPIO_PIN_10); //sensor found something is 1, sensor dosen't found something is 0
			inf2 = !LL_GPIO_IsInputPinSet(GPIOC, LL_GPIO_PIN_11); //sensor found something is 1, sensor dosen't found something is 0
			
		// person entered
			if(inf1) //sensor 1 found something
			{
				while(1) //wait for sensor 2 found something
				{
					inf2 = !LL_GPIO_IsInputPinSet(GPIOC, LL_GPIO_PIN_11); //raed sensor 2 again
					if(inf2) //sensor 1 found something
					{
						++cnt_peep; //people in the room +1 
						state = 1; //use for break while
					}
					if(state == 1)
						break; //break while
				}
				state = 0; //reset state
			}
			
		// perseon exited
			else if(inf2) //sensor 2 found something
			{
				while(1) //wait for sensor 1 found something
				{
					inf1 = !LL_GPIO_IsInputPinSet(GPIOC, LL_GPIO_PIN_10); //raed sensor 1 again
					if(inf1 && cnt_peep!=0) //sensor 1 found something and people are in the room not equal 0 
					{
						--cnt_peep; //people in the room -1 
						state = 1;
					}
					if(state == 1)
						break; //break while
				}
				state = 0; //reset state
			}
					
			 // person is 0
				if(cnt_peep == 0 && ((data[0] == 'y' || data[1] == 'h'))) //people are not in the room and key is in the room
				{
						
						LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_1); //yellow LED is set
					
						read_UID(PICC_ANTICOLL1, PICC_ARG_UID, data); //read card
						request_card(PICC_REQALL, data); //read card
						
						LL_TIM_EnableCounter(TIM10); //Enable timer 10
						read_UID(PICC_ANTICOLL1, PICC_ARG_UID, data); //read card
						while((data[0] == 'y' || data[1] == 'h')) //key is on the RFID
						{
								LL_GPIO_ResetOutputPin(GPIOC, LL_GPIO_PIN_2);	//green LED is set , door unlocked state
								read_UID(PICC_ANTICOLL1, PICC_ARG_UID, data);   //read card
								request_card(PICC_REQALL, data);  //read card
								//start buzzer step 2
								if((LL_TIM_IsActiveFlag_UPDATE(TIM10) == SET)) //if flag timer 10 is set
								{
									LL_GPIO_TogglePin(GPIOC, LL_GPIO_PIN_0); //toggle buzzer
									LL_TIM_ClearFlag_UPDATE(TIM10); //clear flag timer 10
									LL_TIM_SetCounter(TIM10, 0); //set timer 10 is 0
									memset(data, '\0' , 18); //clear data which is read from card
								}
						}
					// key is picked
					memset(data, '\0' , 18); //clear data which is read from card
						
				}

				// person 0 -> 1
				if(cnt_peep >= 1) //1 person or more are in the room
				{
						LL_GPIO_ResetOutputPin(GPIOC, LL_GPIO_PIN_1); //yellow LED is reset
						LL_TIM_EnableCounter(TIM2); //Enable timer 10
						
						// waiting for 5 sec
						while( cnt5 != 0 && !(data[0] == 'y' || data[1] == 'h')) //wiil break when time over 5 sec or key is on the RFID 
						{
			
							read_UID(PICC_ANTICOLL1, PICC_ARG_UID, data); //read card
							request_card(PICC_REQALL, data); //read card
							Door_State(); //put to lock and unlock the door
							if((LL_TIM_IsActiveFlag_UPDATE(TIM2) == SET)) //if flag timer 2 is set
							{
								cnt5--; //count down which each 1 sec from 5 to 0
								LL_TIM_ClearFlag_UPDATE(TIM2); //clear flag for TIM2
								LL_TIM_SetCounter(TIM2, 0); //set timer 2 is 0
							}
						}
						
						// start buzzer step 1
						while(!(data[0] == 'y' || data[1] == 'h')) //will break when key is on the RFID 
						{
								read_UID(PICC_ANTICOLL1, PICC_ARG_UID, data); //read card
								request_card(PICC_REQALL, data); //read card
								Door_State(); //put to lock and unlock the door
								if(LL_TIM_IsActiveFlag_UPDATE(TIM2) == SET) //if flag timer 2 is set
								{
									LL_GPIO_TogglePin(GPIOC, LL_GPIO_PIN_0); //toggle buzzer
									LL_TIM_ClearFlag_UPDATE(TIM2); //clear flag for TIM2
									LL_TIM_SetCounter(TIM2, 0); //set timer 2 is 0
								}
								//key isn't on the RFID and leave the room
								inf2 = !LL_GPIO_IsInputPinSet(GPIOC, LL_GPIO_PIN_11); //raed sensor 2 again
								if(inf2==1) ////sensor 2 found something because leave the room
								{
									LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_0); //buzzer quite
									cnt_peep = 0; //people in the room is 0
									break;
								}
						}

						// key is put
						if(data[0] == 'y' || data[1] == 'h') //key is on the RFID
						{
							LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_1); //yellow LED is Set
							LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_0); //buzzer quite
						}
						
						// clear data
						memset(data, '\0' , 18); //clear data which is read from card	
				}
				
				
				if(cnt_peep == 0 && !(data[0] == 'y' || data[1] == 'h')) //people in the room = 0 and key isn't on the RFID (people is outside the room)
				{
					Door_State(); //put to lock and unlock the door
				}

				
		} // while(request_card(PICC_REQALL, data) == ERR)
	} // while(1)
}

void GPIO_peep_Config(void)
{
		LL_GPIO_InitTypeDef door1_GPIO;
		LL_AHB1_GRP1_EnableClock(LL_AHB1_GRP1_PERIPH_GPIOC);
    
  	door1_GPIO.OutputType = LL_GPIO_OUTPUT_PUSHPULL;
  	door1_GPIO.Pull =LL_GPIO_PULL_NO;
  	door1_GPIO.Speed = LL_GPIO_SPEED_FREQ_HIGH;
  	door1_GPIO.Mode = LL_GPIO_MODE_INPUT;
	
  	door1_GPIO.Pin = LL_GPIO_PIN_10;
  	LL_GPIO_Init(GPIOC, &door1_GPIO); 		// PC10 is infrared outside room

		door1_GPIO.Pin = LL_GPIO_PIN_11;
  	LL_GPIO_Init(GPIOC, &door1_GPIO);			// PC11 is infrared inside room
	
}

void TIM_buzzer_Config(void)
{
	LL_TIM_InitTypeDef tim_buzzer;
	
	LL_APB1_GRP1_EnableClock(LL_APB1_GRP1_PERIPH_TIM2);	
	LL_APB1_GRP1_EnableClock(LL_APB1_GRP1_PERIPH_TIM6);
	LL_APB2_GRP1_EnableClock(LL_APB2_GRP1_PERIPH_TIM9);
	LL_APB2_GRP1_EnableClock(LL_APB2_GRP1_PERIPH_TIM10);
	tim_buzzer.ClockDivision = LL_TIM_CLOCKDIVISION_DIV1;
	tim_buzzer.CounterMode = LL_TIM_COUNTERMODE_UP;
	
	tim_buzzer.Autoreload = 4000-1;		
	tim_buzzer.Prescaler = 8000-1;		
	LL_TIM_Init(TIM2, &tim_buzzer);		// 1 s
	
	tim_buzzer.Autoreload = 4000-1;		
	tim_buzzer.Prescaler = 4000-1;
	LL_TIM_Init(TIM9, &tim_buzzer);  // 0.5 s
	
	tim_buzzer.Autoreload = 4000-1;	
	tim_buzzer.Prescaler = 2000-1;
	LL_TIM_Init(TIM10, &tim_buzzer);  //0.25 s
	
	tim_buzzer.Autoreload = 4000-1;		
	tim_buzzer.Prescaler = 40000-1;
	LL_TIM_Init(TIM6, &tim_buzzer);  // 5 s
}

void GPIO_buzzer_Config(void)
{
		LL_GPIO_InitTypeDef buzzer_GPIO;
		LL_AHB1_GRP1_EnableClock(LL_AHB1_GRP1_PERIPH_GPIOC);
        
  	buzzer_GPIO.OutputType = LL_GPIO_OUTPUT_PUSHPULL;
  	buzzer_GPIO.Pull = LL_GPIO_PULL_NO;
  	buzzer_GPIO.Speed = LL_GPIO_SPEED_FREQ_HIGH;
  	buzzer_GPIO.Mode = LL_GPIO_MODE_OUTPUT;
  	buzzer_GPIO.Pin = LL_GPIO_PIN_0;
  	LL_GPIO_Init(GPIOC, &buzzer_GPIO);
		LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_0);
}

void GPIO_LED_Config(void)
{
	LL_GPIO_InitTypeDef LED_GPIO;
	LL_AHB1_GRP1_EnableClock(LL_AHB1_GRP1_PERIPH_GPIOC);
	
	LED_GPIO.OutputType = LL_GPIO_OUTPUT_PUSHPULL;
  LED_GPIO.Pull =LL_GPIO_PULL_NO;
  LED_GPIO.Speed = LL_GPIO_SPEED_FREQ_HIGH;
  LED_GPIO.Mode = LL_GPIO_MODE_OUTPUT;
	
  LED_GPIO.Pin = LL_GPIO_PIN_1;
  LL_GPIO_Init(GPIOC, &LED_GPIO); //y
	
	LED_GPIO.Pin = LL_GPIO_PIN_2;
  LL_GPIO_Init(GPIOC, &LED_GPIO); //g & r, g is active-low, r is active-high
}

void GPIO_swDoor_Config(void)
{
	LL_GPIO_InitTypeDef SWdoor_GPIO;
	LL_AHB1_GRP1_EnableClock(LL_AHB1_GRP1_PERIPH_GPIOB);
	
	SWdoor_GPIO.OutputType = LL_GPIO_OUTPUT_PUSHPULL;
	SWdoor_GPIO.Pull = LL_GPIO_PULL_NO;
	SWdoor_GPIO.Speed = LL_GPIO_SPEED_FREQ_HIGH;
	SWdoor_GPIO.Mode = LL_GPIO_MODE_OUTPUT;
	
	SWdoor_GPIO.Pin = LL_GPIO_PIN_13;
	LL_GPIO_Init(GPIOB, &SWdoor_GPIO);
	
	SWdoor_GPIO.Mode = LL_GPIO_MODE_INPUT;
	SWdoor_GPIO.Pin = LL_GPIO_PIN_14;
	LL_GPIO_Init(GPIOB, &SWdoor_GPIO);
	
	SWdoor_GPIO.Mode = LL_GPIO_MODE_INPUT;
	SWdoor_GPIO.Pin = LL_GPIO_PIN_15;
	LL_GPIO_Init(GPIOB, &SWdoor_GPIO);

}

void Door_State(void)
{
				swd1_press = swd1;
				swd1 =!LL_GPIO_IsInputPinSet(GPIOB, LL_GPIO_PIN_14); //swd inside
				//read sw door
				if(swd1_press==1 && swd1 ==0)
				{
					if(swd1_state%2 == 0) //unlocked
					{
						LL_GPIO_ResetOutputPin(GPIOC, LL_GPIO_PIN_2); //g tid, r dub
						++swd1_state;
					}
					else if(swd1_state%2 == 1) //locked
					{
						LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_2); //g dub, r tid
						--swd1_state;
					}
				}
				swd2_press = swd2;
				swd2 =!LL_GPIO_IsInputPinSet(GPIOB, LL_GPIO_PIN_15); //swd outside
				if(swd2_press == 1 && swd2 == 0)
				{
					if(swd2_state%2 == 0) //lock
					{
						LL_GPIO_SetOutputPin(GPIOC, LL_GPIO_PIN_2); //g dub, r tid
						++swd2_state;
					}
					else if(swd2_state%2 == 1) //unlock
					{
						LL_GPIO_ResetOutputPin(GPIOC, LL_GPIO_PIN_2); //g tid, r dub
						--swd2_state;
					}
				}
}

void SystemClock_Config(void)
{
  /* Enable ACC64 access and set FLASH latency */ 
  LL_FLASH_Enable64bitAccess();; 
  LL_FLASH_SetLatency(LL_FLASH_LATENCY_1);

  /* Set Voltage scale1 as MCU will run at 32MHz */
  LL_APB1_GRP1_EnableClock(LL_APB1_GRP1_PERIPH_PWR);
  LL_PWR_SetRegulVoltageScaling(LL_PWR_REGU_VOLTAGE_SCALE1);
  
  /* Poll VOSF bit of in PWR_CSR. Wait until it is reset to 0 */
  while (LL_PWR_IsActiveFlag_VOSF() != 0)
  {
  };
  
  /* Enable HSI if not already activated*/
  if (LL_RCC_HSI_IsReady() == 0)
  {
    /* HSI configuration and activation */
    LL_RCC_HSI_Enable();
    while(LL_RCC_HSI_IsReady() != 1)
    {
    };
  }
	
  /* Main PLL configuration and activation */
  LL_RCC_PLL_ConfigDomain_SYS(LL_RCC_PLLSOURCE_HSI, LL_RCC_PLL_MUL_6, LL_RCC_PLL_DIV_3);

  LL_RCC_PLL_Enable();
  while(LL_RCC_PLL_IsReady() != 1)
  {
  };
  
  /* Sysclk activation on the main PLL */
  LL_RCC_SetAHBPrescaler(LL_RCC_SYSCLK_DIV_1);
  LL_RCC_SetSysClkSource(LL_RCC_SYS_CLKSOURCE_PLL);
  while(LL_RCC_GetSysClkSource() != LL_RCC_SYS_CLKSOURCE_STATUS_PLL)
  {
  };
  
  /* Set APB1 & APB2 prescaler*/
  LL_RCC_SetAPB1Prescaler(LL_RCC_APB1_DIV_1);
  LL_RCC_SetAPB2Prescaler(LL_RCC_APB2_DIV_1);

  /* Set systick to 1ms in using frequency set to 32MHz                             */
  /* This frequency can be calculated through LL RCC macro                          */
  /* ex: __LL_RCC_CALC_PLLCLK_FREQ (HSI_VALUE, LL_RCC_PLL_MUL_6, LL_RCC_PLL_DIV_3); */
  LL_Init1msTick(32000000);
  
  /* Update CMSIS variable (which can be updated also through SystemCoreClockUpdate function) */
  LL_SetSystemCoreClock(32000000);
}
