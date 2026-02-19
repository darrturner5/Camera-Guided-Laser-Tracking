# Camera-Guided-Laser-Tracking
Camera Guided Laser Tracking project that features a pan and tilt motions using 2 MG90 Servos and a laser attached to the tilt servo. It use a PD (Proportional-Derivative) Controller that is coded inside of Pythonto track a blue pencil sharpener.

## System Architecture and Details
 - 2 MG90 0-180 Servos           - 
 - Class 3R Laser Diode
 - Arduino Uno R4 Wifi
 - Zipties
 - Active Buzzer
 - Logitech C270 1280x720p

 My Logitech C270 captures frames at about 30FPS. I applied a Mask to detect only teal colors to track my pencil sharpener. The mask makes everything in the background black, except the teal color. Then I was able to detect contours of the shape of the pencil sharpener which allowed me to calculate the centroid using the moments function. This is important because this tells us our measured value which we can use to find the error and put into the controller. The error is filtered and normalized into a -1 to 1 scale to make the PD Controller easier to tune. I added servo clamps to prevent damage to the servos and buzzer logic that locks after 2 seconds. All of this is then sent over the Serial to Arduino.
  
 

![IMG_8340](https://github.com/user-attachments/assets/9d49a03f-f897-4603-bac9-5ddf9d058d10)
![IMG_8338](https://github.com/user-attachments/assets/bd765783-9e30-4e34-b23d-202bcfb552c9)
*Wiring could be a little neater.... But atleast it works!
![IMG_8339](https://github.com/user-attachments/assets/898c2137-c6b4-4951-90e7-a3b86956eaff)

## Propotional - Derivative Controller
First I want to introduce a brief discussion on what each P and D terms actually do in the system. Starting with P control, the equation is U = Kp * Error
 - U = Output
 - Kp = Gain (How strong the correction is)
 - Error = Measured Value - Setpoint (In my case the Measured value is where on screen is the Pencil Sharpener - the middle of the screen being my setpoint.)

The output (Correction) is proportional to the error. Meaning wherever that error is, the system makes a correction to match the error. This is an ongoing process as these conditions change as the object moves. This is what we call a Closed Loop system.
Proportional control is really good to be used on its own with the correct tuning however there are some caveats to look out for:
  - Overshoot
  - Oscillations
    
[![What does Oscillations and Jittering look like?](<img width="348" height="553" alt="image" src="https://github.com/user-attachments/assets/60fb20a3-53fb-4ac4-90b5-5e64c725417c" />
)](https://www.youtube.com/shorts/U7X-p9T4rxE)

It can cause some jittering and overcorrecting around the setpoint which can make precision things that rely on this type of control less reliable. In my project, I was seeing this effect in real time on my tilt servo. The servo would jitter and oscillate up and down when objects came to a complete stop. Since I wanted the laser to be directly onto the target, I opted for the (D) Derivative term and see if I can add this to the controller and see if it might help mitigate this issue from the P controller.

The (D) term measures the rate of change of the error. The expression is Kd * de/dt
 - Kd = Gain
 - de = change in error (current error - previous error)
 - dt = time

When added together, The Derivative term dampens the oscillation and overshooting problem but comes with a whole other set of problems:
 - (D) is very sensitive to any change of error (can even be one pixel)
 - Any noise or pixel that moves gets amplified by the expression and makes the output chaotic.

I learned of this issue when It caused my whole servos to start glitching far left, far right, far up and  far down. This means that before even putting in the (D) term, you would have to add a low pass filter to the error to get rid of the noise coming from the camera.

The filter I used is Simple Exponential Smoothing

<img width="396" height="62" alt="image" src="https://github.com/user-attachments/assets/9c8e0092-9dc8-4d54-8a3d-de3af2b5997a" />

In my case:
- St = smoothed error
- Alpha = smoothing factor
- Y = error
- St-1 = Previous error

In my code:
- output = error * 0.7 + (1-0.7) * prev_error
- prev_error = error

When combined, You get the PD Controller:

U = Kp * error + Kd*de/dt

# Performance
Below is a complete showcase of the PD Controlled Camera Guided Laser System:

[![PD CONTROL SHOWCASE](<img width="348" height="553" alt="image" src="https://github.com/user-attachments/assets/60fb20a3-53fb-4ac4-90b5-5e64c725417c" />
)](https://www.youtube.com/watch?v=3hu1fUoF0aM)

 The (D) term does a really nice job of slowing down and reducing the jitter that I was experiencing earlier. The system tracks really smoothly and fast with slight amounts of jittering and has good accuracy because of the tuned P control.

## Notes
Upon using the (D) Term Ive learned to use Kd gain relatively small. I started my gain around 0.1 and did not know what to expect. Instantly my servos made full revolutions left, right, up, and down. 

[![What does Derivative Noise Amplification look like in a system?](<img width="348" height="553" alt="image" src="https://github.com/user-attachments/assets/60fb20a3-53fb-4ac4-90b5-5e64c725417c" />
)](https://www.youtube.com/watch?v=Cp3NwTRCM4U)

While most of this occured due to the noise amplification of the random pixels from the camera, The Derivative term is still very sensitive even after the filtering. But did help significantly in the amount of noise that it was amplifying and made it a bit more stable. I then lowered the gain excessively small to 0.0005 which looks like nothing but reduced most of the jitter.
After seeing the damage that this term can cause, I opted for a clamp that wouldnt exceed -50, 50 pixels per frame.

I think here would be a great time to talk briefly about the buzzer logic:

- lock_width = 350 # pixels
- lock_time = 2.0 # seconds
- lock_timer = 0.0
- dt = 0.033 # Time per each frame 30fps

            if abs(error_x) < lock_width and abs(error_y) < lock_width:
                lock_timer += dt #30 FPS or change in time that passed since the last frame
            else:
                lock_timer = 0
*if the error ( Measured Value - Setpoint) is less than 350 pixels, (meaning the setpoint is very close to the measured value (Object))*

*Start timer*

*if the error is not less than 350 pixels, reset timer.*

            locked = 1 if lock_timer >= lock_time else 0
            
*When the timer is greater than or equal to 2 seconds, send 1 over Arduino Serial.*

- The accuracy and timing of this depends on the frames per second of the camera. If you have a 60fps camera, Id suggest doubling the dt to 0.066.
- I chose 350 pixels because it was wide enough to capture the whole object. Do not be afraid to make the width much higher such as <= 500 pixels depending on the object you may want to track. Small pixels values less than 150 may not activate the logic because my contours and centroid were always moving (mostly due to lighting) so the bigger the better.


## Tuning:
 - Started with P Control Tuning only, Kd = 0
 - Tuned until there was slight oscillation but still fast enough to track errors
 - Started with an entrememly low Kd Gain, Slowly build up (Too high gain causes too much amplification)
 - Clamp the Derivative amplification (Protects servos)

## Challenges
- Laser did not line up exactly with the target (Offset both servos)
- Mechanical mounting of the Pan and Tilt Servos 
- Servo Jitter  (Addeded PD Controller)
- Derivative kick (Added Clamping)

## How can I improve?
- Finite State Logic with OLED
  
This the one thing Ill definitely play around with. I never used an OLED Screen before and would definitely give that a shot. Especially matching with my buzzer logic. I either want it to say "LOCKED" when it makes the buzzer noise or I may display the time on the screen before it locks.

## HOW TO BUILD
This is a very exciting project and I am very glad and grateful to be sharing this with whomever might be reading. If you can improve on it definitely do so and show me what you've done :)

*ARDUINO*
- Connect servos digital pins to 5 and 6
- xPin = 5
- yPin = 6
- 5V to Power supply (NOT ARDUINO 5V)
- Shared Ground 

  
- Connect Active buzzer (+)
- digital pin 9
- Active buzzer (-) to shared ground


*MAKE SURE ALL DEVICES SHARE THE SAME GROUND!*

I designed this little breakout board to resemble a breadboard. Although you can still use a breadboard. I added a 25V 1000uF Capacitor in parallel between the (+) Rail and (-) Rail to help with current spiking (Servos may draw a lot of current so watch out for that).
- 5V RAIL ON RIGHT ( Servos, and Laser driver circuit)
- Negative Rail (-) on LEFT (Shared Ground. Every device leads back to this!!) (Buzzer, Servos, Laser circuit)
  ![IMG_8345](https://github.com/user-attachments/assets/6312b93a-f315-4c25-95d4-d18612d6b02b)


The actual construction of the Servos took a lot of trial and error and offsetting so adjust to your build.
-  I used Zip ties to hold the two servos together (If you have a better Idea go ahead)
-  The X servo goes to the bottom. This is the pan motion. It moves left to right
-  The Y servo goes on top but facing sideways so it can replicate a up and down motion. This is the Tilt motion.

          servo_x = int(90 - correction_x*40) + 18
          servo_y = int(90 - correction_y*40) + 3
   
*mapping of the servos in Python*


- The constants at the end of the code represents the offsetting that Ive done to achieve the laser to lock on fully with the object.
- These constants are different for every build especially since not all with be aligned the same. You will have to do a little tuning there to get it just right.
- Some systems may need it, some systems might not. it all comes down to how well aligned your pan and tilt servos are.
- Alternatively, if you have a 3D printer and can design a proper frame, that will reduce the need for the offsetting anyways.
- One more note, If you can get a proper laser pointer, you do not need a laser driver circuit. ust mount the laser pointer to the Tilt motion. That reduces the need for all that extra wiring thats clumped up in my build.

## SKILLS DEMONSTRATED
- Closed Loop Control
- Embedded Systems
- Computer Vision
- System Integration (Python and Arduino)
- System Tuning




