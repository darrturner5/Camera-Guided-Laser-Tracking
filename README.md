# Camera-Guided-Laser-Tracking
Camera Guided Laser Tracking project that features a pan and tilt motions using 2 MG90 Servos and a laser attached to the tilt servo. It use a PD (Proportional-Derivative) Controller that is coded inside of Pythonto track a blue pencil sharpener.

## System Architecture and Details
 - 2 MG90 0-180 Servos
 - Class 3R Laser Diode
 - Arduino Uno R4 Wifi
 - Zipties
 - Active Buzzer
 - Logitech C270 1280x720p

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

