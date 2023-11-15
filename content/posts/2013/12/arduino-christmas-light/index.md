---
title: "Arduino Christmas Light"
date: "2013-12-17"
categories: 
  - "gadgets-gizmos"
  - "humour"
  - "personal"
---

Here's a short Arduino sketch that will fade and flash an LED on pin 9. Works a treat on a [Shrimp](http://shrimping.it/blog/ "Shrimping It") as well. Which is what mine runs on - my [Breaded Shrimp](http://start.shrimping.it/kit/stripboard.html "Breaded Shrimp") which stared like on a tiny breadboard like the one at the end of the above link, but which is now permanently soldered onto a piece of copper "breadboard" as you can see in the image that's around here somewhere.

What does it look like? Well, I can't upload a video for security reason (or so WordPress tells me _after_ I've uploaded the video!) Imagine, if you will, the LED in the image fading up from nothing to full brightness, flashing 3 times, then fading back down again where it flashes another three times before starting again.

Maybe it needs a little more work?

More LEDs?

What do you think?

I suppose I could call it minimalist? :-)

Maybe I'm just not cut out for this sort of work?

Happy Christmas, or whichever Winter Festival type celebration that you follow. Whatever it is, do have a nice one.

Anyway, here's the code:

```C
/*
 * This code is a cross between the "Fade" and "Blink" examples
 * Supplied with the Arduino IDE. Not much has changed, just the
 * removal of a few "magic" numbers.
 *
 * This example code is in the public domain.
 */

int brightness = 0;    // Brightness of the LED, right now.
int fadeAmount = 2;    // Fade the LED up/down by this amount each time.
int led        = 9;    // The LED is attached to this pin.
int flashDelay = 200;  // Pause between flashes of the LED.
int fadeDelay  = 20;   // Pause between fadeAmount changes.

void setup()  { 
  // declare pin 9 to be an output:
  pinMode(led, OUTPUT);
} 

void loop()  { 
  // set the brightness of pin 9:
  analogWrite(led, brightness);    

  // change the brightness for next time through the loop:
  brightness = brightness + fadeAmount;

  // reverse the direction of the fading at the ends of the fade: 
  if (brightness <= 0 || brightness >= 255) {

    // Flash the LED a few times at each end
    // of the fading cycle
    for (int x = 0; x < 4; x++) {
      digitalWrite(led, HIGH);
      delay(flashDelay);
      digitalWrite(led, LOW);
      delay (flashDelay);
    }
    
    // And now, reverse the fade.
    fadeAmount = -fadeAmount ; 
  }     
  // Pause a little, to see the fading effect    
  delay(fadeDelay);                            
}
```

Simple stuff, and very Open Source too. Take it, use it, abuse it if you like. Just have fun.

Cheers.
