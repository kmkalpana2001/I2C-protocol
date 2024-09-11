# I2C-protocol

# Contents 
 <div class="toc">
  <ul>
    <li><a href="#header-1">Introduction to I2C Protocol</a></li>
	</ul>
</div>

 <div class="toc">
  <ul>
    <li><a href="#header-2">Working of I2C protocol</a></li>
	</ul>
</div>

<div class="toc">
  <ul>
    <li><a href="#header-3">Start and Stop Conditions</a></li>
	</ul>
</div>

  
<div class="toc">
  <ul>
    <li><a href="#header-4">I2C Packet format</a></li>
	</ul>
</div>

<div class="toc">
  <ul>
    <li><a href="#header-5">Design code for I2C</a></li>
	</ul>
</div>


## <h1 id="header-1">Introduction to I2C Protocol</h1>

I2C stands for Inter-Integrated-Circuit. It is a two wire interface asd compared to SPI which consist of four wires. Here one wore is to sending the clk signal and other wire is for send the data signal.It is a widely used protocol for short distance communication. Max range of I2C is (400KHz-100MHz).


## <h2 id="header-2">Working of I2C protocol</h2>

It uses only 2 bi-directional open-drain lines for data communication called SDA and SCL. Both these lines are pulled high.

Serial Data (SDA) – Transfer of data takes place through this pin.

Serial Clock (SCL) – It carries the clock signal.

![image](https://github.com/user-attachments/assets/d27d16c7-c941-4feb-b9c8-6e63952cf7bf)

According to I2C protocols, the data line can not change when the clock line is high, it can change only when the clock line is low. The 2 lines are open drain, hence a pull-up resistor is required so that the lines are high since the devices on the I2C bus are active low. The data is transmitted in the form of packets which comprises 9 bits. The sequence of these bits are –

Start Condition – 1 bit

Slave Address – 8 bit

Acknowledge – 1 bit


## <h3 id="header-3">Start and Stop Conditions</h3>

We need to send start signal and this signal(SDA) should go from high to low. while SCL should stay high. After sending start condition need to send adddress of peripheral which we wish to communicate the data. For write we need to apply 1 in address bus after 8-bit register. And to read we need to apply 0 in address bus after 8-bit register.

![image](https://github.com/user-attachments/assets/c0827708-2a90-47bd-9a50-567355c466d1)

Once we send an address now we  wait for the peripheral to send an acknowledgement. Acknowledgement will be 0 or 1. After recieving the acknowledgement we start sending data. Again wait for the acknowledgement then data go to adddress bus and  and then stop condition.

Once we complete sending an address and recieve an acknowledgement from a peripheral will now pull our line to an high impedence state so that peripheral could now send the data on the same line(SDA)line . Then will wait for the data to send from peripheral.

In reading condition acknowledgement is not required so will directly send the stop signal.


## <h4 id="header-4">I2C Packet format</h4>

In the I2C communication protocol, the data is transmitted in the form of packets. These packets are 9 bits long, out of which the first 8 bits are put in SDA line and the 9th bit is reserved for ACK/NACK i.e. Acknowledge or Not Acknowledge by the receiver. 

 START condition plus address packet plus one more data packet plus STOP condition collectively form a complete Data transfer.


## <h5 id="header-5">Design code for I2C</h5>
```verilog
module eeprom_top
  (
 input clk,
 input rst,
 input newd,
 input ack,
 input wr,   
 output scl,
 inout sda,
 input [7:0] wdata,
 input [6:0] addr, /////8-bit  7-bit : addr 1-bit : mode
 output reg [7:0] rdata,
 output reg done
  );
  
 reg sda_en = 0;
  
 reg sclt, sdat, donet; 
 reg [7:0] rdatat; 
 reg [7:0] addrt;
 
  
parameter idle = 0, check_wr = 1, wstart = 2, wsend_addr = 3, waddr_ack = 4, 
                wsend_data = 5, wdata_ack = 6, wstop = 7, rsend_addr = 8, 
                raddr_ack = 9, rsend_data = 10,
                rdata_ack = 11, rstop = 12 ;
  
  
  
  
  reg [3:0] state;
  reg sclk_ref = 0;
  integer count = 0;
  integer i = 0;
  
  ////100 M / 400 K = N
  ///N/2
  
  
  always@(posedge clk)
    begin
      if(count <= 9) 
        begin
           count <= count + 1;     
        end
      else
         begin
           count     <= 0; 
           sclk_ref  <= ~sclk_ref;
         end	      
    end
  
  
  
  always@(posedge sclk_ref, posedge rst)
    begin 
      if(rst == 1'b1)
         begin
           sclt  <= 1'b0;
           sdat  <= 1'b0;
           donet <= 1'b0;
         end
       else begin
         case(state)
           idle : 
           begin
              sdat <= 1'b0;
              done <= 1'b0;
              sda_en  <= 1'b1;
              sclt <= 1'b1;
              sdat <= 1'b1;
             if(newd == 1'b1) 
                state  <= wstart;
             else 
                 state <= idle;         
           end
         
            wstart: 
            begin
              sdat  <= 1'b0;
              sclt  <= 1'b1;
              state <= check_wr;
              addrt <= {addr,wr};
            end
            
            
            
            check_wr: begin
                ///addr remain same for both write and read
              if(wr == 1'b1) 
                 begin
                 state <= wsend_addr;
                 sdat <= addrt[0];
                 i <= 1;
                 end
               else 
                 begin
                 state <= rsend_addr;
                 sdat <= addrt[0];
                 i <= 1;
                 end
            end
         
                    
 
 
         
         
            wsend_addr : begin                
                      if(i <= 7) begin
                      sdat  <= addrt[i];
                      i <= i + 1;
                      end
                      else
                        begin
                          i <= 0;
                          state <= waddr_ack; 
                        end   
                    end
         
         
           waddr_ack : begin
             if(ack == 1'b1) begin
               state <= wsend_data;
               sdat  <= wdata[0];
               i <= i + 1;
               end
             else
               state <= waddr_ack;
           end
         
         wsend_data : begin
           if(i <= 7) begin
              i     <= i + 1;
              sdat  <= wdata[i]; 
           end
           else begin
              i     <= 0;
              state <= wdata_ack;
           end
         end
         
          wdata_ack : begin
             if(ack == 1'b1) begin
               state <= wstop;
               sdat <= 1'b0;
               sclt <= 1'b1;
               end
             else begin
               state <= wdata_ack;
             end 
            end
         
              
         
         wstop: begin
              sdat  <=  1'b1;
              state <=  idle;
              done  <=  1'b1;  
         end
         
         ///////////////////////read state
         
         
          rsend_addr : begin
                     if(i <= 7) begin
                      sdat  <= addrt[i];
                      i <= i + 1;
                      end
                      else
                        begin
                          i <= 0;
                          state <= raddr_ack; 
                        end   
                    end
         
         
           raddr_ack : begin
             if(ack == 1'b1) begin
               state  <= rsend_data;
               sda_en <= 1'b0;
             end
             else
               state <= raddr_ack;
           end
         
         rsend_data : begin
                   if(i <= 7) begin
                         i <= i + 1;
                         state <= rsend_data;
                         rdata[i] <= sda;
                      end
                      else
                        begin
                          i <= 0;
                          state <= rstop;
                          sclt <= 1'b1;
                          sdat <= 1'b0;  
                        end         
         end
          
        
         
         
         rstop: begin
              sdat  <=  1'b1;
              state <=  idle;
              done  <=  1'b1;  
              end
         
         
         default : state <= idle;
         
          	 endcase
          end
  end
  
 assign scl = (( state == wstart) || ( state == wstop) || ( state == rstop)) ? sclt : sclk_ref;
 assign sda = (sda_en == 1'b1) ? sdat : 1'bz;
endmodule
```
