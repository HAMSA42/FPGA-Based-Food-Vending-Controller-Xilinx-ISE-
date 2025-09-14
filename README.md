# FPGA-Based-Food-Vending-Controller-Xilinx-ISE-
using FSM model (Finite state machine) implemented the fpga-based food vending machine

module vending_controller (
    input  wire        clk,
    input  wire        rst,

    // coin inputs
    input  wire        coin5,
    input  wire        coin10,
    input  wire        coin25,

    // product selections
    input  wire        selA,
    input  wire        selB,
    input  wire        selC,

    input  wire        cancel,

    output reg  [2:0]  dispense,   // one-hot pulse: [2]=A, [1]=B, [0]=C
    output reg         out_of_stock,

    output reg  [3:0]  chg25,
    output reg  [3:0]  chg10,
    output reg  [3:0]  chg5,

    output reg  [7:0]  balance
);

    // Prices
    parameter PRICE_A = 8'd50;
    parameter PRICE_B = 8'd65;
    parameter PRICE_C = 8'd100;

    // Inventory
    reg [7:0] invA = 8'd5;
    reg [7:0] invB = 8'd5;
    reg [7:0] invC = 8'd5;

    // Latched selections
    reg selA_r, selB_r, selC_r;

    // FSM states
    localparam IDLE           = 3'd0,
               ACCEPT         = 3'd1,
               CHECK_SEL      = 3'd2,
               DISPENSE       = 3'd3,
               RETURN_CHANGE  = 3'd4,
               OUT_OF_STOCK_S = 3'd5,
               CANCEL_RETURN  = 3'd6;

    reg [2:0] state, next_state;

    integer change_tmp, tmp;

    // Sequential part
    always @(posedge clk) begin
        if (rst) begin
            state <= IDLE;
            balance <= 0;
            selA_r <= 0; selB_r <= 0; selC_r <= 0;
            dispense <= 0;
            out_of_stock <= 0;
            chg25 <= 0; chg10 <= 0; chg5 <= 0;
        end else begin
            state <= next_state;

            // coin inputs
            if (coin5 && balance <= 250) balance <= balance + 5;
            if (coin10 && balance <= 245) balance <= balance + 10;
            if (coin25 && balance <= 230) balance <= balance + 25;

            // latch selections
            if (selA) selA_r <= 1;
            if (selB) selB_r <= 1;
            if (selC) selC_r <= 1;

            // defaults
            dispense <= 0;
            out_of_stock <= 0;
        end
    end

    // Next-state logic
    always @(*) begin
        next_state = state;
        case (state)
            IDLE: begin
                if (cancel && balance>0) next_state = CANCEL_RETURN;
                else if (selA_r||selB_r||selC_r) next_state = CHECK_SEL;
                else next_state = ACCEPT;
            end
            ACCEPT: begin
                if (cancel && balance>0) next_state = CANCEL_RETURN;
                else if (selA_r||selB_r||selC_r) next_state = CHECK_SEL;
                else next_state = ACCEPT;
            end
            CHECK_SEL: begin
                if (selA_r) begin
                    if (invA==0) next_state = OUT_OF_STOCK_S;
                    else if (balance>=PRICE_A) next_state = DISPENSE;
                    else next_state = ACCEPT;
                end else if (selB_r) begin
                    if (invB==0) next_state = OUT_OF_STOCK_S;
                    else if (balance>=PRICE_B) next_state = DISPENSE;
                    else next_state = ACCEPT;
                end else if (selC_r) begin
                    if (invC==0) next_state = OUT_OF_STOCK_S;
                    else if (balance>=PRICE_C) next_state = DISPENSE;
                    else next_state = ACCEPT;
                end else next_state = ACCEPT;
            end
            DISPENSE: next_state = RETURN_CHANGE;
            RETURN_CHANGE: next_state = IDLE;
            OUT_OF_STOCK_S: begin
                if (cancel && balance>0) next_state = CANCEL_RETURN;
                else if (!selA_r && !selB_r && !selC_r) next_state = IDLE;
                else next_state = OUT_OF_STOCK_S;
            end
            CANCEL_RETURN: next_state = IDLE;
            default: next_state = IDLE;
        endcase
    end

    // Actions on state transitions
    always @(posedge clk) begin
        if (!rst) begin
            case (next_state)
                DISPENSE: begin
                    if (selA_r) begin
                        dispense <= 3'b100;
                        if (invA>0) invA <= invA-1;
                        change_tmp = balance - PRICE_A;
                    end else if (selB_r) begin
                        dispense <= 3'b010;
                        if (invB>0) invB <= invB-1;
                        change_tmp = balance - PRICE_B;
                    end else if (selC_r) begin
                        dispense <= 3'b001;
                        if (invC>0) invC <= invC-1;
                        change_tmp = balance - PRICE_C;
                    end else change_tmp=0;

                    tmp=change_tmp;
                    chg25 <= tmp/25; tmp=tmp%25;
                    chg10 <= tmp/10; tmp=tmp%10;
                    chg5  <= tmp/5;
                    balance <= 0;
                    selA_r <= 0; selB_r <= 0; selC_r <= 0;
                end
                RETURN_CHANGE: ; // keep change values for 1 cycle
                OUT_OF_STOCK_S: out_of_stock <= 1;
                CANCEL_RETURN: begin
                    tmp = balance;
                    chg25 <= tmp/25; tmp=tmp%25;
                    chg10 <= tmp/10; tmp=tmp%10;
                    chg5  <= tmp/5;
                    balance <= 0;
                    selA_r <= 0; selB_r <= 0; selC_r <= 0;
                end
            endcase
        end
    end

endmodule

