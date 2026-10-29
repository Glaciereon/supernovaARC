# Progress Report #3 (final!)
We have decided to bite the bullet and construct an entire rocket to test the effectiveness of the reefing design. It would have been much more preferred to use a small-scale prototype, but with significant time constraints (my teammate’s college departure, who's coding has been critical to this project) production has been rushed to making a full rocket with the integrated reefing mechanism. The first step in the construction of this rocket, as always, is to plug in basic parameters into the software OpenRocket, which will calculate/approximate basic stability and apogee details. 

![openrocketimage](images/openRocketReefing.png)

The most crucial details when designing a basic rocket like this are the weights of individual components (fins, nose cone, ‘mass component’ in the center to represent avionics, etc) and then match them as closely as possible in the real world. This can be achieved through modifying infill and print material with FDM printing. 

The aerodynamic stability of the rocket, measured in calibers, is approximately 1.52, which is considered well within the ‘sweet spot’ of 1-2 calibers for model rockets to both remain . This value simply measures how many rocket body (body tube) diameters the center of gravity is ahead of the center of pressure. This value is important as it ensures the rocket won’t tumble due to being under stable or ‘weathercock’ (turn into the wind due to stability being too high). The intuitive explanation can be given through this NASA model rocketry diagram: 

![nasaweathercockimage](images/nasaWeatherCock.jpeg)    

It was quite simple to model all the corresponding parts that are associated with the OpenRocket - making a fin can with the same fin dimensions, a simple nose cone, etc. The body tube is one diameter continuous, that being BT-80 (66mm outer, 65mm inner diam) thus not requiring dubious levels of modeling for a transition piece between the two of them and matching two different diameters. The parts are always modeled with 0.05cm of wiggle room to account for thermal expansion.

![projectcoralfincan](images/projectcoralfincan.png)
![projectcoralnosecone](images/projectcoralnosecone.png)

Once the fin can was modeled, it was printed out of basic PLA and then epoxied onto the body tube, with one half of the shock cord attached to a drill hole within that fin can.


![projectcoraltransitiontop](images/projectcoraltransitionTOP.png)
![projectcoraltransitionside](images/projectcoraltransitionSIDE.png)

It can be noted that the transition looks quite unique, with a 'channel' running down the side and a small hook section that leans into the body of the rocket. This was done intentionally, so that the line of the shock cord would be less tangled with the parachute itself or associated shroud lines prior to the two halves separating from the ejection charge. It also allowed for us to mount the servo in a vertical fashion (with the axis of rotation being in line with the body tubes), as the dimensions of the servo would have made it difficult to fit in any other orientation. To make the string naturally collect on the spool due to gravity, this resulted in a requirement of horizontal feeding. The spool, naturally, has raised edges as to prevent the shock cord from slipping off during winching. 
