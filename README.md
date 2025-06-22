# Pipeline-processor-design-module PipelineProcessor(
    input clk, reset
);
    // Pipeline registers
    reg [31:0] FD_IR, DE_IR, EW_IR;
    reg [31:0] DE_reg1, DE_reg2, EW_result;
    reg [4:0] DE_rd, EW_rd;
    
    // Register file
    reg [31:0] regfile [0:31];
    
    // Data memory
    reg [31:0] memory [0:255];
    
    // Pipeline control signals
    reg FD_valid, DE_valid, EW_valid;
    reg stall;
    
    // Fetch Stage
    always @(posedge clk) begin
        if (reset) FD_IR <= 0;
        else if (!stall) FD_IR <= memory[PC >> 2];
    end
    
    // Decode Stage
    always @(posedge clk) begin
        if (reset | stall) begin
            DE_IR <= 0;
            DE_valid <= 0;
        end else begin
            DE_IR <= FD_IR;
            DE_reg1 <= regfile[rs];
            DE_reg2 <= regfile[rt];
            DE_rd <= rd;
            DE_valid <= 1;
        end
    end
    
    // Execute Stage
    always @(posedge clk) begin
        if (reset) begin
            EW_IR <= 0;
            EW_valid <= 0;
        end else begin
            EW_IR <= DE_IR;
            case (opcode)
                ADD: EW_result <= DE_reg1 + DE_reg2;
                SUB: EW_result <= DE_reg1 - DE_reg2;
                LOAD: EW_result <= DE_reg1 + immediate;
            endcase
            EW_rd <= DE_rd;
            EW_valid <= DE_valid;
        end
    end
    
    // Writeback Stage
    always @(posedge clk) begin
        if (EW_valid) begin
            if (opcode == LOAD)
                regfile[EW_rd] <= memory[EW_result >> 2];
            else
                regfile[EW_rd] <= EW_result;
        end
    end
    
    // Hazard detection unit
    always @(*) begin
        stall = 0;
        // Detect load-use hazard
        if ((DE_IR[31:26] == LOAD) && 
            ((FD_IR[25:21] == DE_rd) || (FD_IR[20:16] == DE_rd)))
            stall = 1;
    end
endmodule
