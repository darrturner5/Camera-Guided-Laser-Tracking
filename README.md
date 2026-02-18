# Camera-Guided-Laser-Tracking
Camera Guided Laser Tracking project that features a pan and tilt motions using 2 MG90 Servos and a laser attached to the tilt servo. It use a PD (Proportional-Derivative) Controller that is coded inside of Pythonto track a blue pencil sharpener.

## System Architecture and Details
 - 2 MG90 0-180 Servos
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


## Notes
Upon using the (D) Term Ive learned to use Kd gain relatively small. I started my gain around 0.1 and did not know what to expect. Instantly my servos made full revolutions left, right, up, and down. 
[![What does Derivative Noise Amplification look like in a system?](<img width="348" height="553" alt="image" src="https://github.com/user-attachments/assets/60fb20a3-53fb-4ac4-90b5-5e64c725417c" />
)](https://www.youtube.com/watch?v=Cp3NwTRCM4U)

While most of this occured due to the noise amplification of the random pixels from the camera, The Derivative term is still very sensitive even after the filtering. But did help significantly in the amount of noise that it was amplifying and made it a bit more stable.
After seeing the dmage that this term can cause, I opted for a clamp that wouldnt exceed -50, 50 pixels per frame.

Buzzer Logic:

For the Tuning Process:
 - Started with P Control Tuning only, Kd = 0
 - tuned until there was slight oscillation but still fast enough to track errors
 - started with an entrememly low Kd Gain

## Challenges
- Laser did not line up exactly with the target
- Some lag behind the servos
- Tuning process
- Mechanical mounting of the Pan and Tilt Servos
- Wire Management

## How can I improve?
- Finite State Logic with OLED
  
This the one thing Ill definitely play around with. I never used an OLED Screen before and would definitely give that a shot. Especially matching with my buzzer logic.


