# Methodology

Replace this template with your own. Keep it short: write what a teammate would need to run your work and trust it.

## 1. How to run it
Check in the Powershell if the version is higher than Python 3.10
1. Create a virtual environment:
   python -m venv .venv
2. Go in to the menu of the project where you download the code from GitLab
    cd C:\Users\Lenovo\Documents\embedded-take-home (For example)
3. Activate it in the PowerShell:
   .\.venv\Scripts\Activate.ps1
4. Install dependencies:
   pip install -r requirements.txt
5. Go into the terminal in VScode and activate the environment
    .\.venv\Scripts\Activate.ps1
6. Run the program by putting the code below in the same Powershell:
   python sim/launch.py --visualize

## 2. Thought process
1. I match the qos of the wheel publisher and the gps publisher to the subscriber and created them. I used a print at the last callback function to check the messages.
2. Then, start to extract information from the message: first deal with initializing process. If nothing is the value is None, just gain the message and immediately jump to the next one to avoid only one-data comparison. Deal with the duplication information based on whether the delta values are 0, if so jump to the next message. Renew the values to get ready for the next round.
3. Use the information from the wheel to calculate the distance and velocity.
4. Same for testing the information from the GPS. After setting up the weight of GPS value, calibrate the position values using it. Also calculate the heading using GPS.
5. Create the publisher and publish the information required. First, set up the message then publish will be a good idea.


## 3. Known limitations
1. The headings are mainly dependent on GPS data, but its covariance may lead to inaccuracies.
2. I personally think the timing system may be a little mixed up
3. The weight of GPS in the calibration is not done in a really mathematical way so might not be precise
4. The initial headings is acquired only after 2 sets of GPS values, which is 2 seconds late, so the initial direction may seem to be wrong and lead to less accuracy.
5. I only publish the minimum required Odometry values and didn't publish in a TF2 transform.
