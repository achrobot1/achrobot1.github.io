---
title: "STM32 Guitar Tuner"
date: 2020-02-13
tags: [embedded, audio, STM32]
---

# Guitar Tuner

[Github](https://github.com/achrobot1/guitar-tuner-STM32)

I built a guitar tuner using an electret microphone, an ARM Cortex-M3 processor, and a 128x32 OLED display. For those unfamiliar with this style of guitar tuner, it operates as follows: the user plays a guitar string that they are trying to tune, and the OLED displays tuning instructions (tune up, tune down, or in tune).

I have been wanting to do this project for a long time now and was glad to finally get around to it.

### Parts used
* NUCLEO-STM32F103RB Development board
* SSD1306 OLED Display (128 x 32)
* SparkFun Sound Detector




# Approach/Analysis
Going into this project I had minimal experience programming the STM32F103 board and working with discrete signals. I was not sure how I would detect fundamental frequency from a time domain recording of a guitar string.

I started this project by taking samples from the ADC on the STM32 and sending them over UART to my laptop so I could analyze the signals using Python. Since the highest frequency guitar string is the e string at 330Hz, I had to sample at a frequency of at least 660Hz. I gave myself some cushion and chose to use a 1KHz sampling rate. This also made the timer interrupt setup simple.

The below Python notebook describes how I came up with a way to detect fundamental frequency of a recorded guitar string.

# Implementation
Once I had the unit vectors and the algorithm working in Python, I then had to implement that on the STM32.

![Block Diagram](/images/block_diagram.png)

In summary the STM32 uses several peripherals. A timer is used to generate interrupts at 1KHz. The interrupt service routine reads converted data from the ADC peripheral and fills a buffer that can hold 256 float values. SPI is used to drive the OLED display.

In `main()`, we setup all peripherals, initialize the OLED display and then enter a `while` loop. The `while` loop constantly checks if the buffer is filled with values. If the buffer is full, we disable the timer interrupt, perform an FFT, predict the guitar string being played, estimate the fundamental frequency, fill the OLED display buffer with proper tuning instructions, write to the OLED and re-enable the timer interrupt.

For full code see [github](https://github.com/achrobot1/guitar-tuner-STM32/tree/master/guitar-tuner-STM32)

```
while(1)
    {
	Delay(10);

	if(buffer_count == ADC_BUFF_SIZE) // Check if adc_buffer is filled with values
	{
	    TIM_ITConfig(TIM2, TIM_IT_CC1, DISABLE); // Disable TIM2 interrupt

	    fft(fft_samples, ADC_BUFF_SIZE, temp);
	    get_magnitudes(mag, fft_samples, ADC_BUFF_SIZE);
	    filter_frequencies(mag, freq, 0.0, 4.0, ADC_BUFF_SIZE);
	    filter_frequencies(mag, freq, 60.0, 5.0, ADC_BUFF_SIZE);

	    // Dot product of unit_vecs with signal fft
	    int i;
	    float dp[6] = {0};
	    for(i=0; i<UNIT_VEC_LENGTH; i++)
	    {
		float f = eigen_frequencies[i];
		int index = freq_to_index(f, 1000, ADC_BUFF_SIZE);
		dp[E] += E_fft_unit_vec[i] * mag[index];
		dp[A] += A_fft_unit_vec[i] * mag[index];
		dp[D] += D_fft_unit_vec[i] * mag[index];
		dp[G] += G_fft_unit_vec[i] * mag[index];
		dp[B] += B_fft_unit_vec[i] * mag[index];
		dp[e] += e_fft_unit_vec[i] * mag[index];
	    }
	    
	    // find max string
	    float max_val = 0.0;
	    int max_index;
	    for(i=0; i<6; i++)
	    {
		if(dp[i] > max_val) 
		{
		    max_val = dp[i];
		    max_index = i;
		}
	    }

	    if(UART_DEBUG)
	    {
		char buffer[20];
		/*
		// Send string prediction over uart
		switch(max_index)
		{
		    case E: sprintf(buffer, "String: %s \n\r", "E");
			break;
		    case A: sprintf(buffer, "String: %s \n\r", "A");
			break;
		    case D: sprintf(buffer, "String: %s \n\r", "D");
			break;
		    case G: sprintf(buffer, "String: %s \n\r", "G");
			break;
		    case B: sprintf(buffer, "String: %s \n\r", "B");
			break;
		    case e: sprintf(buffer, "String: %s \n\r", "e");
			break;
		}
		uart_puts(buffer, USART1);
		*/

		float estimated_freq = estimate_frequency(mag, freq, max_index);
		if(max_index==E) estimated_freq /= 2;
		if(max_index==A) estimated_freq /= 3;
		sprintf(buffer, "%.4f\n\r", estimated_freq);
		uart_puts(buffer, USART1);
	    }

	    last4[temp_index] = estimate_frequency(mag, freq, max_index);

	    // float estimated_freq = estimate_frequency(mag, freq, max_index);
	    float estimated_freq = last4_average(last4, temp_index);
	    temp_index = (temp_index+1)%4;

	    float difference;
	    switch(max_index)
	    {
	        // case E: difference =  estimated_freq -  E_freq;
	        case E: difference =  estimated_freq/2 -  E_freq;
	            break;                                    
	        case A: difference =  estimated_freq/3 -  A_freq;
	            break;                             
	        case D: difference =  estimated_freq -  d_freq;
	            break;                             
	        case G: difference =  estimated_freq -  G_freq;
	            break;                             
	        case B: difference =  estimated_freq -  B_freq;
	            break;                             
	        case e: difference =  estimated_freq -  e_freq;
	            break;
	    }

	    uint8_t display_buff[NUM_BYTES];
	    tuning_instruction(difference, display_buff, max_index);

	    ssd1306_display_buffer(display_buff);
	    Delay(20);

	    buffer_count = 0;
	    TIM_ITConfig(TIM2, TIM_IT_CC1, ENABLE);
	    // break;
	}
    }
```
