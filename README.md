# PiLockMonitorHolmium

This is a small script which interfaces with the CsPyController (https://github.com/QuantumQuadrate/CsPyController) which allows for experiments to be immediately paused if a laser unlocks. The lock status is inferred from the image data from a camera peripheral on the Raspberry Pi. The primary intent of this is so that the impact of locks dropping is minimized by interrupting CsPy and waiting for the user to fix their laser locks before proceeding instead of continuing on and taking lots of data which is invalid due to the unlock.

# Functionality

The program effectively has two modes with regards to CsPy communication.
- Brightness mode: The program returns an average pixel brightness across the entire image. This is calculated in `get_avg_brightness()` and can be replaced with more sophisticated image analysis code if needed, but keep in mind that more computational overhead may increase latency for detecting a lock drop. In practice just calculating the average brightness across the whole image seems to be sufficient and typically detects drops well within the time of a single measurement.
- Experiment mode: The program waits for an initial prompt from CsPy stating that it's starting an experiment, and then takes in a threshold value for `get_avg_brightness()`. As long as the value for `get_avg_brightness()` remains above this threshold and doesn't drop below for more than a hard-coded (`frame_threshold`) number of frames in a row, the experiment will proceed like normal. If this condition fails, this program will tell CsPy that the lock dropped and CsPy will pause until it is told to continue.

The program remains continuously in Brightness mode until an experiment is started, and goes back to it once the experiment is finished.

# Usage

### Initial configuration
Run it as a server on a Raspberry Pi which the "Pi Lock Monitor" module of CsPy connects to, and ensure that the program stays running whenever anything is done on CsPy related to the module is enabled. Ensure that you have the correct IP addresses set (in the CsPy GUI and the hard-coded IP in the Pi code) and that the ports match. The IP check on this end may seem unnecessary, but there have been a couple occasions where some random unknown IP address has attempted to (and been rejected from) connecting to the Pi through this script. Probably better safe than sorry.

In principle, once this initial configuration is set up you should not have to touch the actual code on the Pi any more. The only exception would be some kind of a network change which changes the IP addresses.

### Per-experiment configuration (CsPy)
To get the proper threshold to use during an experiment, you need to get two brightness values. Lock your laser, and prompt it for a high "locked" brightness, and then either unlock the laser or physically block the beam from hitting the camera to get a low "unlocked" brightness. Set the threshold to be somewhere in between these two values before running the experiment.

A threshold closer to the "locked" brightness will make the monitor very sensitive to the smallest of brightness drops, but may also make it so that smaller perturbations from acoustic noise causes "false positives" on lock drop detection if it's too close. A threshold too close to the lower value may cause slower response times and, in the worst case, for mode-hops to other transverse modes to not be detected at all if they occur instanteously. For more unstable locks it is advised to take multiple brightness samples to get a sense of how much it changes on a frame-by-frame basis.

Once you are satisfied with your threshold value, make sure everything else on your experiment is prepped and then run it.

### During the experiment (CsPy)
*If your lock stays stable throughout the entire experiment, you shouldn't have to do anything extra compared to running with the Lock Monitor module disabled.*

If the lock drops during the experiment and the Pi detects it, the Pi program will print that the lock dropped with a timestamp and then tell CsPy to pause. To resume, relock your laser and choose `Run/Continue` on CsPy to resume the experiment. Note that if the laser isn't relocked, the monitor will end up pausing again on the next measurement anyway.

This pause occurs at the end of the current measurement, so every time a lock drops that measurement should be considered bad. This doesn't actually mark the measurement as bad in CsPy though, as it doesn't handle bad measurements properly at the time of writing this read-me. The impact of bad measurements can be minimized by running more measurements per iteration, which of course has its own pros and cons independent of the lock monitor.

### Troubleshooting
These are bugs which have existed in the past and should be fixed as I was unable to reproduce them, but if they still exist then these are the workarounds I used:
- The lock monitor thinks the experiment ends immediately after it starts or thinks that the experiment has been restarted multiple times within a few seconds. This seems to be caused by an experiment on CsPy halting improperly, such that messages being sent are backed up in the socket queue and/or too many messages are being sent. Restarting both CsPy and the lock monitor has been the most reliable fix for this in my experience.
- When stopping the lock monitor and starting it up, it refuses to reconnect and spits out a "Address already in use" error. This seems to happen if CsPy doesn't properly close the socket for some reason. The connection will time out after at most a minute, so repeatedly attempting to start the monitor will eventually work.
- The program seems to think the connection dropped if nothing happens for a couple hours or so. I think this happens due to the connection being inactive, and two possible workarounds are to run shorter experiments (may not be a viable option) or to make the monitor think the laser unlocked (by, for example, blocking the camera briefly) every hour or so to keep the connection active.
