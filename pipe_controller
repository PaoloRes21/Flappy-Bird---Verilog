`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Module Name: pipe_controller
// Description: Generates a moving pipe obstacle with a pseudo-random vertical gap.
//              When the pipe moves off-screen or when pipe_reset is asserted, the
//              pipe_x resets to SCREEN_WIDTH and a 4-bit LFSR is updated to generate
//              a new gap value. The computed gap_y is clamped so that it does not
//              exceed MAX_GAP_Y.
// 
// Parameters:
//   SCREEN_WIDTH : Screen width (or pipe reset position).
//   PIPE_WIDTH   : Width of the pipe obstacle.
//   GAP_HEIGHT   : Fixed vertical gap size through which the bird can pass.
//   PIPE_SPEED   : Speed at which the pipe moves leftward (pixels per game clock tick).
//   MIN_GAP_Y    : Minimum vertical position for the gap (smallest allowed gap top).
//   MAX_GAP_Y    : Maximum vertical position for the gap (largest allowed gap top).
//////////////////////////////////////////////////////////////////////////////////

// Module Name: pipe_controller
// Description: Generates a moving pipe obstacle with a pseudo-random vertical gap.
//              Accepts dynamic speed input (pipe_speed) for difficulty scaling.
//
// Parameters:
//   SCREEN_WIDTH : Screen width (or pipe reset position).
//   PIPE_WIDTH   : Width of the pipe obstacle.
//   GAP_HEIGHT   : Vertical gap size the bird must fly through.
//   MIN_GAP_Y    : Minimum vertical position for the gap.
//   MAX_GAP_Y    : Maximum vertical position for the gap.
//////////////////////////////////////////////////////////////////////////////////

module pipe_controller #(
    parameter SCREEN_WIDTH = 1440,
    parameter PIPE_WIDTH   = 40,
    parameter GAP_HEIGHT   = 120,
    parameter MIN_GAP_Y    = 15,
    parameter MAX_GAP_Y    = 457
)(
    input clk,                    // System clock
    input rst,                    // Active-low reset
    input game_clk,               // Game clock (~30Hz)
    
    input pipe_reset,             // Reset pipe position and gap
    input [7:0] pipe_speed,       // Dynamic pipe speed based on difficulty
    
    output reg [10:0] pipe1_x,    // First pipe horizontal position
    output reg [10:0] pipe1_gap_y,// Vertical position of first pipe gap
    output reg [10:0] pipe2_x,    // Second pipe horizontal position
    output reg [10:0] pipe2_gap_y // Vertical position of second pipe gap
);
     // Second pipe offset (distance between pipes)
    localparam PIPE_OFFSET = SCREEN_WIDTH / 2;

    reg [3:0] lfsr1, lfsr2;
    reg [10:0] gap_temp1, gap_temp2;
    
always @(posedge game_clk or negedge rst) begin
    if (!rst) begin
        pipe1_x <= SCREEN_WIDTH + 200;
        pipe2_x <= SCREEN_WIDTH + 900 + PIPE_OFFSET;

        lfsr1 <= 4'b1010;
        lfsr2 <= 4'b1101;

        gap_temp1 = MIN_GAP_Y + (lfsr1 * 35);
        pipe1_gap_y <= (gap_temp1 > MAX_GAP_Y) ? MAX_GAP_Y : gap_temp1;

        gap_temp2 = MIN_GAP_Y + (lfsr2 * 35);
        pipe2_gap_y <= (gap_temp2 > MAX_GAP_Y) ? MAX_GAP_Y : gap_temp2;

    end else if (pipe_reset) begin
        pipe1_x <= SCREEN_WIDTH + 200;
        pipe2_x <= SCREEN_WIDTH + 900 + PIPE_OFFSET;

        lfsr1 <= {lfsr1[2:0], lfsr1[3] ^ lfsr1[2]};
        gap_temp1 = MIN_GAP_Y + (lfsr1 * 35);
        pipe1_gap_y <= (gap_temp1 > MAX_GAP_Y) ? MAX_GAP_Y : gap_temp1;

        lfsr2 <= {lfsr2[2:0], lfsr2[3] ^ lfsr2[2]};
        gap_temp2 = MIN_GAP_Y + (lfsr2 * 35);
        pipe2_gap_y <= (gap_temp2 > MAX_GAP_Y) ? MAX_GAP_Y : gap_temp2;

    end else begin
        // First pipe logic
        if (pipe1_x > pipe_speed) begin
            pipe1_x <= pipe1_x - pipe_speed;
        end else begin
            pipe1_x <= SCREEN_WIDTH + 200;
            lfsr1 <= {lfsr1[2:0], lfsr1[3] ^ lfsr1[2]};
            gap_temp1 = MIN_GAP_Y + (lfsr1 * 35);
            pipe1_gap_y <= (gap_temp1 > MAX_GAP_Y) ? MAX_GAP_Y : gap_temp1;
        end
        
        // Second pipe logic
        if (pipe2_x > pipe_speed) begin
            pipe2_x <= pipe2_x - pipe_speed;
        end else begin
            pipe2_x <= pipe1_x + PIPE_OFFSET; // <-- use pipe1_x for spacing
            lfsr2 <= {lfsr2[2:0], lfsr2[3] ^ lfsr2[2]};
            gap_temp2 = MIN_GAP_Y + (lfsr2 * 35);
            pipe2_gap_y <= (gap_temp2 > MAX_GAP_Y) ? MAX_GAP_Y : gap_temp2;
        end
    end
end


endmodule
