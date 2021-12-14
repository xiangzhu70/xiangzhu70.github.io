


# Host-PCI DMA TX/RX Design


### Notation

* Buffer descriptor (BD)
    * points to a DMA buffer
    * a packet can take multiple BDs, for various length

* sw — software running on the host
* hw — hardware PCI device, such as ethernet controllers, switches, FPGA , anything exchanging data with CPU
* sw_get -- a pointer to the slot hardware will get next
* sw_put -- a pointer to the slot hardware will put next
* hw_get -- a pointer to the slot hardware will get next
* hw_put -- a pointer to the slot hardware will put next

### Design
It is often seen that TX and RX DMA buffer descriptor queues are designed differently in hardware.  It is unnecessary.  This simple design unifies the TX and RX buffer descriptor queue operations.  Only the app-level logic -- buffer data processing, are different for TX and RX.

* Buffer descriptor ring
    * 4 pointers, chasing one after another
        * sw_get ← sw_put ← hw_get ← hw_put ← (sw_get)
        * sw reads hw_* pointers, and advances(writes) own sw_* pointers
        * hw reads sw_* pointers, and advances(writes) own hw_* pointers
    * 4 BD states.  4 pointers divide the ring buffer into 4 regions.  A BD falls in one of the 4 possible regions.  For TX and RX, the meanings of the BDs in the regions are different.  See the “BD in between” rows in the table below for the meanings of BD state.
    * BD ring buffer and pointers are in memory.  No costly PCIe read by CPU.
    * For both packet TX and RX, the BD ring operation flows are the same.  The BD processing parts have different actions depending on TX or RX contexts.  See the “process BD data” rows in the table.

#### Table
|	    |	    |	    |   	|	    |	    |	    |	    |	    |	    |
|---	|---	|---	|---	|---	|---	|---	|---	|---	|---	|
|	    |	            |Software	|	|	|	|Hardware	|	|	|	|
|	|4 pointers	|sw_get	|	|sw_put	|	|hw_get	|	|hw_put	|	|
|	| BD operation	| while (sw_get != hw_put) {process BD data; advance sw_get}|| (sw_put != sw_get) {process BD data; advance sw_put;}	|	|while (hw_get != sw_put) {process BD data; advance hw_get;}	|	|while (hw_put != hw_get) {process BD data; advance hw_put}|
|	| |	|	|	|	|	|	|	|	|	|	|
|TX	|BD in between	|	|BD with free data buffer, ready for pkt_tx() to fill in the data	|	|BD with TX data filled, waiting in memory for hw to take	|	|BD with TX data inside ASIC, waiting to be sent out	|	|BD with hw sending done. Waiting for sw to take back |
|	|driven by	|background tx thread	|	|app layer	|	|hw	|	|hw	|	|
|	|triggered by	|intr_tx_done, async	|	|sync, called by pkt_tx	|	|hw poll, or sw PCI write	|	|hw internal	|	|
|	|process BD data	|free data buffer	|	|fill data buffer pointed by BD	|	|DMA fetching pkt from memory into ASIC	|	|Send pkt to wire	|	|
|	|Good healthy state	|	|many BDs, so pkt_tx() can be called without being blocked	|	|few, hw quickly follows to take the BDs	|	|few, hw quickly follows to send pkt out, not stuck	|	|few, sw background thread quickly takes back BD, frees the associated data buffer, makes BD availble for next tx	|
|	|Bad state	|	|few, or zero, pkt_tx() is blocked	|	|many, hw is stuck	|	|many, hw is stuck	|	|many, sw is stuck	|
|	|	|	|	|	|	|	|	|	|	|
|RX	|BD in between	|	|BD with received data acknoledged by software	|	|BD with rx data processed.  Waiting for hw to take back.  The associated buffer are free for hw to fill in rx data.	|	|BD accepted by hw, available to be filled if there is any incoming pkt from wire	|	|BD with data recieved in memory.  Waiting for sw to take.	|
|	|driven by	|background rx thread	|	|whichever pkt processing thread	|	|hw	|	|hw	|	|
|	|triggered by	|intr_rx_pkt_arrived, async	|	|	|	|hw poll, or sw PCI write	|	|In good state waiting for data arriving from wire; In bad state, waiting for BD	|	|
|	|process BD data	|process received pkt in place, or dispatch to different threads	|	|process received pkt data	|	|hw waiting for data from wire.  if there is, fill data.  if not, wait here.	|	|Get free host memory from avaialble BD, DMA delivering pkt data from ASIC to memory	|	|
|	|Good healthy state	|	|0, or few	|	|few, hw quickly follows to take back BD, so it has data buffer for filling the next received pkt	|	|many, hw has many available BDs and buffers to fill pkts	|	|few, sw quickly responds to intr_rx_pkt_arrived	|
|	|Bad state	|	|many BDs, sw is stuck, not procesing received data	|	|many, hw is stuck	|	|few, hw has no BD available for filling pkts.	|	|many, sw is stuck, not respoinding to intr_rx_pkt_arrived	|
|	|	|	|	|	|	|	|	|	|	|

#### TX (sw->hw) Procedure
 1. Set up packet data buffer and descriptor ring
	 a. Allocate a packet data buffer with N equal-sized slots in contiguous host memory.
	 b. Set up a descriptor ring of size N.  Each one is associated with one data buffer slot.
	 
 2. TX setup.  Set sw_put, hw_get and hw_put to 0.  Set sw_get to N-1.  This means there are N-1 data buffer slots for software to fill in data.
 3. A software app calls pkt_tx().
 4. Software tries to get space (it tries to advance sw_put  if sw_put+1 != sw_get.  For jumbo packet larger than one data buffer slot, it tries to get multiple slots.
 5. Once software sees it can get enough slots, it fills data into the data buffer, and then advances sw_put
 6. Hardware polls sw_put.  Once it notices hw_get is behind sw_put, it loops to catch up with sw_put.  In each loop, it advances hw_get, then consumes the data (DMAing data into hardware, sending them out to ports), and then advances hw_put (putting space back), and sends interrupt tx_done to host.  The loop stops when hw_get catches up sw_put, hw_put catches up hw_get.
 7. A background TX thread works to collect descriptor back.  Upon the interrupt tx_done, it advances sw_get to hw_put-1.

#### RX (hw->sw) Procedure

 1. 1 is the same as TX
 2. RX setup.  Set hw_put, hw_get and sw_put to 0.  Set hw_get to N-1.  This means there are N-1 data buffer slots for hardware to fill in data.
 3. Upon receiving a packet, hardware tries to send it to the host.  It tries get space.  It tries to advance hw_put  if hw_put+1 != hw_get.  For jumbo packet larger than one data buffer slot, it tries to get multiple slots.
 4. Once software sees it can get enough slots, it fills data into the host data buffer (via DMA), and then advances hw_put.  And then it sends interrupt rx_recv.
 5. Software polls hw_put or handles interrupt rx_recv.  Once it notices sw_get is behind hw_put, it loops to catch up with hw_put.  In each loop, it advances sw_get, consumes the data (handling the received packets), and then advances sw_put (putting space back).  The loop stops when sw_get catches up hw_put, and sw_put catches up sw_get.
 6. Hardware polls sw_put, and advances hw_get to sw_put-1.
 


### Capabilities

This design is non-blocking, achieving the data rate at full hw capacity (bottlenecked by only the hardware DMA controller and PCIe bandwidth).

### Diagnostics, monitoring

The 4 pointers can clearly show if the operations are healthy, or which part is stuck.  See the “Good state” and “Bad state” rows.  

### Adapt to different hardware designs

If the hardware  does not cleanly follow the above design, then a software adaption layer is needed to glue the logic between the actual hardware and the software processing logic with the above design.  The glue layer is just a few functions to translate the hardware behaviors.

Typically any hardware design achieving the same results has somewhat similar design, such as hardware continuously fetching data following some queue pointers.  The software glue layer need to set up BD following the actual hardware definition.  It will also translate the results of the hardware actions into hw_get and hw_put changes, so that the software in the above design can understand and act. 

