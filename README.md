# UVM_Verification_of_Combinational_Adder
This repo shows the verification of a 4-bit combinational adder using Universal Verification Methodology(UVM)

## Code

<details><summary>RTL/Design Code</summary>

```systemverilog
///////DUT + Interface 
module adder(
  input clk, rst,
  input [3:0] a,b,
  output reg [4:0]y
);
  
  always@(posedge clk)
    begin
      if(rst)
        y <= 5'd0;
      else
        y <= a + b;
    end
  
endmodule

//////////////////////////////////////////////////////////

interface adder_if();
  
  logic clk;
  logic rst;
  logic [3:0]a;
  logic [3:0]b;
  logic [4:0]y;
  
endinterface
```
</details>

__________________________________________________________

<details><summary>Testbench Code</summary>

```systemverilog
`timescale 1ns / 1ps

`include "uvm_macros.svh"
import uvm_pkg::*;
/////////////////////////////////
//1.TRANSACTION

class transaction extends uvm_sequence_item; //dynamic component as uvm_object
  
  //decalring data memebers for randomization
  rand bit [3:0]a;
  rand bit [3:0]b;
  bit [4:0]y;
  
  function new(input string path = "transaction");  //1 arg as uvm_object
    super.new(path);
  endfunction
  
  //registering data members to uvm_factory for automation
  `uvm_object_utils_begin(transaction)
  `uvm_field_int(a, UVM_DEFAULT)
  `uvm_field_int(b, UVM_DEFAULT)
  `uvm_field_int(y, UVM_DEFAULT)
  `uvm_object_utils_end

endclass

//////////////////////////////////////////////////////////////////////////////
//2.SEQUENCE but we name it as generator
class generator extends uvm_sequence #(transaction);
  `uvm_object_utils(generator)
  
  transaction t;
  
  function new(input string path = "generator"); //1 arg as uvm_object 
    super.new(path);
  endfunction
  
  //task to communicate between sequence and driver 
  virtual task body();
    t = transaction::type_id::create("t"); //create() will have only 1 arg as transaction is uvm_object type
    repeat(10)
      begin
        start_item(t);  //sends the trans packet to driver when gets grant 
        t.randomize();
        finish_item(t); //after getting item_done from driver , prepares the next packet 
        `uvm_info("GEN", $sformatf("Data sent to Driver a: %0d, b: %0d",t.a, t.b), UVM_NONE);  
      end
  endtask
    
endclass
//////////////////////////////////////////////////////////////////////////////
//3.DRIVER , we dont have to build a separate class for sequencer so we go to Driver directly
class driver extends uvm_driver #(transaction);
  `uvm_component_utils(driver)
  
  function new(input string path = "driver", uvm_component parent = null); //2 args as uvm_component / static 
    super.new(path, parent);
  endfunction
  
  virtual adder_if aif; //interface handle so that driver and monitor can get access to the interface 
  transaction data; // a trans container 'data' to store the data sent by sequence
  
  //build phase
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    data = transaction::type_id::create("data"); //1 arg only
    
//config_db and get method to get access to interface in tb_top
    if(!uvm_config_db #(virtual adder_if)::get(this,"","aif",aif))
      `uvm_error("DRV", "Unable to access uvm_config_db");
  endfunction
  
//task to reset logic in DUT 
  task reset_dut();
    aif.rst <= 1'b1;
    aif.a <= 1'b0;
    aif.b <= 1'b0;
    repeat(5) @(posedge aif.clk); //apply/assert rst for 5 clk ticks
    aif.rst <= 1'b0;
    `uvm_info("DRV", "Reset Done", UVM_NONE);
  endtask
    
  //run_phase to communicate between driver and sequencer
  virtual task run_phase(uvm_phase phase);
    reset_dut();
    forever begin //using forever because driver has to be always ready to get packets and send item_done
      //driver's port 
      seq_item_port.get_next_item(data);
      
      //apply this data packet to the DUT through the interface 
      aif.a <= data.a;
      aif.b <= data.b;
      //send item_done signal to sequencer 
      seq_item_port.item_done();
      `uvm_info("DRV", $sformatf("Trigger DUT a: %0d, b: %0d", data.a, data.b), UVM_NONE);
      //wait for 2 clk ticks so that DUT gets enough time to process the logic before receiving another packet of inputs
      @(posedge aif.clk);
      @(posedge aif.clk);
    end
  endtask
  
endclass
//////////////////////////////////////////////////////////////////////////////
//4.MONITOR
class monitor extends uvm_monitor;
  `uvm_component_utils(monitor)
  
  //analysis port to connect monitor and scoreboard
  uvm_analysis_port #(transaction) send;
  
  virtual adder_if aif; //interface instance to access interface
  transaction t; //a trans packet to send data from monitor to scoreboard
  
  function new(input string path = "monitor", uvm_component parent = null); //2 args
    super.new(path, parent);
    //construct the recv port here itself
    send = new("send", this); //analysis_port 
  endfunction
  
  //build_phase
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    t = transaction::type_id::create("t"); //1 arg
    if(!uvm_config_db #(virtual adder_if)::get(this,"","aif",aif))
      `uvm_error("MON", "Unable to access uvm_config_db");
  endfunction
  
  //task to get data from DUT 
  virtual task run_phase(uvm_phase phase);
    @(negedge aif.rst); //we have rst for 5 clock ticks in the driver so in the 6th clock tick, the rst will go down i.e negative edge of rst , thats when we pass the data from DUT to monitor
    forever begin
      repeat(2) @(posedge aif.clk); //wait for 2 clk ticks just like driver
      //from DUT to monitor 
      t.a = aif.a;
      t.b = aif.b;
      t.y = aif.y;
      `uvm_info("MON", $sformatf("Data sent to Scoreboard a: %0d, b: %0d, y: %0d",t.a, t.b, t.y), UVM_NONE);
      send.write(t);//send trans packet to scoreboard using function "write" inside Scoreboard class
    end
    
  endtask
  
endclass
///////////////////////////////////////////////////////////////////////////
//5.SCOREBOARD
class scoreboard extends uvm_scoreboard;    
  `uvm_component_utils(scoreboard)
  
  //uvm_analysis implementation to get the data from monitor
  uvm_analysis_imp #(transaction,scoreboard) recv;
  
  transaction data; // trans data container to hold the packet sent by monitor
  
  function new(input string path = "scoreboard", uvm_component parent = null);
    super.new(path, parent);
    //construct the recv port here itself
    recv = new("recv", this);
  endfunction
  
  //build_phase
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    data = transaction::type_id::create("data"); //1 arg 
  endfunction
  
  //function write to get data from monitor
  virtual function void write(input transaction t);
    data = t; //pass the data in t container to data container
    `uvm_info("SCO", $sformatf("Data rcvd from Monitor a: %0d, b: %0d, y: %0d", data.a, data.b, data.y), UVM_NONE);
    
    //logic to check the stimulus and response 
    if(data.y == data.a + data.b)
      `uvm_info("SCO", "Test Passed", UVM_NONE)
    else
      `uvm_info("SCO", "Test failed", UVM_NONE);
  endfunction

endclass
///////////////////////////////////////////////////////////////////////////
////6.AGENT 
class agent extends uvm_agent;
  `uvm_component_utils(agent)
  
  //agent contains sequencer, driver and monitor
  uvm_sequencer #(transaction) seqr;
  driver d;
  monitor m;
  
  function new(input string path = "agent", uvm_component parent = null);
    super.new(path, parent); 
  endfunction
  
  //build_phase
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    seqr = uvm_sequencer #(transaction)::type_id::create("seqr", this); //2 args as uvm_component  
    d = driver::type_id::create("d", this);
    m = monitor::type_id::create("m", this);
  endfunction
  
  //connect phase to connect sequencer and driver
  virtual function void connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    d.seq_item_port.connect(seqr.seq_item_export); //connect driver and sequencer
  endfunction
  
endclass
///////////////////////////////////////////////////////////////////////////
// 7.ENV
class env extends uvm_env;
  `uvm_component_utils(env)
  
  //env contains agent and scoreboard
  agent a;
  scoreboard s;
  
  function new(input string path = "env", uvm_component parent = null);
    super.new(path, parent);
  endfunction
  
  //build_phase
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    a = agent::type_id::create("a", this);
    s = scoreboard::type_id::create("s", this);
  endfunction
  
  //connect phase to connect mon and sco
  virtual function void connect_phase(uvm_phase phase);
    super.connect_phase(phase);
    a.m.send.connect(s.recv); //connected mon and sco
  endfunction
   
endclass
///////////////////////////////////////////////////////////////////////////
// 8.TEST
class test extends uvm_test;
  `uvm_component_utils(test)
  
  //test contains env and sequence(generator as named by us)
  env e; 
  generator gen;
  
  function new(input string path = "test", uvm_component parent = null);
    super.new(path, parent);
  endfunction
  
  //build_phase
  virtual function void build_phase(uvm_phase phase);
    super.build_phase(phase);
    e = env::type_id::create("e", this); //2 args as uvm_component
    gen = generator::type_id::create("gen"); //1 arg as uvm_object
  endfunction
  
  //task to start the sequence from test class
  virtual task run_phase(uvm_phase phase);
    phase.raise_objection(this); //to hold the simulator till sequence 
    gen.start(e.a.seqr);  //seq.start(e.a.seqr)
    #60;
    phase.drop_objection(this);
  endtask

endclass
///////////////////////////////////////////////////////////////////////////
// 9.TB_TOP
module tb;
 
//tb contains test, interface and DUT
  adder_if aif(); //have to add parenthesis for interface instance
  adder dut(.clk(aif.clk), .rst(aif.rst), .a(aif.a), .b(aif.b), .y(aif.y)); //connection DUT and test class thorugh interface 
  //we dont need to create a test class ,, use run_test
  
  //clk and reset initialization
  initial begin
    aif.clk = 0;
    aif.rst = 0;
    end
  
  //clk generation
  always #10 aif.clk = ~aif.clk; //50Mhz 
  
  initial begin
    //give access of interface to drv and mon
    uvm_config_db #(virtual adder_if)::set(null,"*","aif", aif);
    run_test("test");    
  end
  
  initial begin
    $dumpfile("dump.vcd");
    $dumpvars;
  end
  
endmodule
```
</details>

__________________________________________________________

<details><summary>Simulation</summary><br>

![alt text](<Sim/UVM Based Sequential Adder Part1.png>)
![alt text](<Sim/UVM Based Sequential Adder Part2.png>)
Sim/UVM Based Sequential Adder Part3.png

</details>

__________________________________________________________

<details><summary>Waveform</summary><br>

![alt text](<Sim/UVM Based Sequential Adder Waveform.png>)

</details>