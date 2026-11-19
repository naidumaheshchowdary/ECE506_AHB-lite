module ahb_lite_slave #(
  parameter logic [31:0] BASE_ADDR = 32'h0000_0000
) (
  input  logic        HCLK,
  input  logic        HRESETn,
  // address and control phase
  input  logic        HSEL,
  input  logic [31:0] HADDR,
  input  logic [1:0]  HTRANS,
  input  logic        HWRITE,
  input  logic [2:0]  HSIZE,
  input  logic [2:0]  HBURST,     // connected, not used
  // data phase
  input  logic [31:0] HWDATA,
  input  logic        HREADY,     // bus ready, from the multiplexor
  // test configuration: number of wait states for a legal transfer (0 to 3)
  input  logic [1:0]  cfg_wait,
  output logic [31:0] HRDATA,
  output logic        HREADYOUT,
  output logic        HRESP
);

  localparam logic [2:0] SIZE_WORD    = 3'b010;

  // register file, reset to zero
  logic [3:0][31:0] regs;

  // address phase seen now
  logic ap_active;       // selected NONSEQ or SEQ transfer
  logic ap_legal;        // word access inside the window
  assign ap_active = HSEL & HTRANS[1];   // HTRANS[1] set means NONSEQ or SEQ
  assign ap_legal  = ap_active & (HSIZE == SIZE_WORD) & (HADDR[31:4] == BASE_ADDR[31:4]);

  // data phase state, loaded when an address phase is accepted
  logic       dp_active;     // a NONSEQ/SEQ transfer for this slave is in its data phase
  logic       dp_legal;
  logic       dp_write;
  logic [1:0] dp_idx;
  logic [1:0] wait_cnt;      // wait-state cycles still to insert
  logic       err_second;    // second cycle of the ERROR response

  // response
  always_comb begin
    HREADYOUT = 1'b1;
    HRESP     = 1'b0;
    HRDATA    = 32'h0000_0000;
    if (dp_active) begin
      if (dp_legal) begin
        HREADYOUT = (wait_cnt == 2'd0);
        if (!dp_write) HRDATA = regs[dp_idx];
      end else begin
        HRESP     = 1'b1;
        HREADYOUT = err_second;
      end
    end
  end

  always_ff @(posedge HCLK or negedge HRESETn) begin
    if (!HRESETn) begin
      regs       <= '0;
      dp_active  <= 1'b0;
      dp_legal   <= 1'b0;
      dp_write   <= 1'b0;
      dp_idx     <= 2'd0;
      wait_cnt   <= 2'd0;
      err_second <= 1'b0;
    end else if (HREADY) begin
      // the data phase in progress completes at this edge
      if (dp_active && dp_legal && dp_write)
        regs[dp_idx] <= HWDATA;
      // accept the address phase on the bus
      dp_active  <= ap_active;
      dp_legal   <= ap_legal;
      dp_write   <= HWRITE;
      dp_idx     <= HADDR[3:2];
      wait_cnt   <= cfg_wait;
      err_second <= 1'b0;
    end else if (dp_active) begin
      // bus is held: this slave is inserting wait states or sending the first ERROR cycle
      if (dp_legal) begin
        if (wait_cnt != 2'd0) wait_cnt <= wait_cnt - 2'd1;
      end else begin
        err_second <= 1'b1;
      end
    end
  end

  // unused inputs, kept so that lint tools stay quiet
  logic unused;
  assign unused = &{1'b0, HBURST, HADDR[1:0], HTRANS[0]};

endmodule
