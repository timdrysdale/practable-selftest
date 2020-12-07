# practable-selftest

Self tests for practable hardware using JSON messaging

[![Project Status: Concept – Minimal or no implementation has been done yet, or the repository is only intended to be a limited example, demo, or proof-of-concept.](https://www.repostatus.org/badges/latest/concept.svg)](https://www.repostatus.org/#concept)


### Initial Tasks

Just bringing these up to the top so that they don't get lost....

- define an interface for a self test function in python (argument is test address, return is JSON report)
- define test results format (ensure it can be marshalled by golang, e.g. have an array of test-result units)
- capture some data from an experiment and develop a time-series extraction routine (pass it a tree describing which variables to collect, typically you want the time, and any other values which are relvant)
- develop some tests that can use the time-series data as input
- look at asyncio and websocket client in python to see how that all works
- connect to a live experiment and start sending commands, collecting data, and implementing simple tests on the data you get back e.g. check motor stopped, start it, check motor spinning, check speed is within certain bound. Stop motor. Check motor stopped ... and so on ..


## Overview

When developing hardware, it is useful to have automated tests so you can check for repeatable operation, and that old bugs have not re-entered your firmware (regression tests). Once you are using the hardware in the field, it is also helpful for identifying when a machine requires maintenance. In remote laboratories, a typical use case is to check hardware before giving it to a student, either before every session or at some other pre-determined interval (e.g. daily). Simple checks that speeds, durations, amplitudes, frequencies etc fall within empirically determined bounds are often sufficient to identify gross failures such as worn out parts, disconnected sensors, damaged cables, and broken power supplies. Artificial intelligence for predictive maintenance is out of scope at present, but is attractive for the future.

### Scope

The self test of an experiment requires all relevant elements to be evaluated, however the data/control domain requires different approaches to the video domain, and are usually on different communications links, so this repo will potentially consider all three aspects but separately. Note that as protocols are added, new self tests will be required (e.g. different video encoders and so on).

  - Data/Control (current focus)
  - Video (on the roadmap) 
  - Audio (on the roadmap)
  
## Data/Control

Assuming that software-only/emulated tests are carried out in separate piece of work, this repo concentrates on what to do when you have the hardware present, via a websocket connection and a JSON-based API. We are mainly concerned with whether it is doing the job it is supposed to. Part of the work in developing a fleet of experiments is identifying normal operating parameters (key performance idicators, KPI), and the variability in these across the fleet. 
    
### Reliability

Tests which give false negatives are soon ignored, because maintenance call-outs find nothing they can action. Intermittent failures can occur if the mean value of a KPI for a particular piece of hardware is toward the edge of the expected distribution, such that there is a probability that during normal operation a test may return a failing KPI that is actually indicative of normal operation.  Therefore, it is necessary to tune each test to each particular piece of hardware, so as to account for manufacturing tolerances. This is particularly important with DIY assembled hardware, or hardware that is intended to be assembled from materials with high thermal expansion coefficients, and/or manufactured to a price, trading cost for variability in performance. The tests may be adjusting after maintenance. Two approaches emerge

  - implement test on the hardware itself, and call it via a single command to run the self test. Allow remote updating of the test (may be convenience and security implications)
  
    - better compartmentalisation
	- less standardisation 
	- harder to update tests (requires individually rolling out tests)
	- test parameters require storing on the machine reducing visibility 
	- reduced visibility of individual test details once machine is in the field (unless full report is available to user)
	- require to trust the test provider has written tests which suit your purposes (some activities may need tighter tolerances for example)
	- requires a decision on whether any user can trigger a self test
	- requires a way to secure commands which alter the test parameters such as acceptable KPI ranges
	- requires the hardware to understand how to report the test results, either in summary or in detail
	- requires the management system to keep track of report formats and versions across the fleet (especially if aggregate pool from multiple suppliers/builders)
	   
  - implement the test on a remote server, which has a database of expected parameters, and makes standard API calls to the machine as if it were a user.
    - tests can be updated in a single place
    - pool owners can potentially select different tests depending on their needs (see below)
	- gives comfort that the API is working as expected, not just the hardware (particularly relevant if supporting multiple API versions)
	- requires a database of test parameters to be maintained
    - good visibility of fleet performance over time
	- no particular security required
	- limited possibility to speed up tests where timeouts cannot be changed by user
	- requires elevated access if a non-user feature is needed (e.g. timeout reduction)
	- how do you know which experiment you are testing, when the booking system may not have explicitly revealed this (require experiments to report a gen4 UUID?)
	- de-embed the effect of network latency on timings (require experiments to implement a ping test?)
	- de-embed the effect of quality-of-service vs traffic type / bandwidth (require experiments to implement something like a broadband speed test?)
		  
  
### Assessing suitability for a given activity

Not everyone has the same goals or requirements for a piece of hardware - for example, an activity exploring variability may desire different performances from experiments in the fleet, while another activity may want students to focus on a particular concept and build up measurements over a number of sessions, which may be affected if there is too much variability from session to session. Hence, the idea that there is a single test for each piece of equipment cannot hold true for all future uses of an experiment, because it is unlikely that all future experimental uses (and hence KPI) can be envisaged. It is expected that activity developers will write their own automated tests to assess the suitability of a given pool for their needs, either prior to, or during writing the user interface and any course documentation.

### Implementation 

For centralised tests, it is desirable that they can be run by agents which can communicate with the rest of the system, including authorisation servers. The rest of the system is being implemented in golang, which suggests the following options

  - python-then-golang
    - initial development of test using python, connecting to unsecured local hardware
	- transition "mature" test to golang, using pre-existing methods for authenticating to secured experiments
  - python-plus-golang
	- initial development of test using python, connecting to unsecured local hardware
	- package "mature" test for deployment in python, using golang test-intermediary to handle authentication

It is likely that the python-then-golang approach is only useful for developing the basic test against a single prototype. When multiple copies of an experiment are produced, they are likely to go straight into being accessible only via a secure mechanism, indicating that any interative development of the tests to take into account fleet behaviour, will need to use the python-plus-golang approach.

Therefore, it makes sense to assume for now that the python code simply needs to be able to connect to an unsecured websocket, using either ws:// or wss://, which will either be the hardware, or the golang intermediary.


### Features

A guiding principle in deciding whether a piece of equipment is working well enough or not, is whether the data provided to the user is of an appropriate quality for the activity they are doing. If an activity is about troubleshooting, then failing sensor data is acceptable. If it is an introductory activity on the sensor, then data from a broken or failing sensor is not appropriate because the student not yet far enough along with their study to distinguish working sensor from non-working sensor.

Key implementation note - python is already a scripting language, so write each experiment's test in python as a separate function, and do not attempt to write a macro language for testing (to provide all the features you end up needing, you'd be duplicating python so why bother).

Bearing in mind the above points,we have this non-exhaustive feature list 

 - runs on server
 - checks hardware is ok to send to user 
 - connect to ws:// or wss://
 - reconnect if disconnected, but using backoff (play nice)
 - send and receive JSON 
 - cope with malformed JSON
 - evaluate ping time
 - evaluate bandwidth available (speed test)
 - detect events by looking for messages
 - detect events by analysing the data
 - send commands at pre-determined times
 - send commands in response to events
 - extract data from multiple messages to form time-series
 - ignore non-data messages when assembling time-series
 - time-out rather than hang, if event not forthcoming
 - mixed single point and arrays for the same variable due to adjustable data rates
 - cope with messages of different format, and of dynamic format
 - ensure safety (how to handle the case where the system becomes unresponsive whilst active)
 - selectable test duration (e.g optional testing of slow things like long timeouts)
 - return summary and detailed test results in a standard JSON format

 
 data types for a single variable due to variable sample rates [, such single data points, other times arrays
 - handle system


#### Server system features

We'll eventually need these capabilities

 - report individual test results in JSON message (to admin channel - to be described)
 - report overall summary in JSON message (to admin channel - to be described)
 - receive commands on what to check (from admin channel - to be described)
 - produce human-readable templated report for each experiment self test
 - produce human-readble templated report for historical series of self tests on a given experiment
 - produce human readable templated report

They can be implemented outside of the actual self test function, and work off of the test result returned.

 
### Initial Tasks

- define an interface for a self test function 
- define test results format (ensure it can be marshalled by golang, e.g. have an array of test-result units)
- capture some data from an experiment and develop a time-series extraction routine (pass it a tree describing which variables to collect, typically you want the time, and any other values which are relvant)
- look at asyncio and websocket client in python to see how that all works
- connect to a live experiment and start sending commands, collecting data, and implementing simple tests on the data you get back e.g. check motor stopped, start it, check motor spinning, check speed is within certain bound. Stop motor. Check motor stopped ... and so on ..


### Report format

Golang is statically typed, which means that JSON is marshalled into statically typed structs before it can be interpreted.  If nested objects will have dynamically assigned names, then it is easier to process them if the dynamically assigned name is a value, rather than a key. For example, this is more DIFFICULT to handle because the fruit names are keys....

```
{
  "fruits": {
    "banana": {
      "colour": "yellow",
      "sizeCm": 12
    },
    "apple": {
      "colour": "red",
      "sizeCm": 9
    }
  }
}
```
It instead is far better represented as an array of objects with a named field which holds the key (i.e. make the identification an explicit key-value pair, rather than implicit as per above) - then we can define a fruit type and marshall each item in the array into an array of fruits (or map them based on the value of the ```what``` field).


```
{
  "fruits": [
    {
      "what": "banana",
      "colour": "yellow",
      "sizeCm": 12
    },
    {
      "what": "apple",
      "colour": "red",
      "sizeCm": 9
    }
  ]
}
```

Therefore, test results need defining like this, with some thought given to the different test types, and how to define them, so that you can run whatever combination of tests you want, AND still expressively report the values. So test types could be ...

- binary
- compare value (greater / lower)
- within range (min, max, typ)
- produce std-dev (give mean, std deviation expected, and mean, std dev achieved)
- array (produce results arrays, for plotting)
- etc

These need some thinking about though (e.g. is the std deviation one just a range test that happens to use statistics before coming up with a number to compare to a range - probably, yes, so delete it)

### Example tests

Prep- stop the motor, calibrate if needed
Is the disk stopped when the motor is off?
Does the disk spin at the right speed +/- some tolerance when I set the speed to be 25%
Does the disk spin at the right speed +/- some tolerance when I set the speed to be 50%
Does the disk spin at the right speed +/- some tolerance when I set the speed to be 75%
Does the disk spin at the right speed +/- some tolerance when I set the speed to be 100% (of safe continuous operating speed)
Does the disk spin down from 100% to zero, when I turn the motor off, in the right amount of time, plus or minus some tolerance?
Post - put experiment into resting state - motor off.

## Roadmap items

Some initial thoughts parked here for now

### Video testing

Video testing is relatively straightforward conceptually - a good first test is to check the framerate, or data rate because no frames/data means no video! 

  - Video quality
  
  Secondary refinements include checking for an image that conforms to expectations (e.g. not all-black, not all-white, in focus, in colour if colour expected, latency within bounds, minimal artefacts/corruption). 

- Latency
 
    Latency checking may require movement in the scene, which may not happen quickly for some experiments, e.g. with slow moving apparatus, and is likely only to matter for fast moving experiments only  therefore these secondary refinements are optional, and perhaps to be applied only where they add value. 


 - Video bombing
 
   Malicious behaviour is always a risk with any system. We are intending to reduce the chance of that as close as possible to zero by:
   
   - securing the system using standards-based approach (currently intending OAuth2 via keycloak)
   - verified experiment hosts only in "trusted" tier (anyone can host in community tier)
   - one-way video enforced at server - participants cannot feed video to group members accessing the same experiment
   - privacy protection by either 
     - not showing uncontrolled/contextual backgrounds that could have people in them
	 - physical blurring of the image (frosted glass) 
	 - software blurring (so that the context of the building location can be seen, but any faces passing by are individually blurred for privacy)
   - potential moderation of video content in community tier to avoid the system being abused. 
   - We do not offer end-to-end encryption (our servers decrypt and re-encrypt) so we can potentially moderate content to prevent systematic abuse e.g. non-experiment-based video sharing

   
### Audio Testing

This section contains aide-memoires for future expansion.
   
Proposal - consider using voice filters to mask any overhead conversation if microphones are alive on the experiment and people are nearby
   
Testing - cause a noise and measure its amplitude, duration and frequency.
   
   




