---
layout: default
---

# Programming test

Please see below for our programming test. Read the instructions and upload your
submission to our Dropbox as a compressed file (zip/tar) once you're done.

<a href="https://www.dropbox.com/request/hPxLR2JmZr8BMEMDfwY8"
class="button">Upload your submission</a>

Thanks!

# Instructions

The purpose of the exercise is to evaluate your approach to software development
including object-oriented design, design patterns and testing.

- Complete the exercise in the language of your choice.
- We recommend you spend 2-3 hours on your submission.
- Structure your code as if this were a real production application.
- State any assumptions you make as comments in the code.
- Please include a readme on how to run your program.

## The problem

A small airline wants a simple program that produces flight summary reports
based on flight, route and passenger data.

There are three types of passenger the airline will cater for:

1. General – normal fare-paying passengers.
2. Loyalty Members – repeat customers who get benefits for choosing to fly with
   the airline.
3. Airline Employees – employees of the airline who fly with the airline for
   free.

For each flight the airline charges a base ticket price for a specific route.
Loyalty members can choose to pay with their loyalty points instead.
Loyalty points are worth £1 each. Airline employees always fly free. All
passengers are allocated 1 bag and loyalty members are allowed 1 extra bag. For
simplicity, we assume that every passenger will bring at least 1 bag.

## Your task

Write a console application that accepts two filenames, the first is an input
file, containing route, plane and passenger data, the second is an output
file to which the flight summary report must be written.

### Input

The format of the input file is a set of lines that represent either plane,
route or passenger information. Your program should read each line in the input
file and process each instruction.

For example:

```
add route London Dublin 100 150
add aircraft Gulfstream-G550 8
add passenger airline Trevor
add passenger general Mark
add passenger loyalty Joan 100 FALSE TRUE
```

```
add route {origin} {desination} {cost-to-airline-per-passenger} {ticket-price}
add aircraft {aircraft-model} {total-seats}
add passenger general {passenger-name}
add passenger airline {passenger-name}
add passenger loyalty {passenger-name} {current-loyalty-points} {using-loyalty-points} {has-extra-bag}
```

Loyalty passengers are able to use loyalty points for discounts against tickets.

If they are using their loyalty points then £1 is taken off the price
of the ticket for every loyalty point used. These customers then pay the
remainder of the normal ticket price.

Loyalty passengers have the option to bring an extra bag.

An input file must add only one route and one aircraft.

### Output

Your program should read the input file, compute a flight summary report and
write it to the output JSON file.

```
{
  "passengers": {number}, // total number of passengers on the flight
  "generalPassengers": {number}, // number of general passengers on the flight
  "airlinePassengers": {number}, // number of airline passengers on the flight
  "loyaltyPassengers": {number}, // number of loyalty passengers on the flight
  "bags": {number}, // the total number of bags on the plane
  "loyaltyPointsUsed": {number}, // the total number of loyalty points redeemed by all passengers
  "costOfFlight": {number}, // the total cost to the airline of running the flight
  "revenueBeforeDiscounts": {number}, // the total ticket revenue, ignoring loyalty and airline passenger adjustments
  "revenueAfterDiscounts": {number}, // the total ticket revenue, after adjusting for loyalty members points and airline passengers
  "canFlightProceed": {boolean} // can the flight proceed, according to the rules defined below
}
```

### Flight rules

A flight proceeds only if all of the following rules are met:

1. The total adjusted revenue for the flight exceeds the total cost of the
   flight.
2. The number of passengers does not exceed the number of seats on the plane.

### Example input and output

#### Input file
```
add route London Dublin 100 150
add aircraft Gulfstream-G550 8
add passenger general Mark
add passenger general Tom
add passenger general James
add passenger airline Trevor
add passenger loyalty Alan 50 FALSE FALSE
add passenger loyalty Susie 40 TRUE FALSE
add passenger loyalty Joan 100 FALSE TRUE
add passenger general Jack
```

#### Output file
```
{
  "passengers": 8,
  "generalPassengers": 4,
  "airlinePassengers": 1,
  "loyaltyPassengers": 3,
  "bags": 9,
  "loyaltyPointsUsed": 40,
  "costOfFlight": 800,
  "revenueBeforeDiscounts": 1200,
  "revenueAfterDiscounts": 1010,
  "canFlightProceed": true
}
```

This flight can proceed.

#### Input file

```
add route London Dublin 100 150
add aircraft Gulfstream-G550 12
add passenger general Mark
add passenger general Tom
add passenger general James
add passenger airline Jack
add passenger airline Jane
add passenger airline Steve
```

#### Output file

```
{
  "passengers": 6,
  "generalPassengers": 3,
  "airlinePassengers": 3,
  "loyaltyPassengers": 0,
  "bags": 6,
  "loyaltyPointsUsed": 0,
  "costOfFlight": 600,
  "revenueBeforeDiscounts": 900,
  "revenueAfterDiscounts": 450,
  "canFlightProceed": false
}
```

This flight cannot proceed, as revenue after discounts is less than the cost
of the flight.
