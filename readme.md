
# Aurifex Labs' Verilog Style Guide

## Differences from software

Infinite concurrency if you want to think of it like that.
Structural with small scenes (see litmus tests in RISC-V spec)
A picture is helpful but verilog is code.

There's no way to "trace through the code". We will assume that most of the time you are writing Verilog you don't have software that will run on the device or the whole device is not yet in a state that can where software can be simulated (like the memory controller isn't implemented yet).

So it's better to think of the organization of the code like telling a story that describes a place.

Even sequential elements like FSMs and pipelines are describes structurally or in ways that can
be the inferred.

While behavioral code looks more like software, the inference is still generating a physical circuit. Just like software we need names to be able to debug.

## How much inference?
Inferring gates and wires from behavioral code often results in nice looking code that is difficult to debug. To be clear it is relatively easy to debug the logic using cover statements and inspecting waveforms, but to do PPA optimization or even know that your logic made it into the circuit requires having names for intermediate wires in the static timing analysis (STA) and critical to be able to do things like false path removal.

For example if you put logic assignment directly a case statement for a FSM you no longer have to name the control wires, but that also means you get a generated symbol in debugging.

So err on the side of dividing logic all the way down to the lowest level in a quality, hierarchial fashion and name everything.

FSM: Infer state only as a function of inputs and current state (and minimize inputs with meaningful named intermediate logic)
SRAM: Instantiate at the macro level and get timing from post-si or sdf data
MUX: Use the ternary(?) operator for 2 input and a case statement for larger.

Prefer wire and assign for combinatorial logic.
All counters and other stateful modules should use the register idiom and pure combinatorial logic.



## Basic Rules
Lines must end in semicolons

## SystemVerilog vs Verilog
always_ff
always_comb

## Formalish verifiation
sby

        /// Bottom of module in rtl

        `ifdef FORMAL
            initial begin
                control_state = 'h0;
            end

            always @(posedge clk) begin
                cover_idle: cover (control_state == 4'h0);
                cover_write: cover (control_state == 4'h8);
            end
        `endif
    endmodule

## Number Literals
Always specify the size of literals otherwise they are 32 bits.
assign a = 0; // 32 bit value inferred
assign a = 4'h0

## Clock and Reset
  
Always use clk and rst for the actual clock and reset to registers.
Use prefixed, camelcase for modules that modify or distribute, e.g., externalRst or videoClk.


## Verilog Module IO

all module inputs should be type wire and suffixed with _i.
all module inputs should be type wire and suffixed with _o.

## Register Idiom

all state bits (registers) should be resetable flipflops by default.
all outputs be suffixed with _r.
all inputs should be prefixed with next
register inputs and outputs should be delared reg type together.

Example:

    reg [3:0] state_r;
    reg [3:0] next_state

    always @(posedge clk) begin
        if(rst) begin
            state_r <= 0;
        end else begin
            state_r <= next_state;
        end
    end

    always @(*) begin
        if(enable_i) begin
            next_state = 4'h3;
        end
    end

### FSM Idiom

Use register idiom for the state.
Use a utility counter, that automaticall resets on every state change to keep number of states low.

It's very important to keep state count low so that the state machine can be one-hot encoded.
Try to keep to < 10 states. And remember it's a graph so the bigger the FSM is the more optimization is important and the more obfuscated the timing becomes.
But don't over optimize and have a state do two things with more state bits outside the state machine.
Try to use the counter to minimize states.
Async bus access requires multiple states, but there are different patterns.

Every module should have exactly one control state machine that is in it's own module.
Try to minimize the size of the FSM itself and use named control wires to an attached logic module. Harden together.

Use localparam to create named states suffixed with _STATE
Only <= to next_state in the FSM always block.
Use assign to set output (control) wires as a function of control_state.

Control outputs for a pipelined design need to be fast, so that's why rather than rely on inference it's best to move control logic into combinatorial code.

Example:

    localparam IDLE_STATE = 0;
    localparam COMMAND_ADDRESS_STATE = 1;

    wire bypassSelect;
    assign bypassSelect = control_state == COMMAND_ADDRESS_STATE || controlState == 

    always @(*) begin
            case(control_state_r)
                IDLE_STATE : begin
                    if(transaction_begin_i) begin
                        next_control_state = COMMAND_ADDRESS_STATE;
                    end
                end

                COMMAND_ADDRESS_STATE : begin 
                    next_control_state = IDLE_STATE;
                end

### Module parameters vs real inputs vs localparam

If it can be configured at runtime prefer adding a real input and converting to a parameter later to go from correctness to PPA optimization.

If a value needs to be known at hardening time then use a parameter.

Use WIDTH as a parameter for number of bits wide the element is if obvious.

Parameters of all types should be all caps underscore.

Use localparam for constants that should be generated (and then eventually use code generation).
For example, states for FSMs. (so enums basically)

Also use localparam for true constants (never need to be parameterized)
Example: RISC-V opcodes

## Advanced Verilog features

Features like generate.

Don't use them.
Anywhere that the basic features result in duplication of knowledge use code generation.

## Code Generation

Use high level languages.

If you like to get stuff done, use Python.
If you like to get stuff done and see it and are willing to deal with async, use Javascript.
If you like math, use Scala.

Just don't use Verilog.

But really use Javascript so that you get portability in the browser and the ability to hire web devs to build tools.

## Source Mapping
Generate valid Verilog identifiers as strings in the code gen language to fully qualify names to the bit level.
adder_input0_bit2 instead of adder.input[0][2]
And the just use find in your ide on the code gen source to find symbol fragments.
