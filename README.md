# Flowmeter
The code for the H2PI Flowmeter
#H2PI Flowmeter Program
#Created by Toby Cook
#Date:29/11/2017
import time as t
import RPi.GPIO as GP
import bluetooth
from PyOBEX.client import Client
#These import the necessary modules.


GP.setwarnings(False)
#Whilst usually considered bad practice, this is required as the pins
#cannot be "cleaned up" at the end of the program as it runs forever.
#However it is not a problem since the command to reset the pins sets them as inputs
#and because the program is designed to run on the Pi's startup, the pins will be set
#correctly each time.


GP.setmode(GP.BOARD)
GP.setup(7, GP.IN)
GP.setup(11, GP.OUT)
GP.setup(13, GP.OUT)
GP.setup(15, GP.OUT)
#These lines set up the GPIO pins.


GP.output(13, 0)
GP.output(15, 0)
GP.output(11, 1)
t.sleep(10)
GP.output(11, 0)
#This section resets the 7 segment display connected to the Pi.


count = 0
#This initialises a count used to control the clock input of the 4026 counters used
#in the display.


x = False
while x == False:
    nearby_devices = bluetooth.discover_devices()
    if len(nearby_devices) == 0:
        x = False
    else:
        x = True
        closest_device = nearby_devices[0]
#This loop searches for nearby bluetooth devices, then selects the first one it finds.      


service = bluetooth.find_service(uuid = '1105',address = closest_device)
port = service[0]['port']
client = Client(closest_device, port)
client.connect()
#This section attempts to connect to the device that the previous loop discovered.


condition2 = True
#This sets a condition true so that the rest of the program will run indefinetely


while condition2 == True:
    num = 0
    GP.output(11, 1)
    t.sleep(10)
    GP.output(11, 0)
    GP.wait_for_edge(7, GP.RISING)
    start = t.time()
    condition = True
    channel = 7
    #The section above checks to see if a logic 1 signal comes fro the flowmeter. If one is detected, the program will move on
    #to the next stage.
    while condition == True:
        pin_state = GP.wait_for_edge(channel, GP.RISING, timeout = 5000)
        #This checks to see if signals are still coming from the flowmeter.
        if pin_state is None:
            end_s = t.time() - start
            end_m = end_s / 60
            freq = num / end_s
            litres_min = freq / 7.5
            litre = str(round(litres_min * end_m, 2))
            litres = round(litres_min * end_m, 2)
            cost = str(round(litres * 0.0015288, 2))
            days = str(round(litres / 2.5, 2))
            if litres <= 50:
                client.put("Water Usage.txt", "You have Used:" + litre + "litres of water. This is less than the average volume of water used in a shower(45 Litres.) This is This cost you approximately:" + cost + "pounds, and this amount of water could have watered a person for:" + days + "days. Well done!")
            elif litres > 50 and litres <= 70:
                client.put("Water Usage.txt", "You have Used:" + litre + "litres of water. This is close to the average volume of water used in a shower. This cost you approximately:" + cost + "pounds, and this amount of water could have watered a person for:" + days + "days. Next time try to be less wasteful.")
            else:
                client.put("Water Usage.txt", "You have Used:" + litre + "litres of water. This is significantly more than the average volume of water used for a shower. This cost you approximately:" + cost + "pounds, and this amount of water could have watered a person for:" + days + "days. Next time try to be less wasteful. You could try taking shorter showers, use a regular shower instead of a power shower or consider getting a better shower head.")
            condition = False
            #This section breaks the nested while loop, when no more signals from the flowmeter are recieved and the program has sent a message to the device
            #connected by bluetooth depending on the amount of water used.
        else:
            num = num + 1
            if count != 44550:
                count = count + 1
                if count % 450 == 0:
                    GP.output(13, 1)
                elif count % 222.5 == 0:
                    GP.output(13, 0)
            #This section adds to the variable which stores how many litres of water has been used.
            #The count variable is also increased, and if it is a multiple of 450, sends a pulse to the clock input
            #as 450 pulses from the flowmeter equals 1 litre of water.
