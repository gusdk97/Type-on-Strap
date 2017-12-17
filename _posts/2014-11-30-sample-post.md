---
layout: post
title: Our Code

---
```sh
import smbus
import time
import os
import RPi.GPIO as GPIO
import vlc
from ftplib import FTP

# Define some device parameters
I2C_ADDR  = 0x3f # I2C device address
LCD_WIDTH = 16   # Maximum characters per line

# Define some device constants
LCD_CHR = 1 # Mode - Sending data
LCD_CMD = 0 # Mode - Sending command

LCD_LINE_1 = 0x80 # LCD RAM address for the 1st line
LCD_LINE_2 = 0xC0 # LCD RAM address for the 2nd line
LCD_LINE_3 = 0x94 # LCD RAM address for the 3rd line
LCD_LINE_4 = 0xD4 # LCD RAM address for the 4th line

LCD_BACKLIGHT  = 0x08  # On
#LCD_BACKLIGHT = 0x00  # Off

ENABLE = 0b00000100 # Enable bit

# Timing constants
E_PULSE = 0.0005
E_DELAY = 0.0005

#Open I2C interface
#bus = smbus.SMBus(0)  # Rev 1 Pi uses 0
bus = smbus.SMBus(1) # Rev 2 Pi uses 1

def lcd_init():
    # Initialise display
    lcd_byte(0x33,LCD_CMD) # 110011 Initialise
    lcd_byte(0x32,LCD_CMD) # 110010 Initialise
    lcd_byte(0x06,LCD_CMD) # 000110 Cursor move direction
    lcd_byte(0x0C,LCD_CMD) # 001100 Display On,Cursor Off, Blink Off 
    lcd_byte(0x28,LCD_CMD) # 101000 Data length, number of lines, font size
    lcd_byte(0x01,LCD_CMD) # 000001 Clear display
    time.sleep(E_DELAY)

def lcd_byte(bits, mode):
    # Send byte to data pins
    # bits = the data
    # mode = 1 for data
    #        0 for command

    bits_high = mode | (bits & 0xF0) | LCD_BACKLIGHT
    bits_low = mode | ((bits<<4) & 0xF0) | LCD_BACKLIGHT

    # High bits
    bus.write_byte(I2C_ADDR, bits_high)
    lcd_toggle_enable(bits_high)

    # Low bits
    bus.write_byte(I2C_ADDR, bits_low)
    lcd_toggle_enable(bits_low)

def lcd_toggle_enable(bits):
    # Toggle enable
    time.sleep(E_DELAY)
    bus.write_byte(I2C_ADDR, (bits | ENABLE))
    time.sleep(E_PULSE)
    bus.write_byte(I2C_ADDR,(bits & ~ENABLE))
    time.sleep(E_DELAY)

def lcd_string(message,line):
    # Send string to display

    message = message.ljust(LCD_WIDTH," ")

    lcd_byte(line, LCD_CMD)

    for i in range(LCD_WIDTH):
        lcd_byte(ord(message[i]),LCD_CHR)
    
ftp = FTP("192.168.0.61","root","a1234")
a=[]
ftp.cwd('/music')
import os
ftp.retrlines('LIST',a.append)
#print("LIST")
print(a)
 
 
def download():
	inner = 0
	while (inner<len(a)):
		word = a[inner].split(None,8)
		#print(word)
		filename=word[-1].lstrip()
                #print(filename)
		#print(filename[0])
		n=0
		
		if filename[0]=='C':
                    #print("Christmas")
                    #channel path
                    cha="/home/pi/Downloads/Christmas Carol"
                    l=os.listdir(cha)
                    for i in l:
                        #print(i)
                        if filename != i :
                            n+=1
                    #print(n)
                    #print(len(l))
                    if n == len(l) :
        		local_filename = os.path.join("/home/pi/Downloads/Christmas Carol/" + filename)
        		ftp.retrbinary('RETR '+filename,open(local_filename,'wb').write)


		elif filename[0]=='K':
                    #print("Kids")
                    #channel path
                    cha="/home/pi/Downloads/Kids"
                    l=os.listdir(cha)
                    for i in l:
                        if filename != i :
                            n+=1
                    if n == len(l) :
        		local_filename = os.path.join("/home/pi/Downloads/Kids/" + filename)
        		ftp.retrbinary('RETR '+filename,open(local_filename,'wb').write)

                else:
                    #channel path
                    cha="/home/pi/Downloads/Else"
                    l=os.listdir(cha)
                    for i in l:
                        if filename != i :
                            n+=1
                    if n == len(l) :
        		local_filename = os.path.join("/home/pi/Downloads/Else/" + filename)
        		ftp.retrbinary('RETR '+filename,open(local_filename,'wb').write)

                #print(inner)
        		
		#local_filename = os.path.join("/home/pi/Downloads/ballad/" + filename)
		#print(local_filename)
		#If = open(local_filename,'wb')
		#print(1)
		#ftp.retrbinary('RETR '+filename,open(local_filename,'wb').write)
                #print(2)
		#If.close()
		#print(3)
		inner+=1
        return True


GPIO.setmode(GPIO.BCM)

GPIO.setup(18, GPIO.IN)
GPIO.setup(23, GPIO.IN)
GPIO.setup(24, GPIO.IN)
GPIO.setup(25, GPIO.IN)
GPIO.setup(26, GPIO.IN)


def main():
    # Main program block
    download()
    # Initialise display
    path = "/home/pi/Downloads/"
    a=os.listdir(path)


    lcd_init()
  
    temp=1 #music/channel
    i=1 #channel
    k=0 #music num
    exit=0 #EXIT
    z=1 #initial

    #read txt file
    f=open("/home/pi/mlist.txt", 'r')
    data=f.read()
    f.close
    #print(data) #erase
    file=data
    print(file)

    instance = vlc.Instance()

    #change to the latest music
    player = instance.media_player_new()
    file= data
    media=instance.media_new(file)
    player.set_media(media)
    player.play()
    time.sleep(1)


    while True:# start 
        num=player.get_state()


        #mode (channel/music)
        if GPIO.input(26)==0:
                player.stop()

                #temp==0:channel
                if temp==0:
                    temp=1

                #temp==1:music
                elif temp==1:
                    temp=0

        #channel mode
        if temp==0:
            player.stop()
            lcd_string("     CHANNEL    ",LCD_LINE_1)
            lcd_string("      MODE      ",LCD_LINE_2)                
            while True:
                lcd_string(str(a[i])+str(""),LCD_LINE_1)
                lcd_string(str(num),LCD_LINE_2)
                
                #previous channel
                if GPIO.input(18)==0:
                    time.sleep(1)
                    i=i-1
                    if i<0:
                        i=len(a)-1
                    #print(i) #erase

                #next channel
                if GPIO.input(24)==0:
                    time.sleep(1)
                    i=i+1
                    if i>=len(a):
                        i=0
                    #print(i)  #erase

                #mode
                if GPIO.input(25)==0:
                    temp=1
                    break

                #EXIT
                if GPIO.input(26)==0:
                    exit=1
                    break

        #music mode
        if temp==1:
            lcd_string("      MUSIC     ",LCD_LINE_1)
            lcd_string("      MODE      ",LCD_LINE_2)

            channel= "/home/pi/Downloads/"+a[i]
            print(a[i])
            print(channel)
            b=os.listdir(channel)
            b[k]=file
            print(b)
            
            if z>1:
                file=channel+"/"+b[0]
                media=instance.media_new(file)
                player.set_media(media)
                player.play()
                time.sleep(1)

            z+=1
            
            while True:

                #lcd
                #print(b[k])
                lcd_string(str(b[k])+str(""),LCD_LINE_1)
                lcd_string(str(num),LCD_LINE_2)
                num=player.get_state()
                
                #play next music, when the music is ended
                if num==6:
                    print("end")
                    k=k+1
                    if(k>len(b)):
                        k=0
                    player.stop()
                    file= channel+"/"+b[k]
                    media=instance.media_new(file)
                    player.set_media(media)
                    player.play()
                    time.sleep(1)

        	    #previous music
               	if GPIO.input(18)==0:
                    time.sleep(1)
                    lcd_string("    PREVIOUS    ",LCD_LINE_1)
                    lcd_string("                ",LCD_LINE_2)

                    #print("PREVIOUS",file,k)
                    
            	    k=k-1
            	    if k<0:
                        k=len(b)-1

                    player.stop()
                    file= channel+"/"+b[k]

                    print("previous",file,k)
                    
                    media=instance.media_new(file)
                    player.set_media(media)
                    player.play()
                    time.sleep(1)


        	    #pause
               	if GPIO.input(23)==0:
                
            		if num==3:
                                print(num)
            			player.pause()
            			
            			time.sleep(1)
            			num=4
                                print(num)
                                lcd_string("      PAUSE     ",LCD_LINE_1)
                                lcd_string("                ",LCD_LINE_2)
                                num=player.get_state()

            		elif num==4:
            			player.play()
               			time.sleep(1)
                                lcd_string("      PLAY      ",LCD_LINE_1)
                                lcd_string("                ",LCD_LINE_2)



        	    #next music
            	if GPIO.input(24)==0:
                    time.sleep(1)
                    lcd_string("      NEXT      ",LCD_LINE_1)
                    lcd_string("                ",LCD_LINE_2)

                    #print("NEXT ",file,k) #erase
                    
                    k=k+1
                    if(k>=len(b)):
                        k=0                        
                    player.stop()
                    file= channel+"/"+b[k]

                    #print("next ",file,k) #erase
                    
                    media=instance.media_new(file)
                    player.set_media(media)
                    player.play()
                    time.sleep(1)

                #mode
                if GPIO.input(25)==0:
                    temp=0
                    break


                if GPIO.input(26)==0:
                    time.sleep
                    player.stop()
                    exit=1

                    f=open("/home/pi/mlist.txt", 'w')
                    data=str(file)
                    print(data)
                    f.write(data)
                    f.close()
                    break

            #EXIT
            if exit==1:
        	lcd_string("      GOOD      ",LCD_LINE_1)
                lcd_string("      BYE       ",LCD_LINE_2)
	   	break

    


if __name__ == '__main__':

  try:
    main()
  except KeyboardInterrupt:
    pass
  finally:
    lcd_byte(0x01, LCD_CMD)

```
