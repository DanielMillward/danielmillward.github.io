+++
date = '2026-03-19T10:38:00-06:00'
draft = false
title = 'Rocket Propulsion Elements Book Notes'
categories = ["Rocketry"]
+++

This post is a compilation of the notes I've taken when reading the book [Rocket Propulsion Elements](https://www.amazon.com/Rocket-Propulsion-Elements-George-Sutton/dp/1118753658) by George Sutton. Please note that this won't cover everything the book writes about: I'm most interested in the design of pressure fed and pump-driven bipropellant liquid chemical rocket engines, so things like chapter 17 which covers electric propulsion will largely be ignored.

For equations, I'm planning on writing them in code since that makes it a little clearer to me.

All the units will be in the SI system of units.

# Chapter 1: Classification

Here's some typical values for rocket engines (the feature meanings will be explained later) (Table 1-2, p. 3):

| Feature        | Value           |
| ------------- |---------------|
| Thrust-to-weight ratio      | 75:1 |
| Specific fuel consumption (kg/hr-N), aka kg of fuel/hr per 1N thrust| 0.816-1.428      |
| Specific thrust (N/m^2), aka newtons of thrust per frontal area                          | 239.5k-1.1M   |
| Specific Impulse, thrust per unit of propellant per second                               | 270 sec       |
| Combustion Temperature                                                                   | 2500-4100 C   |
| Exhaust Velocities                                                                       | 1800-4300 m/s |

The "Thrust Chamber" is a reference to the injector, nozzle, and the combustion chamber itself.

Figures 1-3 and 1-4 (pages 6-7) show a high level diagram of a propulsion system for a pressure-fed and turbopump driven system respectively. Some components of note:

- Fuel filter: [Used for preventing debris from entering rocket](https://space.stackexchange.com/questions/65581/what-is-the-oxygen-filter-on-super-heavy-and-how-could-it-get-blocked-on-ift-3)
- [Regulators](https://en.wikipedia.org/wiki/Pressure_regulator): Controls the pressure coming from a high pressure tank to a lower one

Figure 1-4 (page 7) has a great diagram of a turbopump-driven propulsion system. I didn't like the image version I had, so I remade it as an svg here:

![A hand-made recreation of figure 1-4. A simplified schematic of a turbopump feed system.](./RPE-Figure-1-4.svg)