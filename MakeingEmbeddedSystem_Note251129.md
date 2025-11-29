# Debug
When setting a break point, there are 2 typical way to do that
1. Changed the code directly before loading it to flash
2. If the system is running outside of flash, then, there will be some register to be used to store the breaking point address. And the system(or debugger?) has to compare the address everytiem when it advance the PC point(I guess?)

# Challanges
1. Crossing compile and debugger
2. Bring out unit, which mixes hardware and software issue
3. Manufactor
4. Remote debug for shipped unit
5. Changes among the developement cycle

# Some Design Principles
* Modularity
* Test it
* Documenting as you can understand your code 1 year later
* Don't try to optomize everything. Make it work, test and do optomize

# Design Architecture
## How We Start
* From blank state
* Starting from existed project
## The Directions
* The HW is fixed, we design the software to meet the HW
* The features(software) design first, and pick the correct HW to meet the SW

## Diagrams
This book suggests to use diagrams to start our system design because we will have more reliable code if we can better understand of the whole design.

The following diagram should be design by order

### Context Diagram
It's actually the user story

### Block Diagram
This diagram will usually start from the HW block diagram. Each HW component will also be an indenpendt SW block. After that we should add more component(box) to the diagram for thing like:
* HW driver like bus driver(SPI, UART), GPIO, clock.
* Contoller for driver(more like high level driver), namingly, Flash, LED, Logging, Sensor.

Basically, just put in boxes as many as possible at this stage. We can go back and do some optomization by group boxes.