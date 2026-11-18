# Full Contributions

Both projects were built mostly by two people: me and my friend and teammate Aleksei Sergeev. The split was done in a fairly common way: I handled the physical side (design, CAD, and fabrication) and Aleksei wrote the code. I can follow the software closely enough to explain how it works (Kalman Filter, state machine), but vast majority of the code is his

## Project Supernova

Supernova was our school club's American Rocketry Challenge rocket, and most of the work on it came from the two of us

**Gleb Kozlov (my) Contributions:**

- Came up with the original "bowtie" separation latch, which holds the two halves together until apogee and then rotates to release them
- When the first launch barely separated, added a rubber-band-loaded "push rod" (separates the two halves) and anti-rotation teeth (prevents two halves from rotating against each other when bowtie rotates)
- In Year 2, replaced the push rod with nested springs, switched to a slanted rotation surface for less difficult separation, and moved to a way beefier 9 kg/cm servo. Additionally changed anti-rotation teeth to different shape profile (before was equilateral triangle, changed to quadrilateral)
- Designed the snap-fit electronics container, important for electronics organization
- Did almost all fabrication: 3D printing, cutting body tubes, epoxying, wiring and soldering. 3D printing included a variety of material testing, using materials like PLA for testing and quick iterations while also learning about filaments like Kevlar-reinforced PLA and nylon filaments for their exceptional strength (PA-CF and PA-GF)

**What Aleksei did:** wrote the flight software in both years, including the Kalman-filter apogee prediction that flew at launch (v10). His contribution cannot be overstated enough

## Project Coral

Coral was just Aleksei and me, in my basement (outside club), building over the summer to fix the descent-time problem because prior we had been relying on wind being favorable

**What I did:**

- Designed and built the reefing mechanism, taking the winch through a multitude of different versions before landing on the final spool
- Designed the test rocket in OpenRocket and CAD, including the fin can and the transition channel that keeps the shock cord from tangling
- Basic terminal velocity calculations and predictions
- Wiring/soldering of the flight circuit

**What Aleksei did:** Once again, he wrote the control code (r01–r03), including the final reefing state machine

**What we did together:** built the parachute, ran the drop and fan tests, and launched the full-scale test on August 15, 2026. Descent speed went from about 2 m/s to about 5.5 m/s once the winching started

All of my CAD is linked on the [CAD and 3D-design](cadfile.md) page