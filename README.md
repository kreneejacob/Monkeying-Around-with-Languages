# Monkeying-Around-with-Languages
A fun and educational tool utilizing Arduino Uno and a Wave Shield to help teach young children various body parts in English, Spanish, and Italian.  
// This sketch has been modified from Adafruit's Play6_HC Example (https://learn.adafruit.com/adafruit-wave-shield-audio-shield-for-arduino/play6-hc) in order to create a teddy bear that will teach its endusers body part names in English, Spanish and Italian. 

//We used this reference to construct the sound shield from the kit
//https://learn.adafruit.com/adafruit-wave-shield-audio-shield-for-arduino/solder

//MP3 Files Converted to WAV 
//https://drive.google.com/drive/folders/1htgQU5QT2gCqBpdkzU1bkl9KECTQxgz_?usp=sharing

#include <FatReader.h>
#include <SdReader.h>
#include <avr/pgmspace.h>
#include "WaveUtil.h"
#include "WaveHC.h"


SdReader card;    // We loaded our recorded sound files onto an SD card that we then put in the shound shield. These audio files must be PCM 16-bit Mono Wave files at 22KHz samplerate. We used audacity to convert.
FatVolume vol;    // This holds the information for the partition on the card
FatReader root;   // This holds the information for the filesystem on the card
FatReader f;      // This holds the information for the file we're playing

WaveHC wave;      // This is the only wave (audio) object, since we will only play one at a time

#define DEBOUNCE 5  // button debouncer

//Pinds 6,7,8,9 are free to use on the sound shield. We define the button pins below
byte buttons[] = {6, 7, 8, 9};
// Define the number of buttons by finding the length of the button array above. This will be used to create loops to run the sound filed when buttons are pressed. 
#define NUMBUTTONS sizeof(buttons)
// track the button state using the following state identifiers
volatile byte pressed[NUMBUTTONS], justpressed[NUMBUTTONS], justreleased[NUMBUTTONS];

// this function will return the number of bytes currently free in RAM
int freeRam(void)
{
  extern int  __bss_end; 
  extern int  *__brkval; 
  int free_memory; 
  if((int)__brkval == 0) {
    free_memory = ((int)&free_memory) - ((int)&__bss_end); 
  }
  else {
    free_memory = ((int)&free_memory) - ((int)__brkval); 
  }
  return free_memory; 
} 
// RAM function ends here

//checks if SD card error. If so, no sound can be played and error message will be sent.  
void sdErrorCheck(void)
{
  if (!card.errorCode()) return;
  putstring("\n\rSD I/O error: ");
  Serial.print(card.errorCode(), HEX);
  putstring(", ");
  Serial.println(card.errorData(), HEX);
  while(1);
}

void setup() {
  byte i;
  
  // set up serial port
  Serial.begin(9600);
  putstring_nl("WaveHC with ");
  Serial.print(NUMBUTTONS, DEC);
  putstring_nl("buttons");
  
  putstring("Free RAM: ");       // This can help with debugging, running out of RAM is bad
  Serial.println(freeRam());      // if this is under 150 bytes it may indicate an error
  
  // Set the output pins for the DAC control. This pins are defined in the library
  pinMode(2, OUTPUT); // These pins relate to the sound shield.
  pinMode(3, OUTPUT); // These pins relate to the sound shield.
  pinMode(4, OUTPUT); // These pins relate to the sound shield.
  pinMode(5, OUTPUT); // These pins relate to the sound shield.
 
  // pin13 LED
  pinMode(13, OUTPUT);
 
  // Make input & enable pull-up resistors on switch pins
  for (i=0; i< NUMBUTTONS; i++) {
    pinMode(buttons[i], INPUT);
    digitalWrite(buttons[i], HIGH);
  }
  
  //  if (!card.init(true)) { //play with 4 MHz spi if 8MHz isn't working for you
  if (!card.init()) {         //play with 8 MHz spi (default faster!)  
    putstring_nl("Card init. failed!");  // Something went wrong, lets print out why
    sdErrorCheck();
    while(1);                            // then 'halt' - do nothing!
  }
  
  // enable optimize read - some cards may timeout. Disable if you're having problems
  card.partialBlockRead(true);
 
// Now we will look for a FAT partition!
  uint8_t part;
  for (part = 0; part < 5; part++) {     // we have up to 5 slots to look in
    if (vol.init(card, part)) 
      break;                             // we found one, stop searching
  }
  if (part == 5) {                       // if no FAT partition is found, do this
    putstring_nl("No valid FAT partition!");
    sdErrorCheck();      // Something went wrong, send error message
    while(1);                            // then 'halt' - do nothing!
  }
  
  // Lets tell the user about what we found
  putstring("Using partition ");
  Serial.print(part, DEC);
  putstring(", type is FAT");
  Serial.println(vol.fatType(),DEC);    
  
  // Try to open the root directory where sound files are stored
  if (!root.openRoot(vol)) {
    putstring_nl("Can't open root dir!"); // Something went wrong,
    while(1);                             // then 'halt' - do nothing!
  }
  
  // If above is successfully completed, ready to run
  putstring_nl("Ready!");
  
  TCCR2A = 0; // starts timer
  TCCR2B = 1<<CS22 | 1<<CS21 | 1<<CS20;

  //Timer2 Overflow Interrupt Enable
  TIMSK2 |= 1<<TOIE2;


}

SIGNAL(TIMER2_OVF_vect) {
  check_switches();
}

void check_switches()
{
  static byte previousstate[NUMBUTTONS];
  static byte currentstate[NUMBUTTONS];
  byte index;
// this creates a loop to run through. Each button's current state is continually checked for in oder to know if a response is necessary, i.e. if a sound should be played
  for (index = 0; index < NUMBUTTONS; index++) {
    currentstate[index] = digitalRead(buttons[index]);   // read the button
    
    /*     
    Serial.print(index, DEC);
    Serial.print(": cstate=");
    Serial.print(currentstate[index], DEC);
    Serial.print(", pstate=");
    Serial.print(previousstate[index], DEC);
    Serial.print(", press=");
    */
    
    if (currentstate[index] == previousstate[index]) {
      if ((pressed[index] == LOW) && (currentstate[index] == LOW)) {
          // just pressed
          justpressed[index] = 1;
      }
      else if ((pressed[index] == HIGH) && (currentstate[index] == HIGH)) {
          // just released
          justreleased[index] = 1;
      }
      pressed[index] = !currentstate[index];  // remember, digital HIGH means NOT pressed
    }
    //Serial.println(pressed[index], DEC);
    previousstate[index] = currentstate[index];   // keep a running tally of the buttons
  }
}


void loop() {
  byte i;
//what to do if button is pressed. Button 5-8 are [0]-[3] here
  if (justpressed[0]) {
    justpressed[0] = 0;
    playfile("ToesW.WAV");
    while (wave.isplaying && pressed[0]) {
      //Serial.print(".");
    }
    wave.stop();    
  }
  if (justpressed[1]) {
    justpressed[1] = 0;
    playfile("ShouldersW.WAV");
    while (wave.isplaying && pressed[1]) {
      //Serial.print(".");
    }
    wave.stop();    
  }
  if (justpressed[2]) {
    justpressed[2] = 0;
    playfile("HeadW.WAV");
    while (wave.isplaying && pressed[2]) {
      //Serial.print(".");
    }
    wave.stop();    
  }
  if (justpressed[3]) {
    justpressed[3] = 0;
    playfile("kneeW.WAV");
    while (wave.isplaying && pressed[3]) {
      //Serial.print(".");
    }
    wave.stop();    
  }
}



// Plays a full file from beginning to end with no pause.
void playcomplete(char *name) {
  // call our helper to find and play this name
  playfile(name);
  while (wave.isplaying) {
  // do nothing while its playing
  }
  // now it's done playing
}

void playfile(char *name) {
  // see if the wave object is currently doing something
  if (wave.isplaying) {// already playing something, so stop it!
    wave.stop(); // stop it
  }
  // look in the root directory and open the file
  if (!f.open(root, name)) {
    putstring("Couldn't open file "); Serial.print(name); return;
  }
  // OK read the file and turn it into a wave object
  if (!wave.create(f)) {
    putstring_nl("Not a valid WAV"); return;
  }
  
  // ok time to play! start playback
  wave.play();
}







