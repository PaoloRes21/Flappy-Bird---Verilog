`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Module: game_top.v
// Description:
//   - Creates ~25 MHz pixel clock via PLL
//   - Derives ~60 Hz game clock via divider
//   - Reads buttons/switches for start, flap, difficulty
//   - Updates bird physics, world position, and game_over
//   - Handles pipe collision & pipe reset
//   - Increments score/high_score & death count
//   - Instantiates: pipe_controller, drawcon, vga, score_display, accelerometer
//////////////////////////////////////////////////////////////////////////////////

module game_top(
    input clk, // System Clock
    input rst, // Active-Low Reset
    
    input [4:0] btn, // Buttons
    input [2:0] sw, // Switches
    
    // Accelerometer
    input accel_miso,
    output accel_mosi,
    output accel_sclk,
    output accel_cs,
    
    // LED's
    output [2:0] led,
    output led16_r, led17_r,
    
    // VGA
    output [3:0] pix_r,
    output [3:0] pix_g,
    output [3:0] pix_b,
    output hsync,
    output vsync,
    
    // Seven-Segment Display in the Nexys A7/4 Board
    output [6:0] seg,
    output [7:0] an
);

    wire pixclk;
    wire [3:0] draw_r, draw_g, draw_b;
    wire [10:0] curr_x, curr_y;

    reg [20:0] clk_div;
    reg game_clk;

    reg [10:0] blkpos_x;
    reg signed [10:0] blkpos_y;
    reg signed [10:0] bird_velocity;

    reg game_over, btn0_prev, pipe_reset_reg, pipe_init_reset;
    wire pipe_reset = pipe_reset_reg | pipe_init_reset;

    parameter BOTTOM = 11'd577;
    parameter PIPE_WIDTH = 40;
    parameter PIPE_GAP_SIZE = 250;
    localparam HITBOX_X_OFFSET = 40;
    localparam HITBOX_Y_OFFSET = 62;
    localparam HITBOX_WIDTH    = 126;
    localparam HITBOX_HEIGHT   = 87;

    wire [10:0] pipe1_x, pipe1_gap_y, pipe2_x, pipe2_gap_y;
    reg [7:0] score, high_score;
    reg scored_pipe1_flag, scored_pipe2_flag;

    reg [1:0] level;
    wire [7:0] pipe_speed;

    // Sprite control flags
    reg died_by_pipe;
    reg use_dead_sprite;
    reg use_dead_reflected_sprite;
    
    // Game Started
    reg game_started;

    // Flap animation
    reg use_flap_down_sprite;
    reg [3:0] flap_counter;
    
    // Death Counter
    reg [7:0] total_deaths;  // Tracks number of total deaths

    // Accelerometer unlocks
    wire hat_unlocked;
    wire [8:0] accel_x;
    wire [8:0] accel_y;
    wire [11:0] accel_mag;
    
    assign led16_r = hat_unlocked;
    assign led17_r = hat_unlocked;

    // Level selection logic
    always @(*) begin
        case (1'b1)
            sw[2]: level = 3;
            sw[1]: level = 2;
            sw[0]: level = 1;
            default: level = 0;
        endcase
    end

    assign led = (level == 3) ? 3'b111 :
                 (level == 2) ? 3'b011 :
                 (level == 1) ? 3'b001 : 3'b000;

    assign pipe_speed = (level == 3) ? 8'd34 :
                        (level == 2) ? 8'd26 :
                        (level == 1) ? 8'd23 : 8'd18;

    // Clock divider for ~60Hz game clock
    always @(posedge clk) begin
        if (!rst) begin
            clk_div <= 0;
            game_clk <= 0;
        end else begin
            if (clk_div == 21'd833333) begin
                clk_div <= 0;
                game_clk <= ~game_clk;
            end else begin
                clk_div <= clk_div + 1;
            end
        end
    end

    wire effective_game_clk = game_clk & ~game_over;

    // Main game logic
    always @(posedge game_clk) begin
        if (!rst) begin
            blkpos_x <= 11'd200;
            blkpos_y <= 11'd250; // Initial bird height
            bird_velocity <= 0;
            game_over <= 0;
            btn0_prev <= 0;
            pipe_reset_reg <= 0;
            pipe_init_reset <= 1; // Trigger pipe reset at power-on
            score <= 0;
            high_score <= 0;
            scored_pipe1_flag <= 0;
            scored_pipe2_flag <= 0;
            died_by_pipe <= 0;
            use_dead_sprite <= 0;
            use_dead_reflected_sprite <= 0;
            use_flap_down_sprite <= 0;
            flap_counter <= 0;
            game_started <= 0;
        end else if (!game_started && btn[0]) begin
            game_started <= 1;
            pipe_init_reset <= 1; // Reset pipe on first start
            use_dead_sprite <= 0;
            use_dead_reflected_sprite <= 0;
            died_by_pipe <= 0;
        end else begin
            pipe_init_reset <= 0;

            if (game_over) begin
                if (game_started) begin
                    bird_velocity <= bird_velocity + 11'sd2;
                    if (blkpos_y + bird_velocity >= BOTTOM + 12) begin
                        blkpos_y <= BOTTOM + 12;
                        bird_velocity <= 0;
                        use_dead_sprite <= 1;
                        if (died_by_pipe)
                            use_dead_reflected_sprite <= 1;
                    end else begin
                        blkpos_y <= blkpos_y + bird_velocity;
                    end
                end

                if (blkpos_y >= BOTTOM + 12 && btn[0] && ~btn0_prev) begin
                    blkpos_x <= 11'd200;
                    blkpos_y <= 11'd250;
                    bird_velocity <= 0;
                    game_over <= 0;
                    pipe_reset_reg <= 1;
                    score <= 0;
                    scored_pipe1_flag <= 0;
                    scored_pipe2_flag <= 0;
                    use_dead_sprite <= 0;
                    use_dead_reflected_sprite <= 0;
                    use_flap_down_sprite <= 0;
                    flap_counter <= 0;
                end else begin
                    pipe_reset_reg <= 0;
                end

                btn0_prev <= btn[0];
            end else begin
                if (!game_started) begin
                    bird_velocity <= 0;
                end else if (btn[0] && ~btn0_prev) begin
                    bird_velocity <= -11'sd19;
                end else begin
                    bird_velocity <= bird_velocity + 11'sd2;
                end

                btn0_prev <= btn[0];
                pipe_reset_reg <= 0;

                if (game_started) begin
                    if (blkpos_y + bird_velocity < 0) begin
                        blkpos_y <= 0;
                        bird_velocity <= 0;
                    end else begin
                        blkpos_y <= blkpos_y + bird_velocity;
                    end
                end

                flap_counter <= flap_counter + 1;
                if (flap_counter == 4'd8) begin
                    flap_counter <= 0;
                    use_flap_down_sprite <= ~use_flap_down_sprite;
                end

                // Ground collision only after game starts
                if (game_started && blkpos_y >= BOTTOM) begin
                    game_over <= 1;
                    died_by_pipe <= 0;
                    use_dead_sprite <= 1;
                    use_dead_reflected_sprite <= 0;
                    blkpos_y <= BOTTOM;
                    bird_velocity <= 0;
                    total_deaths <= total_deaths + 1;
                end

                // Pipe collision only after game starts
                if (game_started) begin
                
                    // Collision for pipe 1
                    if (((blkpos_x + HITBOX_X_OFFSET) < (pipe1_x + PIPE_WIDTH)) &&
                        ((blkpos_x + HITBOX_X_OFFSET + HITBOX_WIDTH) > pipe1_x)) begin
                        if (((blkpos_y + HITBOX_Y_OFFSET) < pipe1_gap_y) ||
                            ((blkpos_y + HITBOX_Y_OFFSET + HITBOX_HEIGHT) > (pipe1_gap_y + PIPE_GAP_SIZE))) begin
                            game_over <= 1;
                            died_by_pipe <= 1;
                            use_dead_sprite <= 0;
                            use_dead_reflected_sprite <= 0;
                            bird_velocity <= 0;
                            total_deaths <= total_deaths + 1;
                        end
                    end
                
                    // Collision for pipe 2
                    if (((blkpos_x + HITBOX_X_OFFSET) < (pipe2_x + PIPE_WIDTH)) &&
                        ((blkpos_x + HITBOX_X_OFFSET + HITBOX_WIDTH) > pipe2_x)) begin
                        if (((blkpos_y + HITBOX_Y_OFFSET) < pipe2_gap_y) ||
                            ((blkpos_y + HITBOX_Y_OFFSET + HITBOX_HEIGHT) > (pipe2_gap_y + PIPE_GAP_SIZE))) begin
                            game_over <= 1;
                            died_by_pipe <= 1;
                            use_dead_sprite <= 0;
                            use_dead_reflected_sprite <= 0;
                            bird_velocity <= 0;
                            total_deaths <= total_deaths + 1;
                        end
                    end
                end

                // Score update only after game starts
                if (game_started) begin
                    // Score for pipe 1
                    if ((pipe1_x + PIPE_WIDTH < 200) && !scored_pipe1_flag) begin
                        score <= score + 1;
                        if (score + 1 > high_score)
                            high_score <= score + 1;
                        scored_pipe1_flag <= 1;
                    end else if (pipe1_x + PIPE_WIDTH >= 200) begin
                        scored_pipe1_flag <= 0;
                    end
                    
                    // Score for pipe 2
                    if ((pipe2_x + PIPE_WIDTH < 200) && !scored_pipe2_flag) begin
                        score <= score + 1;
                        if (score + 1 > high_score)
                            high_score <= score + 1;
                        scored_pipe2_flag <= 1;
                    end else if (pipe2_x + PIPE_WIDTH >= 200) begin
                        scored_pipe2_flag <= 0;
                    end
                end 
            end
        end
    end

    // Module instantiations
    pipe_controller pipe_inst (
        .clk(clk),
        .rst(rst),
        .game_clk(effective_game_clk),
        .pipe_reset(pipe_reset),
        .pipe_speed(pipe_speed),
        .pipe1_x(pipe1_x),
        .pipe1_gap_y(pipe1_gap_y),
        .pipe2_x(pipe2_x),
        .pipe2_gap_y(pipe2_gap_y)
    );

    drawcon drawcon_inst (
        .clk(pixclk),
        .rst(rst),
        .blkpos_x(blkpos_x),
        .blkpos_y(blkpos_y),
        .curr_x(curr_x),
        .curr_y(curr_y),
        .pipe1_x(pipe1_x),
        .pipe1_gap_y(pipe1_gap_y),
        .pipe2_x(pipe2_x),
        .pipe2_gap_y(pipe2_gap_y),
        .game_started(game_started),
        .game_over(game_over),
        .total_deaths(total_deaths),
        .died_by_pipe(died_by_pipe),
        .use_dead_sprite(use_dead_sprite),
        .use_dead_reflected_sprite(use_dead_reflected_sprite),
        .use_flap_down_sprite(use_flap_down_sprite),
        .hat_unlocked(hat_unlocked),
        .draw_r(draw_r),
        .draw_g(draw_g),
        .draw_b(draw_b),
        .score(score),
        .high_score(high_score)
    );

    vga vga_inst (
        .clk(pixclk),
        .rst(rst),
        .draw_r(draw_r),
        .draw_g(draw_g),
        .draw_b(draw_b),
        .curr_x(curr_x),
        .curr_y(curr_y),
        .pix_r(pix_r),
        .pix_g(pix_g),
        .pix_b(pix_b),
        .hsync(hsync),
        .vsync(vsync)
    );

    score_display score_display_inst (
        .clk(clk),
        .rst(rst),
        .score(score),
        .high_score(high_score),
        .seg(seg),
        .an(an)
    );

    clk_wiz_0 clk_inst (
        .clk_out1(pixclk),
        .clk_in1(clk)
    );

    accelerometer accel_inst (
            .clk(clk),
            .rst(rst),
            .miso(accel_miso),
            .mosi(accel_mosi),
            .sclk(accel_sclk),
            .cs(accel_cs),
            .hat_unlocked(hat_unlocked),
            .accel_x(accel_x),
            .accel_y(accel_y),
            .accel_mag(accel_mag)
        );

endmodule
