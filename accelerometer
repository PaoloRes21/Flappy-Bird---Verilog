`timescale 1ns / 1ps
//////////////////////////////////////////////////////////////////////////////////
// Module: accelerometer.v
// Description:
//   - Generates ~1 MHz SPI clock by dividing 100 MHz
//   - Samples x/y/z from ADXL362 every ~40 ms via state machine:
//       IDLE → START_SAMPLE → INIT_SPI → SEND_ADDR → READ_DATA → WAIT_SAMPLE
//   - On READ_DATA complete: compute absolute diffs, sum them, set hat_unlocked if > threshold
//   - Exposes raw accel_x/y and movement magnitude for debugging
//////////////////////////////////////////////////////////////////////////////////

module accelerometer(
    input clk,                   // System clock
    input rst,                   // Reset signal (active low)
    
    // SPI Interface to ADXL362 accelerometer
    input miso,                  // Master In Slave Out (data from accelerometer)
    output reg mosi,             // Master Out Slave In (data to accelerometer)
    output reg sclk,             // SPI Clock
    output reg cs,               // Chip Select
    
    // Output signals
    output reg hat_unlocked,     // Signal when movement exceeds threshold
    
    // For debugging
    output [8:0] accel_x,        // X acceleration value
    output [8:0] accel_y,        // Y acceleration value
    output [11:0] accel_mag      // Movement magnitude
);

    // State machine states
    localparam IDLE          = 4'd0;
    localparam START_SAMPLE  = 4'd1;
    localparam INIT_SPI      = 4'd2;
    localparam SEND_ADDR     = 4'd3;
    localparam READ_DATA     = 4'd4;
    localparam WAIT_SAMPLE   = 4'd5;
    
    // SPI interface constants
    localparam READ_CMD      = 8'h0B;   // ADXL362 read command
    localparam XDATA_L_ADDR  = 8'h0E;   // ADXL362 X-axis data low byte register
    
    // Timing parameters
    localparam SPI_CLK_DIV   = 16'd50;  // For ~1MHz SPI clock from 100MHz
    localparam SAMPLE_PERIOD = 24'd2500000; // ~40ms between samples at 100MHz
    
    // Threshold for movement detection
    localparam MOVEMENT_THRESHOLD = 12'd512;
    
    // Registers
    reg [3:0] state;
    reg [7:0] byte_counter;
    reg [2:0] bit_counter;
    reg [7:0] tx_data;
    reg [7:0] rx_data;
    
    reg [15:0] spi_clk_counter;
    reg spi_clk_enable;
    
    reg [23:0] sample_timer;
    
    reg [8:0] acc_x_reg;
    reg [8:0] acc_y_reg;
    reg [8:0] acc_z_reg;
    
    reg [8:0] prev_acc_x;
    reg [8:0] prev_acc_y;
    reg [8:0] prev_acc_z;
    
    reg [11:0] diff_x;
    reg [11:0] diff_y;
    reg [11:0] diff_z;
    reg [11:0] movement_magnitude;
    
    // SPI clock generation
    always @(posedge clk) begin
        if (!rst) begin
            spi_clk_counter <= 0;
            spi_clk_enable <= 0;
        end else begin
            if (spi_clk_counter >= SPI_CLK_DIV-1) begin
                spi_clk_counter <= 0;
                spi_clk_enable <= 1;
            end else begin
                spi_clk_counter <= spi_clk_counter + 1;
                spi_clk_enable <= 0;
            end
        end
    end
    
    // Sample timer for periodic readings
    always @(posedge clk) begin
        if (!rst) begin
            sample_timer <= 0;
        end else begin
            if (state == WAIT_SAMPLE) begin
                if (sample_timer >= SAMPLE_PERIOD-1)
                    sample_timer <= 0;
                else
                    sample_timer <= sample_timer + 1;
            end else begin
                sample_timer <= 0;
            end
        end
    end
    
    // Main state machine
    always @(posedge clk) begin
        if (!rst) begin
            state <= IDLE;
            cs <= 1;
            sclk <= 1;
            mosi <= 0;
            byte_counter <= 0;
            bit_counter <= 3'b111;
            tx_data <= 0;
            rx_data <= 0;
            acc_x_reg <= 9'h100;  // Center position
            acc_y_reg <= 9'h100;  // Center position
            acc_z_reg <= 9'h100;  // Center position
            prev_acc_x <= 9'h100; // Center position
            prev_acc_y <= 9'h100; // Center position
            prev_acc_z <= 9'h100; // Center position
            hat_unlocked <= 0;
        end else begin
            case (state)
                IDLE: begin
                    cs <= 1;
                    sclk <= 1;
                    mosi <= 0;
                    byte_counter <= 0;
                    state <= START_SAMPLE;
                end
                
                START_SAMPLE: begin
                    // Store previous values
                    prev_acc_x <= acc_x_reg;
                    prev_acc_y <= acc_y_reg;
                    prev_acc_z <= acc_z_reg;
                    
                    // Start SPI transaction
                    state <= INIT_SPI;
                end
                
                INIT_SPI: begin
                    cs <= 0;  // Assert chip select (active low)
                    tx_data <= READ_CMD;  // Read command
                    bit_counter <= 3'b111;
                    state <= SEND_ADDR;
                end
                
                SEND_ADDR: begin
                    if (spi_clk_enable) begin
                        if (sclk == 1) begin
                            sclk <= 0;
                            mosi <= tx_data[bit_counter];
                        end else begin
                            sclk <= 1;
                            if (bit_counter == 3'b000) begin
                                bit_counter <= 3'b111;
                                byte_counter <= byte_counter + 1;
                                
                                // Prepare next byte to send
                                case (byte_counter)
                                    0: tx_data <= XDATA_L_ADDR;  // X-axis low byte address
                                    default: tx_data <= 8'h00;   // Dummy bytes for reading
                                endcase
                                
                                if (byte_counter == 1) 
                                    state <= READ_DATA;
                            end else begin
                                bit_counter <= bit_counter - 1;
                            end
                        end
                    end
                end
                
                READ_DATA: begin
                    if (spi_clk_enable) begin
                        if (sclk == 1) begin
                            sclk <= 0;
                            mosi <= 0;  // We're just reading now
                        end else begin
                            sclk <= 1;
                            rx_data[bit_counter] <= miso;  // Capture data bit
                            
                            if (bit_counter == 3'b000) begin
                                bit_counter <= 3'b111;
                                
                                // Store received byte based on byte counter
                                case (byte_counter)
                                    2: acc_x_reg[7:0] <= rx_data;    // X-axis low byte
                                    3: acc_x_reg[8] <= rx_data[0];   // X-axis high byte (only need bit 0)
                                    4: acc_y_reg[7:0] <= rx_data;    // Y-axis low byte
                                    5: acc_y_reg[8] <= rx_data[0];   // Y-axis high byte
                                    6: acc_z_reg[7:0] <= rx_data;    // Z-axis low byte
                                    7: acc_z_reg[8] <= rx_data[0];   // Z-axis high byte
                                endcase
                                
                                byte_counter <= byte_counter + 1;
                                
                                if (byte_counter >= 7) begin
                                    // We've read all the data we need
                                    cs <= 1;  // Deselect the device
                                    
                                    // Calculate absolute differences
                                    diff_x <= (acc_x_reg > prev_acc_x) ? 
                                                (acc_x_reg - prev_acc_x) : 
                                                (prev_acc_x - acc_x_reg);
                                    diff_y <= (acc_y_reg > prev_acc_y) ? 
                                                (acc_y_reg - prev_acc_y) : 
                                                (prev_acc_y - acc_y_reg);
                                    diff_z <= (acc_z_reg > prev_acc_z) ? 
                                                (acc_z_reg - prev_acc_z) : 
                                                (prev_acc_z - acc_z_reg);
                                        
                                    // Simple magnitude calculation (sum of absolute differences)
                                    // Using sum instead of sqrt for simplicity and resource saving
                                    movement_magnitude <= diff_x + diff_y + diff_z;
                                    
                                    // Unlock hat if movement exceeds threshold
                                    if ((diff_x + diff_y + diff_z) > MOVEMENT_THRESHOLD)
                                        hat_unlocked <= 1;
                                        
                                    state <= WAIT_SAMPLE;
                                end
                            end else begin
                                bit_counter <= bit_counter - 1;
                            end
                        end
                    end
                end
                
                WAIT_SAMPLE: begin
                    // Wait for sample timer to expire before next sample
                    if (sample_timer >= SAMPLE_PERIOD-1)
                        state <= START_SAMPLE;
                end
                
                default: state <= IDLE;
            endcase
        end
    end
    
    // Export acceleration values for debugging
    assign accel_x = acc_x_reg;
    assign accel_y = acc_y_reg;
    assign accel_mag = movement_magnitude;

endmodule
