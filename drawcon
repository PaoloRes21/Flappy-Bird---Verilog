`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Module: drawcon.v
// Description:
//   - Renders the scrolling sky → grass → dirt → ground background
//   - Animates moving clouds with gradients and highlights
//   - Draws death-count tombstones on grass
//   - Overlays bird sprite in one of five states (idle, flap, rotated, dead, reflected)
//   - Renders two gradient-colored pipes with border lines
//   - Displays crown sprite when hat_unlocked
//   - Shows title/game-over/ID images from ROMs
//   - Renders "SCORE" and "HIGH SCORE" text bars
//////////////////////////////////////////////////////////////////////////////////

module drawcon(
    input clk, // System Clock
    input rst, // Active-Low Reset
    
    // Bird and World Position
    input [10:0] blkpos_x,
    input [10:0] blkpos_y,
    
    // Pipe Position
    input [10:0] pipe1_x,
    input [10:0] pipe1_gap_y,
    input [10:0] pipe2_x,
    input [10:0] pipe2_gap_y,
    
    // Game State
    input game_started,
    input game_over,
    input died_by_pipe,
    
    // Flappy Bird Gameover and Dead Sprites
    input use_dead_sprite,
    input use_dead_reflected_sprite,
    input use_flap_down_sprite,  
    
    // Pixel Scan Coordinates from VGA
    input [10:0] curr_x,
    input [10:0] curr_y,
    
    // Accelerometer Unlock Flag
    input hat_unlocked,
    
    // Scores and Deaths
    input [7:0] score,
    input [7:0] high_score,
    input [7:0] total_deaths,
    
    // Final RGB Outputs 4-bits each
    output [3:0] draw_r,
    output [3:0] draw_g,
    output [3:0] draw_b
);

reg [26:0] cloud_timer = 0;
reg [10:0] cloud_offset = 0;
wire [10:0] cloud_x = (curr_x + cloud_offset) % 1440;

// Moving Cloud Animation
always @(posedge clk) begin
    if (!rst) begin
        cloud_timer <= 0;
        cloud_offset <= 0;
    end else begin
        if (cloud_timer >= 25_000_000 - 1) begin 
            cloud_timer <= 0;
            cloud_offset <= (cloud_offset == 11'd1439) ? 0 : cloud_offset + 2;
        end else begin
            cloud_timer <= cloud_timer + 1;
        end
    end
end

// Screen shake timer and offset
reg [26:0] shake_timer = 0;
wire shake_active = (shake_timer < 27'd100_000_000); // 1 second
reg signed [3:0] shake_offset_x = 0;
reg signed [3:0] shake_offset_y = 0;

// Flash timer for ground flash
reg [26:0] flash_timer = 0;
wire flash_active = (flash_timer < 27'd100_000_000); // 1.0 second

// Smooth vertical gradient (static, clean, no lines) for the Sky
reg [7:0] top_r8 = 8'd20;    // Top color (deep sky blue)
reg [7:0] top_g8 = 8'd100;
reg [7:0] top_b8 = 8'd200;
reg [7:0] bot_r8 = 8'd120;   // Bottom color (light blue)
reg [7:0] bot_g8 = 8'd240;
reg [7:0] bot_b8 = 8'd255;
reg [15:0] blended_r16;
reg [15:0] blended_g16;
reg [15:0] blended_b16;
wire [15:0] weight_top = (720 - curr_y);
wire [15:0] weight_bot = curr_y;

// Flash and Shake when Bird Dies
always @(posedge clk) begin
    if (!rst) begin
        shake_timer <= 0;
        flash_timer <= 0;
    end else if (game_over) begin
        // SHAKE: 1 second (visible)
        if (shake_active) begin
            shake_timer <= shake_timer + 1;
            case (shake_timer[3:2])
                2'b00: begin shake_offset_x <=  5; shake_offset_y <= -3; end
                2'b01: begin shake_offset_x <= -5; shake_offset_y <=  4; end
                2'b10: begin shake_offset_x <=  4; shake_offset_y <=  3; end
                2'b11: begin shake_offset_x <= -4; shake_offset_y <= -4; end
            endcase
        end else begin
            shake_offset_x <= 0;
            shake_offset_y <= 0;
        end

        // FLASH: 1 second
        if (flash_timer < 27'd100_000_000)
            flash_timer <= flash_timer + 1;
    end else begin
        shake_timer <= 0;
        flash_timer <= 0;
        shake_offset_x <= 0;
        shake_offset_y <= 0;
    end
end

//////////////////////////////////////////////////////////
// 1. Background + Clouds + Score-Bar (Current + High Score)
//////////////////////////////////////////////////////////
reg [3:0] bg_r, bg_g, bg_b;
wire [10:0] sx = curr_x + shake_offset_x;
wire [10:0] sy = curr_y + shake_offset_y;
integer i;

always @* begin
    // --- Base background (sky → bright grass → dark grass → dirt → ground) ---
    if (sy > 11'd759) begin
            // GROUND FLASH (flashes red)
    if (flash_timer < 27'd100_000_000 && flash_timer[23]) begin
        bg_r = 4'b1111; bg_g = 4'b0000; bg_b = 4'b0000;
        end else begin
            bg_r = 4'b1111;  bg_g = 4'b1110;  bg_b = 4'b1011;
        end    
    end else if (sy > 11'd751) begin
        bg_r = 4'b1011;  bg_g = 4'b1010;  bg_b = 4'b1000; // dirt
    end else if (sy > 11'd745) begin        
        bg_r = 4'b0010;  bg_g = 4'b1001;  bg_b = 4'b0010; // dark grass
    end else if (sy > 11'd720) begin  
        bg_r = 4'b0011;  bg_g = 4'b1111;  bg_b = 4'b0011; // bright grass
    end 
    
    // Sky
    else begin
        // Weighted average using 16-bit intermediate for precision
        blended_r16 = (top_r8 * weight_top + bot_r8 * weight_bot) / 720;
        blended_g16 = (top_g8 * weight_top + bot_g8 * weight_bot) / 720;
        blended_b16 = (top_b8 * weight_top + bot_b8 * weight_bot) / 720;

        // Reduce to 4-bit VGA colors
        bg_r = blended_r16[7:4];
        bg_g = blended_g16[7:4];
        bg_b = blended_b16[7:4];
    end
    
    // --- Tombstone display at the top of the grass (visual death counter) ---    
    for (i = 0; i < total_deaths; i = i + 1) begin
        if (
            sy >= 709 && sy < 721 &&     // Above Grass area
            sx >= (20 + i*12) && sx < (28 + i*12)  // 8 pixels wide, spaced by 4
        ) begin
            // Gray tombstone color
            bg_r = 4'b1000; bg_g = 4'b1000; bg_b = 4'b1000;
        end
    end

    // --- Cloud overlays (with improved gradients and fluffy appearance) ---
    // Cloud 1 - Base shape with fluffy edges
    if (
        // Cloud 1 - Main body with gradient
        ((cloud_x>=306 && cloud_x<=356 && curr_y>=100 && curr_y<=109) ||
        (cloud_x>=299 && cloud_x<=368 && curr_y>=110 && curr_y<=119) ||
        (cloud_x>=285 && cloud_x<=380 && curr_y>=120 && curr_y<=129) ||
        (cloud_x>=270 && cloud_x<=380 && curr_y>=130 && curr_y<=139) ||
        (cloud_x>=298 && cloud_x<=400 && curr_y>=140 && curr_y<=149) ||
        (cloud_x>=260 && cloud_x<=420 && curr_y>=150 && curr_y<=159) ||
        (cloud_x>=232 && cloud_x<=435 && curr_y>=160 && curr_y<=169) ||
        (cloud_x>=275 && cloud_x<=350 && curr_y>=170 && curr_y<=179) ||
        (cloud_x>=366 && cloud_x<=380 && curr_y>=170 && curr_y<=179))
    ) begin
        // Subtle gradient - brighter at top, slightly darker at bottom
        if (curr_y < 130) begin
            // Top of cloud - brightest white
            bg_r = 4'b1111; bg_g = 4'b1111; bg_b = 4'b1111;
        end else if (curr_y < 150) begin
            // Middle - slightly less bright
            bg_r = 4'b1110; bg_g = 4'b1110; bg_b = 4'b1111;
        end else begin
            // Bottom - slightly gray
            bg_r = 4'b1101; bg_g = 4'b1101; bg_b = 4'b1110;
        end
    end
    
    // Cloud 1 - Fluffy highlights (extra white spots)
    if (
        (cloud_x>=300 && cloud_x<=315 && curr_y>=108 && curr_y<=115) ||
        (cloud_x>=330 && cloud_x<=345 && curr_y>=105 && curr_y<=112) ||
        (cloud_x>=280 && cloud_x<=295 && curr_y>=125 && curr_y<=135) ||
        (cloud_x>=360 && cloud_x<=375 && curr_y>=122 && curr_y<=132) ||
        (cloud_x>=250 && cloud_x<=265 && curr_y>=153 && curr_y<=163) ||
        (cloud_x>=400 && cloud_x<=412 && curr_y>=148 && curr_y<=158)
    ) begin
        // Extra bright highlights
        bg_r = 4'b1111; bg_g = 4'b1111; bg_b = 4'b1111;
    end
    
    // Cloud 1 - Shadow (adds depth)
    if (
        (cloud_x>=290 && cloud_x<=410 && curr_y>=173 && curr_y<=177)
    ) begin
        // Light shadow under cloud
        bg_r = 4'b0111; bg_g = 4'b1011; bg_b = 4'b1110;
    end

    // Cloud 2 with similar improvements
    if (
        // Cloud 2 - Main body with gradient
        ((cloud_x>=776 && cloud_x<=826 && curr_y>=310 && curr_y<=319) ||
        (cloud_x>=769 && cloud_x<=838 && curr_y>=320 && curr_y<=329) ||
        (cloud_x>=755 && cloud_x<=850 && curr_y>=330 && curr_y<=339) ||
        (cloud_x>=740 && cloud_x<=850 && curr_y>=340 && curr_y<=349) ||
        (cloud_x>=743 && cloud_x<=870 && curr_y>=350 && curr_y<=359) ||
        (cloud_x>=730 && cloud_x<=890 && curr_y>=360 && curr_y<=369) ||
        (cloud_x>=702 && cloud_x<=905 && curr_y>=370 && curr_y<=379) ||
        (cloud_x>=745 && cloud_x<=850 && curr_y>=380 && curr_y<=389) ||
        (cloud_x>=770 && cloud_x<=830 && curr_y>=380 && curr_y<=389))
    ) begin
        // Subtle gradient - brighter at top, slightly darker at bottom
        if (curr_y < 330) begin
            // Top of cloud - brightest white
            bg_r = 4'b1111; bg_g = 4'b1111; bg_b = 4'b1111;
        end else if (curr_y < 350) begin
            // Middle - slightly less bright
            bg_r = 4'b1110; bg_g = 4'b1110; bg_b = 4'b1111;
        end else begin
            // Bottom - slightly gray
            bg_r = 4'b1101; bg_g = 4'b1101; bg_b = 4'b1110;
        end
    end
    
    // Cloud 2 - Fluffy highlights
    if (
        (cloud_x>=775 && cloud_x<=790 && curr_y>=315 && curr_y<=325) ||
        (cloud_x>=810 && cloud_x<=822 && curr_y>=312 && curr_y<=320) ||
        (cloud_x>=760 && cloud_x<=775 && curr_y>=332 && curr_y<=342) ||
        (cloud_x>=840 && cloud_x<=855 && curr_y>=333 && curr_y<=343) ||
        (cloud_x>=730 && cloud_x<=745 && curr_y>=358 && curr_y<=368) ||
        (cloud_x>=880 && cloud_x<=895 && curr_y>=357 && curr_y<=367)
    ) begin
        // Extra bright highlights
        bg_r = 4'b1111; bg_g = 4'b1111; bg_b = 4'b1111;
    end
    
    // Cloud 2 - Shadow
    if (
        (cloud_x>=730 && cloud_x<=850 && curr_y>=383 && curr_y<=388)
    ) begin
        // Light shadow under cloud
        bg_r = 4'b0111; bg_g = 4'b1011; bg_b = 4'b1110;
    end

    // Cloud 3 with similar improvements
    if (
        // Cloud 3 - Main body with gradient
        ((cloud_x>=1206 && cloud_x<=1256 && curr_y>=210 && curr_y<=219) ||
        (cloud_x>=1199 && cloud_x<=1268 && curr_y>=220 && curr_y<=229) ||
        (cloud_x>=1185 && cloud_x<=1280 && curr_y>=230 && curr_y<=239) ||
        (cloud_x>=1170 && cloud_x<=1280 && curr_y>=240 && curr_y<=249) ||
        (cloud_x>=1155 && cloud_x<=1275 && curr_y>=250 && curr_y<=259) ||
        (cloud_x>=1160 && cloud_x<=1320 && curr_y>=260 && curr_y<=269) ||
        (cloud_x>=1132 && cloud_x<=1335 && curr_y>=270 && curr_y<=279) ||
        (cloud_x>=1175 && cloud_x<=1220 && curr_y>=280 && curr_y<=289) ||
        (cloud_x>=1240 && cloud_x<=1280 && curr_y>=280 && curr_y<=289))
    ) begin
        // Subtle gradient - brighter at top, slightly darker at bottom
        if (curr_y < 235) begin
            // Top of cloud - brightest white
            bg_r = 4'b1111; bg_g = 4'b1111; bg_b = 4'b1111;
        end else if (curr_y < 255) begin
            // Middle - slightly less bright
            bg_r = 4'b1110; bg_g = 4'b1110; bg_b = 4'b1111;
        end else begin
            // Bottom - slightly gray
            bg_r = 4'b1101; bg_g = 4'b1101; bg_b = 4'b1110;
        end
    end
    
    // Cloud 3 - Fluffy highlights
    if (
        (cloud_x>=1205 && cloud_x<=1220 && curr_y>=215 && curr_y<=225) ||
        (cloud_x>=1240 && cloud_x<=1252 && curr_y>=212 && curr_y<=222) ||
        (cloud_x>=1190 && cloud_x<=1205 && curr_y>=232 && curr_y<=242) ||
        (cloud_x>=1265 && cloud_x<=1278 && curr_y>=230 && curr_y<=240) ||
        (cloud_x>=1160 && cloud_x<=1175 && curr_y>=258 && curr_y<=268) ||
        (cloud_x>=1305 && cloud_x<=1320 && curr_y>=257 && curr_y<=267)
    ) begin
        // Extra bright highlights
        bg_r = 4'b1111; bg_g = 4'b1111; bg_b = 4'b1111;
    end
    
    // Cloud 3 - Shadow
    if (
        (cloud_x>=1170 && cloud_x<=1290 && curr_y>=283 && curr_y<=289)
    ) begin
        // Light shadow under cloud
        bg_r = 4'b0111; bg_g = 4'b1011; bg_b = 4'b1110;
    end

    // --- Small puffball clouds for added atmosphere ---
    
    // Small cloud 1 (left of cloud 1)
    if (
        (cloud_x>=180 && cloud_x<=230 && curr_y>=130 && curr_y<=150)
    ) begin
        if ((cloud_x-205)*(cloud_x-205) + (curr_y-140)*(curr_y-140) < 400) begin
            // Circular puff
            bg_r = 4'b1110; bg_g = 4'b1110; bg_b = 4'b1111;
        end
    end
    
    // Small cloud 2 (right of cloud 2)
    if (
        (cloud_x>=940 && cloud_x<=990 && curr_y>=340 && curr_y<=360)
    ) begin
        if ((cloud_x-965)*(cloud_x-965) + (curr_y-350)*(curr_y-350) < 400) begin
            // Circular puff
            bg_r = 4'b1110; bg_g = 4'b1110; bg_b = 4'b1111;
        end
    end
    
    // Small cloud 3 (between cloud 1 and 2)
    if (
        (cloud_x>=510 && cloud_x<=560 && curr_y>=200 && curr_y<=220)
    ) begin
        if ((cloud_x-535)*(cloud_x-535) + (curr_y-210)*(curr_y-210) < 400) begin
            // Circular puff
            bg_r = 4'b1110; bg_g = 4'b1110; bg_b = 4'b1111;
        end
    end

    // --- High-score overlay ---
    if (game_started &&
        sy >= 11'd770 && sy < 11'd810 &&
        sx >= 500               && sx < (500 + high_score * 12) &&
        ((sx - 500) % 12) < 10) begin
        bg_r = 4'b1111; bg_g = 4'b1000; bg_b = 4'b0000;
    end
    // --- Current-score overlay ---
    if (game_started &&
        sy >= 11'd770 && sy < 11'd810 &&
        sx >= 500           && sx < (500 + score * 12) &&
        ((sx - 500) % 12) < 10) begin
        bg_r = 4'b1011;  bg_g = 4'b0100;  bg_b = 4'b0000;
    end
end


//////////////////////////////////////////////////////////
// 2. Bird Sprite Drawing Logic
//////////////////////////////////////////////////////////
reg [3:0] bird_r, bird_g, bird_b;
reg [15:0] addr;

wire [11:0] rom_pixel_normal;
wire [11:0] rom_pixel_flap_down;
wire [11:0] rom_pixel_rotated;
wire [11:0] rom_pixel_dead;
wire [11:0] rom_pixel_dead_reflected;
wire [11:0] rom_pixel_crown;
wire [11:0] rom_pixel_title;

// Selection priority
wire [11:0] selected_pixel =
    use_dead_sprite ?
        (use_dead_reflected_sprite ? rom_pixel_dead_reflected : rom_pixel_dead) :
    (game_over && died_by_pipe) ? rom_pixel_rotated :
    (use_flap_down_sprite ? rom_pixel_flap_down : rom_pixel_normal);

always @(posedge clk) begin
    if (!rst) begin
        bird_r <= 0; bird_g <= 0; bird_b <= 0;
        addr <= 0;
    end else begin
        if (curr_x < blkpos_x || curr_x >= blkpos_x + 200 ||
            curr_y < blkpos_y || curr_y >= blkpos_y + 200) begin
            bird_r <= 0; bird_g <= 0; bird_b <= 0;
        end else begin
            bird_r <= selected_pixel[11:8];
            bird_g <= selected_pixel[7:4];
            bird_b <= selected_pixel[3:0];
            addr <= (curr_x == blkpos_x && curr_y == blkpos_y) ? 0 : addr + 1;
        end
    end
end

//////////////////////////////////////////////////////////
// 3. Pipe Drawing
//////////////////////////////////////////////////////////
localparam PIPE_WIDTH    = 80;
localparam PIPE_GAP_SIZE = 250;
localparam GROUND_Y      = 11'd721;

reg pipe_active;
reg [3:0] pipe_r, pipe_g, pipe_b;
reg [10:0] local_pipe_x;

always @* begin
    pipe_active = 1'b0;
    pipe_r = 4'd0; pipe_g = 4'd0; pipe_b = 4'd0;

    // First pipe
    if ((curr_x >= pipe1_x) && (curr_x < pipe1_x + PIPE_WIDTH)) begin
        if ((curr_y < pipe1_gap_y) || ((curr_y >= pipe1_gap_y + PIPE_GAP_SIZE) && (curr_y < GROUND_Y))) begin
            pipe_active = 1'b1;
            local_pipe_x = curr_x - pipe1_x;

            // Gradient body
            if (local_pipe_x < 6)
                pipe_g = 4'd15;
            else if (local_pipe_x < 16)
                pipe_g = 4'd12;
            else if (local_pipe_x < 21)
                pipe_g = 4'd14;
            else if (local_pipe_x < 61)
                pipe_g = 4'd12;
            else if (local_pipe_x < 71)
                pipe_g = 4'd14;
            else
                pipe_g = 4'd7;

            pipe_r = 4'd0;
            pipe_b = 4'd0;

            // Black borders
            if (local_pipe_x < 5 || local_pipe_x >= (PIPE_WIDTH - 5)) begin
                pipe_r = 0; pipe_g = 0; pipe_b = 0;
            end

            if (curr_y < pipe1_gap_y) begin
                if (curr_y < 5 || curr_y >= (pipe1_gap_y - 5)) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
                if ((pipe1_gap_y - curr_y) >= 30 && (pipe1_gap_y - curr_y) < 35) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
            end

            if (curr_y >= pipe1_gap_y + PIPE_GAP_SIZE) begin
                if (curr_y < (pipe1_gap_y + PIPE_GAP_SIZE + 5) || curr_y >= (GROUND_Y - 5)) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
                if ((curr_y - (pipe1_gap_y + PIPE_GAP_SIZE)) >= 30 &&
                    (curr_y - (pipe1_gap_y + PIPE_GAP_SIZE)) < 35) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
            end
        end
    end
    
    // Second pipe
    else if ((curr_x >= pipe2_x) && (curr_x < pipe2_x + PIPE_WIDTH)) begin
        if ((curr_y < pipe2_gap_y) || ((curr_y >= pipe2_gap_y + PIPE_GAP_SIZE) && (curr_y < GROUND_Y))) begin
            pipe_active = 1'b1;
            local_pipe_x = curr_x - pipe2_x;

            // Gradient body
            if (local_pipe_x < 6)
                pipe_g = 4'd15;
            else if (local_pipe_x < 16)
                pipe_g = 4'd12;
            else if (local_pipe_x < 21)
                pipe_g = 4'd14;
            else if (local_pipe_x < 61)
                pipe_g = 4'd12;
            else if (local_pipe_x < 71)
                pipe_g = 4'd14;
            else
                pipe_g = 4'd7;

            pipe_r = 4'd0;
            pipe_b = 4'd0;

            // Black borders
            if (local_pipe_x < 5 || local_pipe_x >= (PIPE_WIDTH - 5)) begin
                pipe_r = 0; pipe_g = 0; pipe_b = 0;
            end

            if (curr_y < pipe2_gap_y) begin
                if (curr_y < 5 || curr_y >= (pipe2_gap_y - 5)) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
                if ((pipe2_gap_y - curr_y) >= 30 && (pipe2_gap_y - curr_y) < 35) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
            end

            if (curr_y >= pipe2_gap_y + PIPE_GAP_SIZE) begin
                if (curr_y < (pipe2_gap_y + PIPE_GAP_SIZE + 5) || curr_y >= (GROUND_Y - 5)) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
                if ((curr_y - (pipe2_gap_y + PIPE_GAP_SIZE)) >= 30 &&
                    (curr_y - (pipe2_gap_y + PIPE_GAP_SIZE)) < 35) begin
                    pipe_r = 0; pipe_g = 0; pipe_b = 0;
                end
            end
        end
    end
end

//////////////////////////////////////////////////////////
// 5. Gameover Screen
//////////////////////////////////////////////////////////

// Game Over image coordinates
localparam GAMEOVER_X = 569;
localparam GAMEOVER_Y = 300;

// Determine if current pixel lies within the game over image area (200x200)
wire [9:0] go_img_x = curr_x - GAMEOVER_X;
wire [9:0] go_img_y = curr_y - GAMEOVER_Y;

wire show_gameover_image = game_over &&
                           curr_x >= GAMEOVER_X && curr_x < GAMEOVER_X + 300 &&
                           curr_y >= GAMEOVER_Y && curr_y < GAMEOVER_Y + 300;

// Address into the game over ROM
reg [17:0] go_addr;
always @(*) begin
    if (show_gameover_image)
        go_addr = go_img_y * 10'd300 + go_img_x;
    else
        go_addr = 0;
end

wire [11:0] rom_pixel_gameover;
wire [3:0] go_r = rom_pixel_gameover[11:8];
wire [3:0] go_g = rom_pixel_gameover[7:4];
wire [3:0] go_b = rom_pixel_gameover[3:0];

//////////////////////////////////////////////////////////
// 6. Unlocking the Crown
//////////////////////////////////////////////////////////
// Add inside bird sprite declarations
// Crown rendering logic for 50x50 sprite
reg [3:0] crown_r, crown_g, crown_b;
reg [11:0] crown_addr;

wire crown_visible = hat_unlocked && ~game_over &&
                     curr_x >= blkpos_x + 75 &&
                     curr_x <  blkpos_x + 125 &&
                     curr_y >= blkpos_y + 20 &&
                     curr_y <  blkpos_y + 70;

always @(posedge clk) begin
    if (!rst) begin
        crown_r <= 0; crown_g <= 0; crown_b <= 0;
        crown_addr <= 0;
    end else if (crown_visible) begin
        crown_r <= rom_pixel_crown[11:8];
        crown_g <= rom_pixel_crown[7:4];
        crown_b <= rom_pixel_crown[3:0];
        crown_addr <= (curr_y - blkpos_y - 20) * 6'd50 + (curr_x - (blkpos_x + 70));
    end else begin
        crown_r <= 0; crown_g <= 0; crown_b <= 0;
    end
end

//////////////////////////////////////////////////////////
// 4. Hard-coded "Score" and "High Score" text rendering
//////////////////////////////////////////////////////////
// Letter dimensions and positioning
localparam SCORE_TEXT_X = 200;        // Updated X position to 200
localparam SCORE_TEXT_Y = 770;        // Updated Y position to 770
localparam HIGH_SCORE_TEXT_Y = 810;   // High Score text positioned 10px below Score
localparam LETTER_HEIGHT = 30;        // Height of letters
localparam STROKE_WIDTH = 5;          // Thickness of the letter strokes
localparam LETTER_SPACING = 5;        // Space between letters

// Individual letter widths (excluding spacing)
localparam S_WIDTH = 20;
localparam C_WIDTH = 20;
localparam O_WIDTH = 25;
localparam R_WIDTH = 25;
localparam E_WIDTH = 20;
localparam H_WIDTH = 20;
localparam I_WIDTH = 10;
localparam G_WIDTH = 20;

// Letter positions for "SCORE" (with spacing)
localparam S_X = SCORE_TEXT_X;
localparam C_X = S_X + S_WIDTH + LETTER_SPACING;
localparam O_X = C_X + C_WIDTH + LETTER_SPACING;
localparam R_X = O_X + O_WIDTH + LETTER_SPACING;
localparam E_X = R_X + R_WIDTH + LETTER_SPACING;

// Letter positions for "HIGH SCORE" (with spacing)
localparam H1_X = SCORE_TEXT_X;
localparam I_X = H1_X + H_WIDTH + LETTER_SPACING;
localparam G_X = I_X + I_WIDTH + LETTER_SPACING;
localparam H2_X = G_X + G_WIDTH + LETTER_SPACING;
localparam HS_S_X = H2_X + H_WIDTH + LETTER_SPACING * 2; // Extra spacing after "HIGH"
localparam HS_C_X = HS_S_X + S_WIDTH + LETTER_SPACING;
localparam HS_O_X = HS_C_X + C_WIDTH + LETTER_SPACING;
localparam HS_R_X = HS_O_X + O_WIDTH + LETTER_SPACING;
localparam HS_E_X = HS_R_X + R_WIDTH + LETTER_SPACING;

// Score text drawing logic
reg score_text_active;
reg high_score_text_active; // Added missing high_score_text_active declaration
reg [3:0] score_text_r, score_text_g, score_text_b;
reg [3:0] high_score_text_r, high_score_text_g, high_score_text_b; // Added color variables for high score
reg [10:0] diagonal_x_min, diagonal_x_max;

always @* begin
    score_text_active = 1'b0;
    high_score_text_active = 1'b0;
    
    // Set default colors - use the colors from your score and high score blocks
    // Current score block color (reddish)
    score_text_r = 4'b1011; score_text_g = 4'b0100; score_text_b = 4'b0000;
    // High score block color (orange)
    high_score_text_r = 4'b1111; high_score_text_g = 4'b1000; high_score_text_b = 4'b0000;
    
    // Process "SCORE" text
    if (curr_y >= SCORE_TEXT_Y && curr_y < SCORE_TEXT_Y + LETTER_HEIGHT &&
        curr_x >= SCORE_TEXT_X && curr_x < E_X + E_WIDTH) begin
        
        // Letter 'S'
        if (curr_x >= S_X && curr_x < S_X + S_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= SCORE_TEXT_Y && curr_y < SCORE_TEXT_Y + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Middle horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                score_text_active = 1'b1;
            end
            // Bottom horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT) begin
                score_text_active = 1'b1;
            end
            // Left vertical stroke (top half)
            else if (curr_x >= S_X && curr_x < S_X + STROKE_WIDTH && 
                     curr_y >= SCORE_TEXT_Y && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT/2) begin
                score_text_active = 1'b1;
            end
            // Right vertical stroke (bottom half)
            else if (curr_x >= S_X + S_WIDTH - STROKE_WIDTH && curr_x < S_X + S_WIDTH && 
                     curr_y >= SCORE_TEXT_Y + LETTER_HEIGHT/2 && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT) begin
                score_text_active = 1'b1;
            end
        end
        
        // Letter 'C'
        else if (curr_x >= C_X && curr_x < C_X + C_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= SCORE_TEXT_Y && curr_y < SCORE_TEXT_Y + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Bottom horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT) begin
                score_text_active = 1'b1;
            end
            // Left vertical stroke
            else if (curr_x >= C_X && curr_x < C_X + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
        end
        
        // Letter 'O'
        else if (curr_x >= O_X && curr_x < O_X + O_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= SCORE_TEXT_Y && curr_y < SCORE_TEXT_Y + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Bottom horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT) begin
                score_text_active = 1'b1;
            end
            // Left vertical stroke
            else if (curr_x >= O_X && curr_x < O_X + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Right vertical stroke
            else if (curr_x >= O_X + O_WIDTH - STROKE_WIDTH && curr_x < O_X + O_WIDTH) begin
                score_text_active = 1'b1;
            end
        end
        
        // Letter 'R'
        else if (curr_x >= R_X && curr_x < R_X + R_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= SCORE_TEXT_Y && curr_y < SCORE_TEXT_Y + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Middle horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                score_text_active = 1'b1;
            end
            // Left vertical stroke
            else if (curr_x >= R_X && curr_x < R_X + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Right vertical stroke (top half)
            else if (curr_x >= R_X + R_WIDTH - STROKE_WIDTH && curr_x < R_X + R_WIDTH && 
                     curr_y >= SCORE_TEXT_Y && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT/2) begin
                score_text_active = 1'b1;
            end
            // Diagonal stroke (bottom half)
            else if (curr_y > SCORE_TEXT_Y + LETTER_HEIGHT/2 &&
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT) begin
                // Create a diagonal by using a line equation
                
                diagonal_x_min = R_X + STROKE_WIDTH + 
                                (curr_y - (SCORE_TEXT_Y + LETTER_HEIGHT/2)) * 
                                (R_WIDTH - 2*STROKE_WIDTH) / (LETTER_HEIGHT/2);
                diagonal_x_max = diagonal_x_min + STROKE_WIDTH;
                
                if (curr_x >= diagonal_x_min && curr_x < diagonal_x_max) begin
                    score_text_active = 1'b1;
                end
            end
        end
        
        // Letter 'E'
        else if (curr_x >= E_X && curr_x < E_X + E_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= SCORE_TEXT_Y && curr_y < SCORE_TEXT_Y + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
            // Middle horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                score_text_active = 1'b1;
            end
            // Bottom horizontal stroke
            else if (curr_y >= SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < SCORE_TEXT_Y + LETTER_HEIGHT) begin
                score_text_active = 1'b1;
            end
            // Left vertical stroke
            else if (curr_x >= E_X && curr_x < E_X + STROKE_WIDTH) begin
                score_text_active = 1'b1;
            end
        end
    end
    
    // Process "HIGH SCORE" text
    else if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT &&
             curr_x >= SCORE_TEXT_X && curr_x < HS_E_X + E_WIDTH) begin
        
        // Letter 'H'
        if (curr_x >= H1_X && curr_x < H1_X + H_WIDTH) begin
            // Middle horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                curr_y < HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= H1_X && curr_x < H1_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Right vertical stroke
            else if (curr_x >= H1_X + H_WIDTH - STROKE_WIDTH && curr_x < H1_X + H_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Letter 'I'
        else if (curr_x >= I_X && curr_x < I_X + I_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Bottom horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Vertical stroke (middle)
            else if (curr_x >= I_X + (I_WIDTH/2 - STROKE_WIDTH/2) && 
                     curr_x < I_X + (I_WIDTH/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Letter 'G'
        else if (curr_x >= G_X && curr_x < G_X + G_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Bottom horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= G_X && curr_x < G_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Right vertical stroke (bottom half)
            else if (curr_x >= G_X + G_WIDTH - STROKE_WIDTH && curr_x < G_X + G_WIDTH && 
                     curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT/2 && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Middle-right horizontal stroke
            else if (curr_x >= G_X + G_WIDTH/2 && curr_x < G_X + G_WIDTH && 
                     curr_y >= HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Second Letter 'H'
        else if (curr_x >= H2_X && curr_x < H2_X + H_WIDTH) begin
            // Middle horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                curr_y < HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= H2_X && curr_x < H2_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Right vertical stroke
            else if (curr_x >= H2_X + H_WIDTH - STROKE_WIDTH && curr_x < H2_X + H_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Letter 'S' in "SCORE"
        else if (curr_x >= HS_S_X && curr_x < HS_S_X + S_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Middle horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Bottom horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke (top half)
            else if (curr_x >= HS_S_X && curr_x < HS_S_X + STROKE_WIDTH && 
                     curr_y >= HIGH_SCORE_TEXT_Y && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT/2) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Right vertical stroke (bottom half)
            else if (curr_x >= HS_S_X + S_WIDTH - STROKE_WIDTH && curr_x < HS_S_X + S_WIDTH && 
                     curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT/2 && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Letter 'C' in "SCORE"
        else if (curr_x >= HS_C_X && curr_x < HS_C_X + C_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Bottom horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= HS_C_X && curr_x < HS_C_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Letter 'O' in "SCORE"
        else if (curr_x >= HS_O_X && curr_x < HS_O_X + O_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Bottom horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= HS_O_X && curr_x < HS_O_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Right vertical stroke
            else if (curr_x >= HS_O_X + O_WIDTH - STROKE_WIDTH && curr_x < HS_O_X + O_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
        
        // Letter 'R' in "SCORE"
        else if (curr_x >= HS_R_X && curr_x < HS_R_X + R_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Middle horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= HS_R_X && curr_x < HS_R_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Right vertical stroke (top half)
            else if (curr_x >= HS_R_X + R_WIDTH - STROKE_WIDTH && curr_x < HS_R_X + R_WIDTH && 
                     curr_y >= HIGH_SCORE_TEXT_Y && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT/2) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Diagonal stroke (bottom half)
            else if (curr_y > HIGH_SCORE_TEXT_Y + LETTER_HEIGHT/2 &&
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                // Create a diagonal by using a line equation
                
                diagonal_x_min = HS_R_X + STROKE_WIDTH + 
                                (curr_y - (HIGH_SCORE_TEXT_Y + LETTER_HEIGHT/2)) * 
                                (R_WIDTH - 2*STROKE_WIDTH) / (LETTER_HEIGHT/2);
                diagonal_x_max = diagonal_x_min + STROKE_WIDTH;
                
                if (curr_x >= diagonal_x_min && curr_x < diagonal_x_max) begin
                    high_score_text_active = 1'b1; // Changed to high_score_text_active
                end
            end
        end
        
        // Letter 'E' in "SCORE"
        else if (curr_x >= HS_E_X && curr_x < HS_E_X + E_WIDTH) begin
            // Top horizontal stroke
            if (curr_y >= HIGH_SCORE_TEXT_Y && curr_y < HIGH_SCORE_TEXT_Y + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Middle horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 - STROKE_WIDTH/2) && 
                     curr_y < HIGH_SCORE_TEXT_Y + (LETTER_HEIGHT/2 + STROKE_WIDTH/2)) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Bottom horizontal stroke
            else if (curr_y >= HIGH_SCORE_TEXT_Y + LETTER_HEIGHT - STROKE_WIDTH && 
                     curr_y < HIGH_SCORE_TEXT_Y + LETTER_HEIGHT) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
            // Left vertical stroke
            else if (curr_x >= HS_E_X && curr_x < HS_E_X + STROKE_WIDTH) begin
                high_score_text_active = 1'b1; // Changed to high_score_text_active
            end
        end
    end
    
    // No need to change colors here - we already set the defaults at the beginning
    // We're keeping the colors as you defined, matching the score and high score blocks
end

//////////////////////////////////////////////////////////
// Title Page
//////////////////////////////////////////////////////////

localparam TITLE_X = 569;
localparam TITLE_Y = 300;
localparam TITLE_W = 300;
localparam TITLE_H = 300;

wire show_title = ~game_started && 
                  curr_x >= TITLE_X && curr_x < TITLE_X + 300 &&
                  curr_y >= TITLE_Y && curr_y < TITLE_Y + 300;

wire [9:0] title_img_x = curr_x - TITLE_X;
wire [9:0] title_img_y = curr_y - TITLE_Y;

reg [17:0] title_addr;
always @(*) begin
    if (show_title)
        title_addr = title_img_y * 10'd300 + title_img_x;
    else
        title_addr = 0;
end

wire [3:0] title_r = rom_pixel_title[11:8];
wire [3:0] title_g = rom_pixel_title[7:4];
wire [3:0] title_b = rom_pixel_title[3:0];

//////////////////////////////////////////////////////////
// To Display ID's
////////////////////////////////////////////////////////////
wire [11:0] rom_pixel_id; // 12-bit color output from ROM
reg [13:0] id_addr;       // For 78x38 = 2964 entries

localparam ID_WIDTH  = 80;
localparam ID_HEIGHT = 50;
localparam ID_X = 50;
localparam ID_Y = 770;

wire show_id_sprite = (curr_x >= ID_X && curr_x < ID_X + ID_WIDTH &&
                       curr_y >= ID_Y && curr_y < ID_Y + ID_HEIGHT);

wire [11:0] id_img_x = curr_x - ID_X;
wire [11:0] id_img_y = curr_y - ID_Y;

always @(*) begin
    if (show_id_sprite)
        id_addr = id_img_y * ID_WIDTH + id_img_x;
    else
        id_addr = 0;
end

wire [3:0] id_r = rom_pixel_id[11:8];
wire [3:0] id_g = rom_pixel_id[7:4];
wire [3:0] id_b = rom_pixel_id[3:0];

//////

//localparam AKEEL_ID_WIDTH  = 99; // adjust based on actual width
//localparam AKEEL_ID_HEIGHT = 21;  // adjust based on actual height
//localparam AKEEL_ID_X = 50;
//localparam AKEEL_ID_Y = ID_Y + 100;  // 100 pixels lower

//wire show_akeel_id_sprite = (curr_x >= AKEEL_ID_X && curr_x < AKEEL_ID_X + AKEEL_ID_WIDTH &&
//                             curr_y >= AKEEL_ID_Y && curr_y < AKEEL_ID_Y + AKEEL_ID_HEIGHT);

//wire [10:0] akeel_img_x = curr_x - AKEEL_ID_X;
//wire [10:0] akeel_img_y = curr_y - AKEEL_ID_Y;

//reg [13:0] akeel_addr;
//always @(*) begin
//    if (show_akeel_id_sprite)
//        akeel_addr = akeel_img_y * AKEEL_ID_WIDTH + akeel_img_x;
//    else
//        akeel_addr = 0;
//end

//wire [11:0] rom_pixel_akeel;
//wire [3:0] akeel_r = rom_pixel_akeel[11:8];
//wire [3:0] akeel_g = rom_pixel_akeel[7:4];
//wire [3:0] akeel_b = rom_pixel_akeel[3:0];

//// To display score in the VGA display
//reg [9:0] font [0:9];
//initial begin
//    font[0] = 10'b01111110; // 0
//    font[1] = 10'b00110000; // 1
//    font[2] = 10'b01101101; // 2
//    font[3] = 10'b01111001; // 3
//    font[4] = 10'b00110011; // 4
//    font[5] = 10'b01011011; // 5
//    font[6] = 10'b01011111; // 6
//    font[7] = 10'b01110000; // 7
//    font[8] = 10'b01111111; // 8
//    font[9] = 10'b01111011; // 9
//end

//////////////////////////////////////////////////////////
// Text Drawing Logic for score and high score
////////////////////////////////////////////////////////////
//wire draw_score_text;
//reg [3:0] score_r, score_g, score_b;

//// Set position and digit area (8x10 per digit)
//localparam SCORE_X = 11'd800;
//localparam SCORE_Y = 11'd770;

//// Break score into digits
//wire [3:0] score_tens = score / 10;
//wire [3:0] score_ones = score % 10;
//wire [3:0] high_tens = high_score / 10;
//wire [3:0] high_ones = high_score % 10;

//reg is_digit_pixel;

//always @(*) begin
//    is_digit_pixel = 0;
//    score_r = 0;
//    score_g = 0;
//    score_b = 0;

//    // Display "Score" at SCORE_X, SCORE_Y
//    if (curr_y >= SCORE_Y && curr_y < SCORE_Y + 10) begin
//        // Display current score
//        if (curr_x >= SCORE_X && curr_x < SCORE_X + 8) begin // tens
//            if (font[score_tens][curr_y - SCORE_Y] & (1 << (7 - (curr_x - SCORE_X))))
//                is_digit_pixel = 1;
//        end else if (curr_x >= SCORE_X + 10 && curr_x < SCORE_X + 18) begin // ones
//            if (font[score_ones][curr_y - SCORE_Y] & (1 << (7 - (curr_x - (SCORE_X + 10)))))
//                is_digit_pixel = 1;
//        end

//        // Display high score below
//        else if (curr_x >= SCORE_X && curr_x < SCORE_X + 8 && curr_y >= SCORE_Y + 12 && curr_y < SCORE_Y + 22) begin
//            if (font[high_tens][curr_y - (SCORE_Y + 12)] & (1 << (7 - (curr_x - SCORE_X))))
//                is_digit_pixel = 1;
//        end else if (curr_x >= SCORE_X + 10 && curr_x < SCORE_X + 18 && curr_y >= SCORE_Y + 12 && curr_y < SCORE_Y + 22) begin
//            if (font[high_ones][curr_y - (SCORE_Y + 12)] & (1 << (7 - (curr_x - (SCORE_X + 10)))))
//                is_digit_pixel = 1;
//        end
//    end

//    if (is_digit_pixel) begin
//        score_r = 4'b1111;
//        score_g = 4'b1111;
//        score_b = 0;
//    end
//end

// Modify final draw assignment
assign draw_r = (high_score_text_active ? high_score_text_r :
                (score_text_active ? score_text_r :
                (show_title && rom_pixel_title != 12'h000 ? title_r :
                (show_gameover_image && rom_pixel_gameover != 12'h000 ? go_r :
                (crown_r != 0 ? crown_r :
                (show_id_sprite && rom_pixel_id != 12'h000 ? id_r :
                (bird_r != 0 ? bird_r : (pipe_active ? pipe_r : bg_r))))))));

assign draw_g = (high_score_text_active ? high_score_text_g :
                (score_text_active ? score_text_g :
                (show_title && rom_pixel_title != 12'h000 ? title_g :
                (show_gameover_image && rom_pixel_gameover != 12'h000 ? go_g :
                (crown_g != 0 ? crown_g :
                (show_id_sprite && rom_pixel_id != 12'h000 ? id_g :
                (bird_g != 0 ? bird_g : (pipe_active ? pipe_g : bg_g))))))));

assign draw_b = (high_score_text_active ? high_score_text_b :
                (score_text_active ? score_text_b :
                (show_title && rom_pixel_title != 12'h000 ? title_b :
                (show_gameover_image && rom_pixel_gameover != 12'h000 ? go_b :
                (crown_b != 0 ? crown_b :
                (show_id_sprite && rom_pixel_id != 12'h000 ? id_b :
                (bird_b != 0 ? bird_b : (pipe_active ? pipe_b : bg_b))))))));

//////////////////////////////////////////////////////////
// 7. ROM Instantiations
//////////////////////////////////////////////////////////
blk_mem_gen_0 bird_mem_inst (
    .clka(clk),
    .addra(addr),
    .douta(rom_pixel_normal)
);

blk_mem_gen_flap_down flap_down_inst (
    .clka(clk),
    .addra(addr),
    .douta(rom_pixel_flap_down)
);

blk_mem_gen_rotated rotated_bird_mem_inst (
    .clka(clk),
    .addra(addr),
    .douta(rom_pixel_rotated)
);

blk_mem_gen_dead dead_bird_mem_inst (
    .clka(clk),
    .addra(addr),
    .douta(rom_pixel_dead)
);

blk_mem_gen_dead_reflected dead_reflected_inst (
    .clka(clk),
    .addra(addr),
    .douta(rom_pixel_dead_reflected)
);

blk_mem_gen_gameover gameover_inst (
    .clka(clk),
    .addra(go_addr),
    .douta(rom_pixel_gameover)
);

blk_mem_gen_crown crown_inst (
    .clka(clk),
    .addra(crown_addr),
    .douta(rom_pixel_crown)
);

blk_mem_gen_title title_rom_inst (
    .clka(clk),
    .addra(title_addr),
    .douta(rom_pixel_title)
);

blk_mem_gen_id id_display_inst (
    .clka(clk),
    .addra(id_addr),
    .douta(rom_pixel_id)
);

//blk_mem_gen_akeel akeel_display_inst (
//    .clka(clk),
//    .addra(akeel_addr),
//    .douta(rom_pixel_akeel)
//);
endmodule
