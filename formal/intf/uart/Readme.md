# Formal verification of modules

## Docker to use for formal
Not yet available publicly.

## Run formal
`podman run -i -t --rm -v <c:/open-logic>:/open-logic ghcr.io/sverrham/formal:24.10 bash`

The to run formal
`sby --yosys "yosys -m ghdl" -f olo_intf_uart.sby`


## Changes needed to the vhdl module
need to add the real generic default value for clock period, as ghdl does not work with real generics.

### Assertions
Trying to verify the Uart module, thinking process.
I want to verify the uart module, so I am trying to write some assertions that will verify the functionality.
My first thought was to verify that when an input is accepted so it should be transmitted we will see a startbit on the Tx line, so I tried this assertion:
`send_data : assert always Tx_Valid and Tx_Ready |=> {UArt_Tx[*]; not Uart_Tx};`
This passed the proof and cover, but there was this in the log:
```
formal_olo_intf_uart.psl:20:16:warning: property cannot fail [-Wuseless]
SBY 21:19:42 [olo_intf_uart_proof] base: send_data : assert always Tx_Valid and Tx_Ready |=> {Uart_Tx[*]; not Uart_Tx};
```
So it seems that assertion does not do anything, as it will never fail :/

Why will it never fail? 
- Is it the trigger? the left side of `|=>`
- Is it the undefined length of `Uart_Tx[*]`?

Changing the assertion to:
```
   send_data : assert always Tx_Valid and Tx_Ready |=> {Uart_Tx; not Uart_Tx};
```
The warning dissapears and the assertion fails, as expected since it is not expected that the TX goes low when it is accepted, it accepts data when transmitting last stopbit so there can be some cycles before the start bit is transmitted.


#### Assumptions
Some more updates after a bit discussion.
Assumptions to clean up some asserts.
1. `assume Rst` So we should have a reset in the start.
2. `assume always not Rst -> not Rst` When reset is cleared its assumed it will not be set next cycle.

#### Assertions
I expect the input to only accept one dataword and send it before accepting the next.
So to test that I assert that there is only one Tx_Ready out pulse, it can not be high two consecutive clock cycles.
`assert always Tx_Valid and Tx_Ready |=> not Tx_Ready`

TX data: I want to verify that when the module accepts a dataword, Tx_Ready high at same time as Tx_Valid high there will be a start bit transmitted a couple of cycles later.
`assert always Tx_Valid and Tx_Ready and not Rst |=> {UArt_Tx[*2 to 3]; not Uart_Tx}`

TX Data correctness, Not quite sure how to test this at the moment, so currently this is not tested.

RX validation:
I want to verify that we can not get two valid signals right after one another, there should only be one valid and then not another one
until a complete word has been received on the Uart_Rx signal.
`assert always Rx_Valid and not Rst |=> not Rx_Valid;`

