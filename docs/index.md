# Welcome to Project Supernova / Project Coral!

Two years of aerospace engineering for the American Rocketry Challenge (ARC), where a rocket has to hit an exact apogee altitude and land inside a set interval of time with the payload (1-2 eggs) intact

**Project Supernova**: Lexington High School’s American Rocketry Challenge rocket, designed for active ascent/apogee control. I created the original “bowtie” (separation) design and spearheaded fabrication, which included 3D printing, wiring and soldering, and overall airframe assembly. My friend and teammate, Aleksei Sergeev, wrote the flight software, including Kalman-filter apogee prediction. Our total number of flights 2nd year was 10+ flights while first year was only around 3; we missed national qualification by 6 points corresponding to 1.5s of descent time

**Project Coral**: An independent project with Aleksei, outside the team, kind of like Supernova but active descent control. I designed and built the servo-winch parachute-reefing mechanism; Aleksei once again wrote the majority of the control code onboard the Arduino Nano. On the first test flight, the descent rate increased from 2 to ~5.5 m/s once reefing started

**Quick Summary** Start here:

1. [Supernova Year 2](supernova/supernovayear2.md) for the final design and results
2. [Coral Update 3](coral/thirdupdate.md/#results) for the reefing test flight
3. [Final Supernova launch code](code/supernovasecondyear/v10bailout.md)
4. [Coral launch code](code/reefcode/r03_reefing_kalman_state_machine.md)

All the code is on this site as a series of versions. The [codebase note](code/codebase.md) explains the organization
