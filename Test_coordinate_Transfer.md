# Test_coordinate_Transfer

You will help me define a JavaScript programming project to be run in Claude Code by another agent.  The JavaScript program will have two stages.  Stage 1 will create several coordinate transformations based on a limited data set.  Stage 2 will apply the coordinate transformations to a larger dataset.  The data being transformed is from a machine test.

Take a look at C:\Users\paulp\Documents\Claude_Projects\Test_coordinate_Transfer\Run_Example_1.csv.  From the line that has -- Center find UTSDATA – in column A to the line that has --------- in all columns is the data we will use in Stage 1 to make the coordinate transfer matrices.  The data after the line that has --------- in all columns is the data that will be transformed in Stage 2.

The following coordinate systems are defined:

1.      Global with axies East, North, and Elevation and an arbitrary origin location

2.      Machine with axies X, Y, Z with its origin at the center of the machine at local ground level, the X direction facing the front of the machine, the Y direction facing the top of the machine, and Z defined by right hand rule.  The machine is on an incline with unknown orientation relative to the global coordinate system.

3.      MachineG with axies XG, YG, ZG  with its origin at the center of the machine at local ground level, YG parallel to Global Elevation, XG pointing toward the front of the machine and ZG defined by right hand rule. 

4.      MachineP which is a cylindrical coordinate system with its origin at the center of the machine at local ground level, YP equal to machine Y, RP being the distance from the origin in the XZ plane, and AP being the angle from the machine X direction in degrees with positive  values going to the left of the +X direction

5.      MachinePG which is a cylindrical coordinate system with its origin at the center of the machine at local ground level, YPG equal to MachineG YP, RPG being the distance from the origin in the XGZG plane, and APG being the angle from the machine XG direction in degrees with positive  values going to the left of the +XG direction.

The device under test is a stationary machine that has a linkage system on it.  The tip of the linkage is called Point J Prime. 

In Stage 1 Point J prime is being measured by two systems in two different coordinate systems.  It is first being measured in a global coordinate system by a UTS survey tool in terms of East, North, and Elevation.  Second, the same point is being measured in the machine coordinate system in terms of X, Y, and Z.  Nine sets of measurement data were taken as the linkage rotating Point J Prime in a circle in the XZ plane centered on the Y axis.  There is some uncertainty in the data from both measurement systems.  Create a coordinate transformation matrix from the global coordinate system to the machine coordinate system with a least squares best fit to the Stage 1 data.  Once that is done, create the transformation matrices to go from the machine coordinate system to the MahcineG, MachineP, and MachinePG coordinate systems.

In Stage 2 there is again UTS data for Point J prime as well as a variable number of points measured in the machine coordinate system.  Use the Global to Machine coordinate system to translate the UTS data and label it as UTS-J.  Then translate all the points from the machine coordinate system to the MahcineG, MachineP, and MachinePG coordinate systems.

Program output will be a CSV file with the same name as the input file appended with the word OUTPUT.  It will have a Stage 1 section that shows all the translation matricies. That will be followed by a Stage 2 section that will show the translation ofall the points to all the coordinate systems.
