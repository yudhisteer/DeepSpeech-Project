# Phase 1: Automating QC with DeepSpeech

In the Quality Control(QC) Department at RT Knits Ltd, we have people taking measurements of garments to ensure there are no discrepancies between the actual size of the garment and the desired one. If any aberration, the value is typed in an excel sheet.

After all the pieces(front panel, back panel, side panels,...) of the garments have been cut in the Cutting Department, they are assembled in the Make-Up Department. After being assembled, a group of people need to check if the final product matches the desired measurements given by the client. These QC people uses a tape measure to take the measurements and after each measurement between two points, they enter the value in a specific excel sheet, they then move on to the next measurements and continue as such. 

### Data on Current Procedure

Different clients require different types of measurement to be taken. For the purpose of this project, I collected data on measuring T-Shirts for ASOS. I timed the process and counted the number of T-shirts they need to measure and the number of people required to do so.

| Attempt | #1 | #2 | #3 | #4 | #5 | #6 | #7 | #8 | #9 | #10 | #11 | #12 |
| :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: | :---: |
| Seconds | 301 | 283 | 290 | 286 | 289 | 285 | 287 | 287 | 272 | 276 | 269 | 254 |

## Abstract





## Methods

As a first step, I chose to automate the process of data entry on the excel. Using Natural Language Processing(NLP) - the **DeepSpeech** model, I imagine the QC people to have a headset in which they would talk into and the system would recognise specific keywords, and populate their excel sheet. 

Inside the main() function, a while loop has been used to continuously input measurements. While the user does not say **“stop”**, the program keeps asking for measurements. Each measurement is recorded inside a list called **“data”**. The list holds the measurements for 1 garment at a time. Each measurement should be input in English. The conditions to alter the state of the program (i.e “next”,”stop”) should be input in English. If the user does not say “next” or “stop” when asked for input, the input is treated as a measurement and recorded in the list. If the user says “next” or “stop”, the list is saved to a dictionary by adding it as a definition, with the current garment number being the index. An integer variable “count” has been used to keep track of the garment number (garment number 1, garment number 2, etc...). The variable “count” is incremented. The list “values” is re-initialised to empty. If the user did say “next”, the loop starts again, allowing input of measurements for next garment. If the user said “stop”, the while loop stops executing.

(To be revised...)

## Plan of Action

1. Import Dependencies
2. Setup API for Google Sheet
3. Configuring DeepSpeech
4. Stream Audio
5. Check for keywords: "Start" & "Stop"
6. Check for keywords: "Next"
7. Clean Data
8. Data Entry

### 1. Import Dependencies

Libraries like ```word2number``` allow conversion of words into numbers that will be inserted into the Google Sheet. ```oauth2client.service_account import ServiceAccountCredentials``` allow setting up an instance on Google Cloud Console.

```
import time, logging
from datetime import datetime
import threading, collections, queue, os, os.path
import deepspeech
from word2number import w2n
import numpy as np
import pyaudio
import wave
import webrtcvad
from halo import Halo
from scipy import signal
from time import sleep
global global_data
global_data=[]
import gspread
from oauth2client.service_account import ServiceAccountCredentials
from pprint import pprint
```

### 2. Setup API for Google Sheet

 We start by creating a template on Google Sheet as shown below. ```Sample``` are the different garments that will be measured and ```specs``` are the different parts(sleeve, collar, hem,...) that need to be measured.
 
![image](https://user-images.githubusercontent.com/59663734/138092576-40613d98-4ffe-4062-9390-92c8112f4c05.png)


We initialize our google sheet with our credentials:

```
scope = ["https://spreadsheets.google.com/feeds",'https://www.googleapis.com/auth/spreadsheets',"https://www.googleapis.com/auth/drive.file","https://www.googleapis.com/auth/drive"]

creds = ServiceAccountCredentials.from_json_keyfile_name("creds.json", scope)

client = gspread.authorize(creds)

sheet = client.open("Template1").sheet1

logging.basicConfig(level=20)
```
### 3. Configuring DeepSpeech

DeepSpeech has been trained using machine learning techniques based on Baidu's Deep Speech Research Paper. It is an open-source speech to text engine which uses TensorFlow for easy implementation.

Fortunately, DeepSpeech already has some pre-built functions as follows:
- Filter & segment audio with voice activity detection.
- Generator that yields all audio frames from microphone.
- Generator that yields series of consecutive audio frames comprising each utterence, separated by yielding a single None.

We then need to stream from microphone to DeepSpeech using VAD:


### 4. Stream Audio

In the ```main``` function, we need to load DeepSpeech model and stream the speech from microphone to DeepSpeech using VAD.

```
def main(ARGS):
    # Load DeepSpeech model
    if os.path.isdir(ARGS.model):
        model_dir = ARGS.model
        ARGS.model = os.path.join(model_dir, 'output_graph.pb')
        ARGS.scorer = os.path.join(model_dir, ARGS.scorer)

    print('Initializing model...')
    logging.info("ARGS.model: %s", ARGS.model)
    model = deepspeech.Model(ARGS.model)
    if ARGS.scorer:
        logging.info("ARGS.scorer: %s", ARGS.scorer)
        model.enableExternalScorer(ARGS.scorer)

    # Start audio with VAD
    vad_audio = VADAudio(aggressiveness=ARGS.vad_aggressiveness,
                         device=ARGS.device,
                         input_rate=ARGS.rate,
                         file=ARGS.file)
    print("Listening (ctrl-C to exit)...")
    frames = vad_audio.vad_collector()

    # Stream from microphone to DeepSpeech using VAD
    spinner = None
    if not ARGS.nospinner:
        spinner = Halo(spinner='line')
    stream_context = model.createStream()
    wav_data = bytearray()
    for frame in frames:
        if frame is not None:
            if spinner: spinner.start()
            logging.debug("streaming frame")
            stream_context.feedAudioContent(np.frombuffer(frame, np.int16))
            if ARGS.savewav: wav_data.extend(frame)
        else:
            if spinner: spinner.stop()
            logging.debug("end utterence")
            if ARGS.savewav:
                vad_audio.write_wav(os.path.join(ARGS.savewav, datetime.now().strftime("savewav_%Y-%m-%d_%H-%M-%S_%f.wav")), wav_data)
                wav_data = bytearray()
            text = stream_context.finishStream()
```

### 5. Check for keywords: "Start" & "Stop"

The program will be activated upon hearing the word** "Start"** and will stop when the word **"Stop"** is registered. The data is stored as a string(for example: ```"start 23 next 34 next 45 stop"```) hence, we first check that the two keywords are present in the text and if so, we split the text and store it in an array called "data" : ```['start' '23' 'next' '34' 'next' '45' 'stop']```

The ```numpy.where()``` function returns the indices of elements in an input array where the given condition is satisfied. ```len(np.where(data == "start")[0]) > 0``` will return ```true``` if the word "start" is present. We store the index of the word "Start" using ```start = np.where(data == "start")[0][0]``` and print ```"the start commande is here"```.

```
if "start" and "stop"  in text.split():
  data=np.array(text.split())
  if len(np.where(data=="start")[0])>0:
     start=np.where(data=="start")[0][0]
     print("the start commande is here")
 else:
    print("the start commande is not here")

```

We do the same for the word "stop":

```
# We check if word "Stop" is in Text

 if len(np.where(data=="stop")[0])>0:
    stop=np.where(data=="stop")[0][0]
    print("the stop commande is here")
 else:
    print("the stop commande is not here")
```

### 6. Check for keywords: "Next"

We start by removing the "Start" and "Start" words in our array by by slicing our array using the variable ```start``` and ```stop``` created above. We store the data in between in the variable ```needed```= ```['23' 'next' '34' 'next' '45']```.

```
#We are removing "Start" & "Stop" from Text
if len(np.where(data == "stop")[0]) > 0 and len(np.where(data == "start")[0]) > 0:
   needed = np.array(data[start + 1:stop])
   print(np.array(data[start+1:stop])) #start = 0; stop = 6. Therefore we take only values of text from index 1 to 6(index 6 is not taken)
   print("needed: " + str(needed))
```
We then need to find indexes of the word "next" in the array ```needed``` and store them in the array ```nexts``` as such ```nexts = [1 3]```. Using the condition of length of the array ```nexts``` not equal to zero, we stored the first number and the last one in the array ```needed``` using the variable ```first``` and ```last``` respectively. We create an empty list called ```digits``` and appende dthe first number to it using the variable ```first```. 

We then write a ```for``` loop that will iterate through all the nunbers in between the first and last number and append it to the list ```digits```. We finish by appending the last number using the variable ```last```. The result is as such = ```digits = [array(['23'], dtype='<U5'), array(['34'], dtype='<U5'), array(['45'], dtype='<U5')]```.

```
if len(np.where(data=="stop")[0])>0 and len(np.where(data=="start")[0])>0:
    needed=np.array(data[start+1:stop])
    nexts=np.where(needed=="next")[0]
    if len(nexts)==0 :
        print("no parametre 'next' was detected")	
    elif len(nexts)!=0 :
        print(f"{len(nexts)+1} numbers are detected")
        first=needed[0:nexts[0]]
        last=needed[nexts[-1]+1:]
        digits=[]
        digits.append(first)
        for i in range(len(nexts)-1):
           digits.append(needed[nexts[i]+1:nexts[i+1]])
        digits.append(last)
```

### 7. Clean Data

We create an empty list called ```nums```. Using a ```for``` loop nested inside another ```for``` loop, we iterate through the values of the list ```digits``` and take only the numbers assigned to the variable ```empty``` and append it to the ```nums```.

```
nums=[]
empty=""
for i in digits:
    for j in i:
        empty+=str(j)
        empty+=" "
    nums.append(empty)
    empty=""
```

### 8. Data Entry

From the excel template I created, we need to start populating it as from ```B3``` which is the second column. Furthermore, each garments requires 7 measurements to be taken so for the first garment we need to do the entry from cell ```B3``` to ```B9.``` For the second garment, the entry will start from ```C3``` to ```C9``` and will continue as such for other garments. 

Variable ```column``` has an initial value 2(For the second column: ```B2```) which is updated after each 7 measurements. Using a ```for``` loop, we iterate through the values inside our list ```nums`` , convert them to numbers using ```w2n.word_to_num(nums[i])``` and populate the desired cell. 

```
#Columns has initial value of 2
columns=int(sheet.cell(1000,26).value)

for i in range(len(nums)):
 try:
     sheet.update_cell(3+i,columns,w2n.word_to_num(nums[i]) )
 except:
     print(f"a number speled wrong ?{nums[i]}")
     sheet.update_cell(3+i,columns,w2n.word_to_num("one"))
     
#Updating our variable columns     
sheet.update_cell(1000,26,int(sheet.cell(1000,26).value)+1)
```

## Testing

I did several tests and the result has been satisfactory except some few exceptions. One major drawback of the model is that it has been trained on US or UK English accents and
the model has not been accurately predicting the Mauritian accent. For example, the algorithm would confuse the word "next" with "necks" or "any" with "eight". There have been several instances of mispronounciation and below is a script that would replace the most frequent ones with the correct words:

```
# We replace mispronounced words

text=text.replace("necks","next")
text=text.replace("and","next")
text=text.replace("any","eight")
text=text.replace("men","next")
text=text.replace("that","next")
text=text.replace("starts","start")
text=text.replace("started","start")
text=text.replace("stopped","stop")
text=text.replace("for the","forty")
text=text.replace("for","four")	
print("Recognized: %s" % text)
```

## Conclusion

## Improvement
