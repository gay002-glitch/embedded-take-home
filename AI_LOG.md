# AI Usage Log

Worthwhile to claim here that I wrote this AI log mainly using chat-GPT, in the same chat that we went through all the things. I was extremely tired to think of anything so I really need it to help me do this. I actually left some notes about AI and my process within the code, but that is not all. So the prompt that I provided chat-GPT to write this is to let it objectively assess what I did and what it did.
I have to confess that as the first time to get into such unfamiliar project I spent tons of effort just to understand the basics and without AI I can hardly move forward. Still, I wrote both call-back parts myself and AI helped me to deal with some basic logical problems to avoid zero-division and some other defects.


## 1. Understanding the ROS-style publisher/subscriber structure and QoS

**What I asked:**

I asked ChatGPT to explain how the provided ROS-style scaffold works, including Nodes, topics, publishers, subscribers, callbacks, QoS, and how a WheelTicks message reaches my `wheel_tick_callback`. Basically it was AI that helped me to go through and fully understand the whole process, I was extremely confused after reading README.

**What I kept vs. rewrote, and why:**

I used the explanation to understand the structure, but I checked the actual publisher source before choosing the QoS settings. I used `BEST_EFFORT` because the provided encoder and GPS publishers use BEST_EFFORT, instead of blindly using a default QoS profile.

I wrote the subscriber setup by imitating encoder_publisher code

**What the AI got wrong that I had to catch:**

There was no major code error in this part, but I learned not to rely only on an AI explanation for QoS because a wrong reliability setting would silently prevent the subscriber from receiving messages.

**How I verified it ran correctly (not just that it compiled):**

I ran the simulator with:

`python sim/launch.py --no-faults` and this is what AI told me to put in the terminal

and temporarily printed incoming wheel and GPS messages.(there are notes of print(...) inside the .py) I confirmed that wheel messages arrived at approximately 50 Hz and GPS messages at approximately 1 Hz.

## 2. Implementing wheel odometry and handling the first/duplicate messages

**What I asked:**

I asked ChatGPT how to turn the cumulative wheel tick count into incremental distance, and how to handle the first message and duplicate messages without causing a divide-by-zero error.

**What I kept vs. rewrote, and why:**

I kept the idea of storing the first `tick_count` and `timestamp` and returning immediately because there is no previous message available yet for computing a delta.

I also kept the idea of computing:

`delta_tick_count = msg.tick_count - self.last_tick_count`

and

`delta_tick_time = msg.timestamp - self.last_tick_time`

before converting the tick difference into distance using `DIST_PER_TICK`.

I wrote the position update myself using:

`x += distance * cos(heading)`

`y += distance * sin(heading)`

after understanding that wheel ticks only provide traveled distance and not direction.
I wrote the original program all by myself after understanding the general idea, and AI helped me to deal with many zero-division or other logical defects.

**What the AI got wrong that I had to catch:**

Some of the early discussion treated the duplicate-message test too loosely. I had to think about the fact that an exact duplicate has both the same tick count and the same timestamp, and that the important safety condition is preventing a zero or invalid time interval from reaching the velocity division.

I also learned that dropped ticks cannot be individually detected from the cumulative counter, so they have to be corrected indirectly through GPS fusion instead of trying to detect each missing tick.

**How I verified it ran correctly (not just that it compiled):**

I first ran with `--no-faults` and checked that the position moved continuously. Then I ran with faults enabled and verified that duplicate messages did not crash the program or produce infinite/NaN velocity values. 

NEVERTHELESS, I think that there is actually a mistake because no-fault setting will still give duplicated messages!!!

## 3. GPS fusion and heading estimation

**What I asked:**

I asked ChatGPT how to combine the wheel-derived position with the noisy GPS position, how to use GPS covariance, and how to estimate heading when there is no heading sensor provided by the simulator.

**What I kept vs. rewrote, and why:**

I kept the idea of estimating heading from the direction between GPS positions using `atan2(dy, dx)`.
AI only provided many mathematical ways, but the actual work that I did in stretch B is my original thought.

**What the AI got wrong that I had to catch:**

The AI initially suggested several different ways of defining the wheel/GPS weight, and some were not dimensionally or conceptually consistent. I had to simplify the approach and decide on one heuristic rather than treating the AI's first formula as authoritative.

The AI also pointed out later that heading should ideally be calculated from raw GPS position differences rather than from already fused positions, because using fused coordinates can mix the heading estimate with the fusion correction itself.

**How I verified it ran correctly (not just that it compiled):**

I used the simulator visualizer and watched the fused trajectory over time. I compared its behavior with the GPS points and checked that the estimate followed the rover path instead of drifting straight away when wheel odometry accumulated error.

In fact, I have no way to mathematically prove I am right, as this method comes out of my own math knowledge.

## 4. Constructing and publishing the Odometry message

**What I asked:**

I asked ChatGPT how the nested `nav_msgs/Odometry` message structure works and whether I could pass all of the position values directly into the `Odometry(...)` constructor.

**What I kept vs. rewrote, and why:**

My original idea was to construct the whole Odometry message in one call. After checking the actual dataclass definition and discussing it with ChatGPT, I learned that the top-level Odometry object contains nested objects such as `header`, `pose`, and `twist`.

I therefore kept the AI-assisted approach of first creating:

`msg = Odometry()`

and then assigning fields individually, including:

`msg.header.stamp`

`msg.header.frame_id`

`msg.child_frame_id`

`msg.pose.pose.position.x`

`msg.pose.pose.position.y`

I wrote the final `self.odom_pub.publish(msg)` call myself.

**What the AI got wrong that I had to catch:**

The main issue was not a wrong final answer, but that my first interpretation of the docstring treated the listed nested fields as if they were direct constructor arguments. I had to inspect the actual Odometry dataclass to confirm the object layout instead of relying only on the field list.

**How I verified it ran correctly (not just that it compiled):**

I ran:

`python sim/launch.py --no-faults --visualize`

and confirmed that the simulator no longer printed the warning that nothing had been published on `/odom`. The visualization also showed the odometry trajectory updating.

## 5. Debugging the monitoring output and receive-rate calculation

**What I asked:**

I asked ChatGPT how to implement the required monitoring output: time since the latest wheel message, time since the latest GPS message, wheel receive rate, and current fused position.

**What I kept vs. rewrote, and why:**

I kept the idea of recording `time.monotonic()` when each message arrives, because the monitoring task is measuring local elapsed time rather than sensor timestamps.

I also changed the receive-rate calculation to:

`wheel_msg_count / elapsed_time`

instead of taking the reciprocal of the age of the most recent message.

**What the AI got wrong that I had to catch:**

During debugging, I initially mixed the message timestamp with `time.monotonic()`, which produced a huge negative GPS age.

I also initially calculated wheel receive rate as:

`1 / (time.monotonic() - last_wheel_time)`

This was wrong because it measured the reciprocal of the time since the latest message rather than the average receive rate. It also occasionally caused a `ZeroDivisionError`.

After seeing the runtime traceback and incorrect values, I used AI to help identify both problems and rewrote the calculation.

**How I verified it ran correctly (not just that it compiled):**

I reran the simulator and checked the actual terminal output. The GPS age became a small positive value instead of a very large negative number, the wheel rate stayed near the expected ~50 Hz instead of jumping between values such as ~32 Hz and ~67 Hz, and the timer callback stopped raising `ZeroDivisionError`.