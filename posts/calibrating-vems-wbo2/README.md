# Calibrating VEMS with WBO2



**DISCLAIMER**: Improperly setting any of the values can cause your vehicle to fail emission tests (as evident by how we got here to begin with) or run very poorly.  
I am not responsible for any damage you may suffer!

## Preparing the ECU

First, we need to prepare some things on the ECU-side.
So plug in your laptop and get VEMSTune started and connected to the ECU.  
Now go to `Base Setup` -> `ECU Calibration` and enter the following factory settings:

- `WBO2 Sensor 1 Ri target`: `165` (you never need to set this otherwise).
- `Pump PID nernst DC target`: `135`.
- `Pump Zero PW`: `100`.

If you have an email or something with your ECU order that specified the `Pump PID nernst DC target` and `Pump Zero PW`, then set your value to these values and you're basically done.  
However, verifying the settings isn't a bad idea, so let's go do that.  
Or if you don't have this email, then you also need to do these steps.

Next up, we need to unplug our WBO2 sensor.  
Since mine was already installed prior, it meant I had to dig into the engine bay a bit.
Luckily, for now, we are only interested in the ECU-side plug.

Once you have your connector easy to access, power up the ECU and voltmeter (or if you have a multimeter, set it to `volt`).  
Then when ready, connect the positive lead to `Pump+` (Pin1/Red wire) and the negative lead to `Pump-` (Pin2/Yellow wire).
If your reading by some miracle is `0.00` then congratulations, you're done with this step!  
However, chances are, it's not reading `0.00`, so we'll have to adjust the `Pump Zero PW` value until it does.  

- Increase the value to ???.
- Decrease the value to ???.

Now, we need to calibrate the `Pump PID nernst DC target`.  
To do this, with the WBO2 still unplugged, go to `Tools` => `WBO2 Calibration` and click `Start Cal` to enable the heater.  
Normally this should be done by sending `manlp02` in a serial terminal, but I cannot find it in VEMSTune so this will have to do.  
If someone knows how to do it otherwise from *inside VEMSTune* (no things like PuTTY), then let me know!
Measure the voltage between the `Nernst Cell Signal` (Pin 6/Black wire) and `Pump-` (Pin2/Yellow wire). 
Then adjust the `Pump PID nernst DC target` value until it reads `0.45V` on your voltmeter.  
You may need to restart the ECU after changing the value.
Once you've done this, then you're all done here and you should never have to re-adjust these settings!

## Calibrating the WBO2 itself

Now we need to calibrate for the WBO2 itself.  
To do this, you can do one of two things:
- Do it the easy route by using the WBO2 Calibration wizard in VEMSTune (free-air calibration): Useful when you still have to install everything anyways.
- The hardway with measuring and math: Useful when you are just replacing your sensor or already have everything installed.

### The easy way

TODO

### The hard way

If you choose the hardway, like I did, we're gonna need to measure something of our sensor. 
Each sensor has a laser-trimmed resistor inside the plastic housing which tells us *something* about the sensor (I don't know what though).  
As a result of this each sensor differs slightly so we still need to measure it to make sure our ECU doesn't misinterpret readings and cause your engine to run lean or rich while your ECU thinks you're running stoich (or even run rich when your ECU thinks you're already really lean).  
To measure this resistance take your multimeter and put it into `Ohm` mode and measure the resistance between `Pump+` (Pin 1/Red wire) and `Rcal` (Pin 5/Green wire).  
Look at your screen and remember that value (or even better: Write it down) as we're gonna need it.
Then, you can then set your `WBO2 Sensor 1 Calibration` to a value calculated using this formula: `(Rcal * 0.8) + 81.4`, where `Rcal` is the value we just measured.  
Here are some example values as an example:

| Rcal | Formula              | Calibration |
|------|----------------------|-------------|
| 80   | `(80 * 0.8) + 81.4`  | 145.4       |
| 100  | `(100 * 0.8) + 81.4` | 161.4       |
| 190  | `(190 * 0.8) + 81.4` | 233.4       |

Now plug in the sensor again and you should be all good.  
If you want to, you can still check by using [the easy way](#the-easy-way).

## Verification at a garage

## Final Thoughts
