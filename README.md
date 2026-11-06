
 
1. Aim
To design, code and simulate a traffic light controller for a two road junction using Verilog HDL. The project is done in three parts:
•	Part A: a timer based controller that changes the lights after fixed delays.
•	Part B: a sensor based controller that changes the lights only when a vehicle is detected on the side road.
•	Part C (extension): the timer based controller with a pedestrian push button and a WALK phase.
2. Tools Used
•	Verilog HDL (IEEE 1364) for the design and the testbenches.
•	Any Verilog simulator can run the files: ModelSim, Xilinx Vivado, Icarus Verilog with GTKWave, or EDA Playground.
•	Each design has its own testbench which also writes a .vcd file for viewing the waveform.
3. Introduction
A traffic light controller is a sequential circuit. It must remember which light is on right now (the present state) and decide what to show next. This behaviour is described by a Finite State Machine (FSM). In this project the FSM is a Moore machine, which means the lights depend only on the present state and not directly on the inputs.
Every design has the same three parts:
•	State register: flip-flops that hold the present state and change on the clock edge.
•	Next-state logic: combinational logic (or a counter) that decides the next state.
•	Output logic: combinational logic that turns the state into signals for the red, yellow and green lamps.
Road 1 is the main road and Road 2 is the side road. A red light is always kept on one road while the other road is green or yellow, so the two roads never get green together.
 
4. Part A: Timer Based Traffic Light Controller
4.1 Working
The controller has six states, S0 to S5. A 4-bit counter measures how long the controller stays in each state. Long states (S0 and S3) wait until the counter reaches SEC5 = 15. Short states (S1, S2, S4, S5) wait until the counter reaches SEC1 = 3. When the limit is reached the counter is cleared and the next state starts. A high clr input resets the controller to S0 immediately (asynchronous reset).
The two all-red states S2 and S5 give vehicles time to clear the junction before the other road gets its green light.
4.2 Output encoding
The output lights[5:0] is arranged as {R1, Y1, G1, R2, Y2, G2}. A bit value of 1 means that lamp is ON.
State	Road 1	Road 2	lights[5:0]	Stays for
S0	Red	Green	100001	count < 15, so 16 clock cycles
S1	Red	Yellow	100010	count < 3, so 4 clock cycles
S2	Red	Red	100100	count < 3, so 4 clock cycles
S3	Green	Red	001100	count < 15, so 16 clock cycles
S4	Yellow	Red	010100	count < 3, so 4 clock cycles
S5	Red	Red	100100	count < 3, so 4 clock cycles
Table 1: States and light outputs of the timer based controller.
The counter starts from 0, so a limit of 15 gives 16 clock cycles and a limit of 3 gives 4 clock cycles. One full cycle is 16 + 4 + 4 + 16 + 4 + 4 = 48 clock cycles. The names SEC5 and SEC1 mean a long delay and a short delay (about 4 to 1). If the clock is slowed down to a few hertz, these become real seconds.
4.3 State diagram
 
Fig. 1: State diagram of the timer based controller (top half of each circle is Road 1, bottom half is Road 2)
4.4 Block diagram
 
Fig. 2: Block diagram of the timer based controller
The comparator checks the counter against the limit of the present state. If the counter is below the limit the state register holds its value and the counter increments. Otherwise the next-state logic loads the following state and clears the counter. The output decoder converts the 3-bit state into the six lamp signals.
4.5 Verilog code (Traffic_Light_Time.v)
`timescale 1ns / 1ps
 
module Traffic_Light_Time(
    input  wire clk,
    input  wire clr,
    output reg [5:0] lights      // {R1, Y1, G1, R2, Y2, G2}
);
 
    reg [2:0] state;
    reg [3:0] count;
 
    parameter S0 = 3'b000, S1 = 3'b001, S2 = 3'b010,   // states
              S3 = 3'b011, S4 = 3'b100, S5 = 3'b101;
 
    parameter SEC5 = 4'b1111, SEC1 = 4'b0011;          // delays
 
    // ---------- State register + counter (sequential) ----------
    always @(posedge clk or posedge clr)
    begin
        if (clr == 1)
        begin
            state <= S0;
            count <= 0;
        end
        else
            case (state)
                S0: if (count < SEC5)
                    begin
                        state <= S0;
                        count <= count + 1;
                    end
                    else
                    begin
                        state <= S1;
                        count <= 0;
                    end
 
                S1: if (count < SEC1)
                    begin
                        state <= S1;
                        count <= count + 1;
                    end
                    else
                    begin
                        state <= S2;
                        count <= 0;
                    end
 
                S2: if (count < SEC1)
                    begin
                        state <= S2;
                        count <= count + 1;
                    end
                    else
                    begin
                        state <= S3;
                        count <= 0;
                    end
 
                S3: if (count < SEC5)
                    begin
                        state <= S3;
                        count <= count + 1;
                    end
                    else
                    begin
                        state <= S4;
                        count <= 0;
                    end
 
                S4: if (count < SEC1)
                    begin
                        state <= S4;
                        count <= count + 1;
                    end
                    else
                    begin
                        state <= S5;
                        count <= 0;
                    end
 
                S5: if (count < SEC1)
                    begin
                        state <= S5;
                        count <= count + 1;
                    end
                    else
                    begin
                        state <= S0;
                        count <= 0;
                    end
 
                default: state <= S0;
            endcase
    end
 
    // ---------- Output logic (combinational, Moore type) ----------
    always @(*)
    begin
        case (state)
            S0: lights = 6'b100001;   // red    green
            S1: lights = 6'b100010;   // red    yellow
            S2: lights = 6'b100100;   // red    red
            S3: lights = 6'b001100;   // green  red
            S4: lights = 6'b010100;   // yellow red
            S5: lights = 6'b100100;   // red    red
            default: lights = 6'b100001;   // red green
        endcase
    end
 
endmodule

4.6 Explanation of the code
Code	Explanation
`timescale 1ns / 1ps	Sets the time unit to 1 ns and the precision to 1 ps. Every delay in the testbench is in ns.
module Traffic_Light_Time(...)	Starts the module. Inputs are clk and clr. The output is the 6-bit lights bus.
output reg [5:0] lights	The six lamp signals {R1,Y1,G1,R2,Y2,G2}. It is declared reg because it is assigned inside an always block.
reg [2:0] state; reg [3:0] count;	state holds the present state (3 bits are enough for 6 states). count is the 4-bit delay counter, range 0 to 15.
parameter S0 ... S5	Gives each state a binary code: S0 = 000 up to S5 = 101. Names make the code easy to read.
parameter SEC5 = 4'b1111, SEC1 = 4'b0011	The delay limits: 15 for the long phases and 3 for the short phases.
always @(posedge clk or posedge clr)	Sequential block. It runs on every rising clock edge and also immediately when clr rises (asynchronous reset).
if (clr == 1) begin state <= S0; count <= 0; end	Reset: go to S0 and clear the counter.
case (state)	Chooses what to do depending on the present state.
S0: if (count < SEC5) begin state <= S0; count <= count + 1; end	Counter is below the limit: stay in S0 and add 1 to the counter.
else begin state <= S1; count <= 0; end	Limit reached: move to the next state and restart the counter.
S1, S2, S4, S5 (limit SEC1) and S3 (limit SEC5)	Same pattern as S0. The short states use SEC1 and S3 uses SEC5. The last state S5 goes back to S0, so the cycle repeats forever.
default: state <= S0;	Safety branch. If the register ever holds an unused code (110 or 111) the controller recovers to S0.
<= (non-blocking assignment)	Used in clocked blocks so that state and count are updated together at the clock edge.
always @(*)	Combinational output block. It runs again whenever state changes.
S0: lights = 6'b100001;	Each state drives its lamp pattern. S0 means R1 and G2 are on (Road 1 red, Road 2 green).
S2 and S5: lights = 6'b100100;	Both roads red. This is the safety gap between the two green phases.
default: lights = 6'b100001;	Same as S0 for unused state codes.
endmodule	Ends the module.

4.7 Testbench (Traffic_Light_Time_tb.v)
`timescale 1ns / 1ps
 
module Traffic_Light_Time_tb;
 
    reg clk;
    reg clr;
    wire [5:0] lights;
 
    // Device under test
    Traffic_Light_Time uut (
        .clk(clk),
        .clr(clr),
        .lights(lights)
    );
 
    // Clock: period 20 ns
    always #10 clk = ~clk;
 
    initial begin
        clk = 0;
        clr = 1;          // reset the controller first
        #25 clr = 0;      // release reset, run freely
        #1500;
        $finish;
    end
 
    initial begin
        $dumpfile("traffic_time.vcd");
        $dumpvars(0, Traffic_Light_Time_tb);
        $monitor("t=%0t  clr=%b  lights=%b", $time, clr, lights);
    end
 
endmodule
The testbench resets the controller for 25 ns with clr = 1, then releases it. The clock toggles every 10 ns, so the period is 20 ns. The simulation runs for about 1500 ns, which is more than one full cycle of 960 ns.
4.8 Simulation result
 
Fig. 3: Timing diagram of the timer based controller
State	Time (ns)	lights[5:0]	Meaning
S0	0 to 330	100001	Road 1 red, Road 2 green
S1	330 to 410	100010	Road 1 red, Road 2 yellow
S2	410 to 490	100100	Both red
S3	490 to 810	001100	Road 1 green, Road 2 red
S4	810 to 890	010100	Road 1 yellow, Road 2 red
S5	890 to 970	100100	Both red
S0	970 onwards	100001	Cycle repeats
Table 2: Observed state sequence of the timer based controller.
Observation: the sequence 100001, 100010, 100100, 001100, 010100, 100100 repeats every 960 ns. The long phases are 320 ns (16 clocks) and the short phases are 80 ns (4 clocks). The first S0 is slightly longer because it includes the reset period at the start.
 
5. Part B: Sensor Based Traffic Light Controller
5.1 Working
This controller has no timer. Road 1 stays green until a vehicle is detected on Road 2, and Road 2 stays green until a vehicle is detected on Road 1. SNS1 is the sensor on Road 1 and SNS2 is the sensor on Road 2. TL1 and TL2 are the 2-bit light outputs of the two roads.
Light code: green = 00, yellow = 01, red = 10.
State	TL1 (Road 1)	TL2 (Road 2)	Next state
S0 (00)	Green (00)	Red (10)	SNS2 = 1: go to S1.  SNS2 = 0: stay in S0
S1 (01)	Yellow (01)	Red (10)	Always go to S2 (yellow lasts one clock)
S2 (10)	Red (10)	Green (00)	SNS1 = 1: go to S3.  SNS1 = 0: stay in S2
S3 (11)	Red (10)	Yellow (01)	Always go to S0 (yellow lasts one clock)
Table 3: State table of the sensor based controller.
5.2 State diagram
 
Fig. 4: State diagram of the sensor based controller
5.3 Logic equations and circuit
Using the state codes S0 = 00, S1 = 01, S2 = 10 and S3 = 11 (Q1Q0), the flip-flop inputs and the outputs can be written as Boolean equations.
Signal	Equation
D1 (next Q1)	Q1 XOR Q0
D0 (next Q0)	Q1'.Q0'.SNS2 + Q1.Q0'.SNS1
TL1[1]	Q1
TL1[0]	Q1'.Q0
TL2[1]	Q1'
TL2[0]	Q1.Q0
Table 4: Boolean equations derived from Table 3.
These equations were checked against all four states. For example, in S2 (Q1Q0 = 10): TL1 = 10 (red) and TL2 = 00 (green), which matches the table.
 
Fig. 5: Gate level circuit of the sensor based controller
5.4 Verilog code (Traffic_Light_Sensor.v)
`timescale 1ns / 1ps
 
module Traffic_Light_Sensor(
    input clk,
    input clr,
    input SNS1,
    input SNS2,
    output reg [1:0] TL1, TL2
);
 
    reg [1:0] state, nextstate;
 
    parameter S0 = 2'b00, S1 = 2'b01, S2 = 2'b10, S3 = 2'b11;
    parameter green = 2'b00, yellow = 2'b01, red = 2'b10;
 
    // ---------- State register ----------
    always @(posedge clk, posedge clr)
    begin
        if (clr) state <= S0;
        else     state <= nextstate;
    end
 
    // ---------- Next-state logic ----------
    always @(*)
    begin
        case (state)
            S0: if (SNS2) nextstate = S1;
                else      nextstate = S0;
            S1: nextstate = S2;
            S2: if (SNS1) nextstate = S3;
                else      nextstate = S2;
            S3: nextstate = S0;
            default: nextstate = S0;
        endcase
    end
 
    // ---------- Output logic ----------
    always @(*)
    begin
        if (state == S0)
        begin
            TL1 = green;
            TL2 = red;
        end
        else if (state == S1)
        begin
            TL1 = yellow;
            TL2 = red;
        end
        else if (state == S2)
        begin
            TL1 = red;
            TL2 = green;
        end
        else
        begin
            TL1 = red;
            TL2 = yellow;
        end
    end
 
endmodule

5.5 Explanation of the code
Code	Explanation
input clk, clr, SNS1, SNS2	Clock, reset and the two vehicle sensors.
output reg [1:0] TL1, TL2	2-bit light signals of the two roads.
reg [1:0] state, nextstate;	state is the present state. nextstate is decided by the combinational logic.
parameter S0 ... S3	2-bit codes of the four states.
parameter green, yellow, red	Codes for the lamp colours: 00, 01 and 10. They make the output block easy to read.
always @(posedge clk, posedge clr)	State register block. clr is asynchronous.
if (clr) state <= S0; else state <= nextstate;	On reset go to S0, otherwise copy nextstate into state on each clock edge.
always @(*) with case (state)	Next-state logic. It re-evaluates whenever state or a sensor input changes.
S0: if (SNS2) nextstate = S1; else nextstate = S0;	Stay in S0 until a vehicle is seen on Road 2.
S1: nextstate = S2;	Yellow phase: always continue to S2.
S2: if (SNS1) nextstate = S3; else nextstate = S2;	Stay in S2 until a vehicle is seen on Road 1.
S3: nextstate = S0;	Yellow phase: always return to S0.
default: nextstate = S0;	Recovery for any unused code.
always @(*) with if (state == S0) ...	Output block. Each state sets TL1 and TL2. For example S0 gives TL1 = green and TL2 = red.
else begin TL1 = red; TL2 = yellow; end	The remaining state S3 gives Road 1 red and Road 2 yellow.

5.6 Testbench (Traffic_Light_Sensor_tb.v)
`timescale 1ns / 1ps
 
module Traffic_Light_Sensor_tb;
 
    reg clk, clr, SNS1, SNS2;
    wire [1:0] TL1, TL2;
 
    Traffic_Light_Sensor uut (
        .clk(clk), .clr(clr),
        .SNS1(SNS1), .SNS2(SNS2),
        .TL1(TL1), .TL2(TL2)
    );
 
    // Clock: period 10 ns
    initial clk = 1;
    always #(10/2) clk = ~clk;
 
    initial begin
        clr = 1;
        #13;
        clr = 0;
 
        SNS1 = 0; SNS2 = 0; #(10);
        SNS1 = 0; SNS2 = 1; #(40);   // car on road 2 -> road 1 turns yellow/red
        SNS1 = 0; SNS2 = 0; #(40);
        SNS1 = 1; SNS2 = 0; #(30);   // car on road 1 -> road 2 turns yellow/red
        SNS1 = 0; SNS2 = 0; #(20);
        SNS1 = 0; SNS2 = 1; #(30);
        $finish;
    end
 
    initial begin
        $dumpfile("traffic_sensor.vcd");
        $dumpvars(0, Traffic_Light_Sensor_tb);
        $monitor("t=%0t SNS1=%b SNS2=%b TL1=%b TL2=%b", $time, SNS1, SNS2, TL1, TL2);
    end
 
endmodule
The testbench resets the design for 13 ns. Then it applies sensor patterns: no vehicle for 10 ns, a vehicle on Road 2 for 40 ns, no vehicle for 40 ns, a vehicle on Road 1 for 30 ns, no vehicle for 20 ns, and finally a vehicle on Road 2 again. The clock period is 10 ns.
5.7 Simulation result
 
Fig. 6: Timing diagram of the sensor based controller
State	Time (ns)	TL1	TL2	Reason
S0	0 to 25	00 green	10 red	Reset, no Road 2 vehicle
S1	25 to 35	01 yellow	10 red	SNS2 = 1 seen at 25 ns
S2	35 to 105	10 red	00 green	Waits for SNS1
S3	105 to 115	10 red	01 yellow	SNS1 = 1 seen at 105 ns
S0	115 to 155	00 green	10 red	Waits for SNS2
S1	155 to 165	01 yellow	10 red	SNS2 = 1 seen at 155 ns
S2	165 onwards	10 red	00 green	Waits for SNS1
Table 5: Observed behaviour of the sensor based controller.
Observation: the lights change only when the matching sensor is active. While both sensors are 0 the controller holds the present state. The yellow states S1 and S3 last exactly one clock period (10 ns) in this simulation.
 
6. Part C: Extension with Pedestrian Button
6.1 Idea
The timer based controller is extended with a pedestrian push button ped_btn and a new state S6 (WALK). A button press is stored in the register ped_pending, so a short press is not missed. At the next all-red state (S2 or S5) the controller goes to S6 instead of giving green to a road. In S6 both roads are red and the output walk = 1 for 8 clock cycles (SECP = 7). After that the request is cleared and the normal cycle continues from where it would have gone: to S3 if S6 was entered from S2, or to S0 if it was entered from S5. A 1-bit register ret remembers this.
6.2 State diagram
 
Fig. 7: State diagram with the pedestrian state S6
6.3 Verilog code (Traffic_Light_Pedestrian.v)
`timescale 1ns / 1ps
 
// Extension: timer based controller with a pedestrian push button.
// A button press is remembered (ped_pending). At the next all-red
// state (S2 or S5) the controller enters S6 (WALK) with both roads red.
module Traffic_Light_Pedestrian(
    input  wire clk,
    input  wire clr,
    input  wire ped_btn,
    output reg [5:0] lights,     // {R1, Y1, G1, R2, Y2, G2}
    output reg       walk
);
 
    reg [2:0] state;
    reg [3:0] count;
    reg       ped_pending;       // button press waiting to be served
    reg       ret;               // 0: go to S3 after WALK, 1: go to S0
 
    parameter S0 = 3'b000, S1 = 3'b001, S2 = 3'b010,
              S3 = 3'b011, S4 = 3'b100, S5 = 3'b101, S6 = 3'b110;
 
    parameter SEC5 = 4'd15, SEC1 = 4'd3, SECP = 4'd7;
 
    always @(posedge clk or posedge clr)
    begin
        if (clr)
        begin
            state <= S0; count <= 0; ped_pending <= 0; ret <= 0;
        end
        else
        begin
            if (ped_btn) ped_pending <= 1;          // remember the request
 
            case (state)
                S0: if (count < SEC5) count <= count + 1;
                    else begin state <= S1; count <= 0; end
                S1: if (count < SEC1) count <= count + 1;
                    else begin state <= S2; count <= 0; end
                S2: if (count < SEC1) count <= count + 1;
                    else begin
                        count <= 0;
                        if (ped_pending) begin state <= S6; ret <= 0; end
                        else state <= S3;
                    end
                S3: if (count < SEC5) count <= count + 1;
                    else begin state <= S4; count <= 0; end
                S4: if (count < SEC1) count <= count + 1;
                    else begin state <= S5; count <= 0; end
                S5: if (count < SEC1) count <= count + 1;
                    else begin
                        count <= 0;
                        if (ped_pending) begin state <= S6; ret <= 1; end
                        else state <= S0;
                    end
                S6: if (count < SECP) count <= count + 1;
                    else begin
                        count <= 0;
                        ped_pending <= 0;           // request served
                        state <= (ret) ? S0 : S3;
                    end
                default: begin state <= S0; count <= 0; end
            endcase
        end
    end
 
    always @(*)
    begin
        walk = 1'b0;
        case (state)
            S0: lights = 6'b100001;   // red    green
            S1: lights = 6'b100010;   // red    yellow
            S2: lights = 6'b100100;   // red    red
            S3: lights = 6'b001100;   // green  red
            S4: lights = 6'b010100;   // yellow red
            S5: lights = 6'b100100;   // red    red
            S6: begin lights = 6'b100100; walk = 1'b1; end  // WALK
            default: lights = 6'b100001;
        endcase
    end
 
endmodule

6.4 Explanation of the new parts
Code	Explanation
input ped_btn; output reg walk;	New push button input and the WALK lamp output.
reg ped_pending; reg ret;	ped_pending stores a button press. ret chooses the state after WALK (0 = S3, 1 = S0).
parameter S6 = 3'b110; SECP = 4'd7	Code of the WALK state and its duration (8 clock cycles).
if (ped_btn) ped_pending <= 1;	Remembers the request even if the button is released quickly.
S2: ... if (ped_pending) begin state <= S6; ret <= 0; end	At the end of the all-red state S2, serve the request if one is waiting, else go on to S3.
S5: ... if (ped_pending) begin state <= S6; ret <= 1; end	Same at the end of S5, but return to S0 afterwards.
S6: ... ped_pending <= 0; state <= (ret) ? S0 : S3;	After the WALK time the request is cleared and the normal sequence resumes.
S6: begin lights = 6'b100100; walk = 1'b1; end	Both roads red and walk lamp on.

6.5 Testbench (Traffic_Light_Pedestrian_tb.v)
`timescale 1ns / 1ps
 
module Traffic_Light_Pedestrian_tb;
 
    reg clk, clr, ped_btn;
    wire [5:0] lights;
    wire walk;
 
    Traffic_Light_Pedestrian uut (
        .clk(clk), .clr(clr), .ped_btn(ped_btn),
        .lights(lights), .walk(walk)
    );
 
    always #10 clk = ~clk;          // period 20 ns
 
    initial begin
        clk = 0; clr = 1; ped_btn = 0;
        #25 clr = 0;
        #150  ped_btn = 1;          // press while road 2 is green (S0)
        #20   ped_btn = 0;
        #1000 ped_btn = 1;          // second press later in the cycle
        #20   ped_btn = 0;
        #1000 $finish;
    end
 
    initial begin
        $dumpfile("traffic_ped.vcd");
        $dumpvars(0, Traffic_Light_Pedestrian_tb);
        $monitor("t=%0t btn=%b lights=%b walk=%b", $time, ped_btn, lights, walk);
    end
 
endmodule
6.6 Simulation result
 
Fig. 8: Timing diagram of the pedestrian controller
The button is pressed at 175 ns while Road 2 is green (S0). The controller finishes S0, S1 and S2 as normal. At 490 ns it enters S6 and walk stays 1 until 650 ns. It then continues with S3. A second press at 1195 ns is served in the same way at 1610 ns.
7. Comparison of the Designs
Feature	Timer based	Sensor based	With pedestrian
Number of states	6	4	7
State register	3 bits	2 bits	3 bits
Delay counter	4-bit counter	Not needed	4-bit counter
Inputs	clk, clr	clk, clr, SNS1, SNS2	clk, clr, ped_btn
Outputs	lights[5:0]	TL1[1:0], TL2[1:0]	lights[5:0], walk
Light change decided by	Fixed delays	Vehicle sensors	Fixed delays + button
Table 6: Comparison of the three designs.
8. Result
The timer based, sensor based and pedestrian controllers were written in Verilog and verified with testbenches. All three produced the expected light sequences, and the two roads never showed green at the same time.
9. Conclusion
A traffic light controller is a good example of a Moore FSM. The timer based design is simple and predictable but wastes time when one road is empty. The sensor based design adapts to traffic and needs fewer states. The pedestrian extension shows how a new state and a stored request can be added without changing the rest of the design. Using parameters for the states and delays makes the code easy to modify.
10. Applications and Future Scope
•	Traffic signals at road junctions, pedestrian crossings and railway crossings.
•	The design can be implemented on an FPGA board by slowing the clock to 1 Hz with a clock divider and driving LEDs.
•	Future improvements: emergency vehicle priority, a night mode with a blinking yellow, and a four road junction.
11. Files Submitted
File	Description
Traffic_Light_Time.v	Timer based controller
Traffic_Light_Time_tb.v	Testbench for the timer based controller
Traffic_Light_Sensor.v	Sensor based controller
Traffic_Light_Sensor_tb.v	Testbench for the sensor based controller
Traffic_Light_Pedestrian.v	Timer based controller with pedestrian button
Traffic_Light_Pedestrian_tb.v	Testbench for the pedestrian controller


