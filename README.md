## Goal
In this project the goal is to safely navigate around a virtual highway with other traffic that is driving +-10 MPH of the 50 MPH speed limit. The car's localization and sensor fusion data are provided, there is also a sparse map list of waypoints around the highway. The car should try to go as close as possible to the 50 MPH speed limit, which means passing slower traffic when possible, note that other cars will try to change lanes too. The car should avoid hitting other cars at all cost as well as driving inside of the marked road lanes at all times, unless going from one lane to another. The car should be able to make one complete loop around the 6946m highway. Since the car is trying to go 50 MPH, it should take a little over 5 minutes to complete 1 loop. Also the car should not experience total acceleration over 10 m/s^2 and jerk that is greater than 10 m/s^3.

## Solution Approach
Here we go through the main steps that we need to successfully get the car to drive around the track without colliding with other vehicles or violating any rules:

### 1- Lane Keeping:
We start off by trying to keep the car moving in it's lane, this is achieved by using Frenet coordinates, where the D-coorindate stays constant, corresponding to the center of the desired lane (2 for the leftmost lane, 6 for the middle lane, 10 for the righmost lane), and the S-coordinate increases by a specific distance increment with each cycle, then we convert from Frenet to X-Y coordinates, using the given helper function `getXY()`, giving us x & y values that we can feed to the `next_x_vals` & `next_y_vals` vectors which in return feeds the values to the simulator.

### 2- Smooth Trajectory Generation (Limiting Lateral Acceleration & Jerk):
This works well to keep the car in it's lane, but it creates a trajectory with sharp corners, which leads to a high lateral acceleration and jerk at these corner points. So in order to create a smooth trajectory, we use the spline library, to fit a spline to a vector of widely spaced anchor points (that starts with the vehicle position, or future position) along our desired lane, which creates a smooth spline, and then we sample this spline to get the actual x & y values, that we then feed to the simulator in a similar fashion as above.

It is worth noting that before fitting the spline we convert all the anchor points to vehicle coorindates in order to keep the spline math simple, and to make it behave as expected, and then we convert all the sampled points from the spline back to the global coordinates.

In order to keep the path smooth even while changing lanes, each newly created path has to start from where the previous path ended, which requires us to poll the simulator each time we create a new path, and add whatever points not yet processed from the previous path, to the new path, and start adding new points after that.

### 3- Collision Avoidance:
In order to avoid hitting vehciles in front of us, we loop through the 'sensor_fusion' vector and check all the detected vehicles which are in the same lane and in front of us, then we read their velocities and S-Coorinates. Using it's velocity we can predict the detected vehicle future position (it's S-Coordinate when the ego vehicle will be at the end of it's path), and compare it to the future S-Cooridinate of the ego vehicle, that way we don't have to wait until the ego vehicle is actually too close to the vehicle in front to take action.

In case a `too_close` flag is detected, a left lane change is immediately attempted, if the attempt fails, we gradually reduce the the ego vehicle speed, till the flag is reset.

### 4- Limiting Longitudinal Acceleration & Jerk:
To avoid high acceleration at the start of the simulation, we start by setting the velocity to zero, and gradually increasing the velocity at each cycle by 0.224 mph (0.1 m/sec) (divided by 0.02 sec for each cycle gives 5 m/s<sup>2</sup> which is our max acceleration) until we reach the maximum speed of (49.50 mph).

### 5- Lane Changing:
When a lane change is attempted, we have to check for nearby cars in the target lane, in a similar fashion as we checked for the front vehicles in our lane, but the difference here is that we check for all the cars whether they are in front of us or backwards, and we make sure that there is a safe distance between us and any of them, hence the check for absolute difference `(abs(check_car_s - car_s) < 10)`.

After each lane change we set the counter `lane_change_counter` to a specific (delay) value, to prevent the car from immediately changing lanes again, which might lead the car to hesitate between 2 lanes and ultimately drive on the lane line itself.

#### 5.1- Left Lane Changes:
Left lane changes are attempted whenever there is a slow car in front of us.

#### 5.2- Right Lane Changes:
Right lane changes are attempted whenever there is a slow car in front of us and left lane attempts fail repeatedly for a set counter value `right_lane_attempt_counter`.

Whenever a lane change is successful the `right_lane_attempt_counter` is reset, to prevent any subsequent unnecessary and potentially harmful right lane changes, making the right lane change an undesirable measure that we only take when all else fails.
