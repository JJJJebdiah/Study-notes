# Verilog 学习笔记:流水线

#### **背景**

在设计 HDMI 项目时，我在实现 TMDS 编码算法的过程中遇到了一些困难。TMDS（Transition Minimized Differential Signaling）是一种高速串行传输协议，用于将视频、音频和控制信号编码成串行数据流。TMDS 编码需要处理大量的数据流，包括对输入数据进行逐字节的复杂逻辑处理，同时保证输出的时序对齐。

由于 TMDS 编码的过程涉及多级逻辑操作（如平衡位计算、差分编码和数据流对齐），我决定采用流水线结构来实现这一算法。然而，在设计过程中，我发现流水线的信号对齐、阶段划分和变量传递逻辑比预想的复杂。这促使我深入学习和总结流水线设计的关键点和实践方法。

## 1.流水线的基本原理

- **分阶段处理**： 将一个复杂操作分解为多个简单的阶段，每个阶段在一个时钟周期内完成。

- **寄存器隔离**： 在每个阶段之间使用寄存器存储中间结果，确保数据传递的同步性。

- **并行处理**： 不同阶段可以同时处理不同的数据，提高吞吐量。

## 2.流水线结构设计标准

1.**模块化设计**：

- 将每一级流水线封装为独立逻辑，分离控制信号和数据处理逻辑。

2.**寄存器分割**：

- 使用寄存器在每一级之间存储数据，确保时序和信号对齐。

3.**清晰的数据流**：

- 明确每一级流水线的输入、处理和输出，尽量避免跨级依赖。

4.**控制信号同步**：

- 确保控制信号（如有效位、停止位等）与数据同步传递。

5.**复位逻辑**：

- 添加复位逻辑清零寄存器，避免仿真或实际运行时的初始状态不一致。

## 3.流水线结构模板

以下是一个典型的流水线设计模板，假设数据经过 3 个处理阶段（Stage 1、Stage 2、Stage 3）：

```verilog
module pipeline_example (
    input wire clk,              // System clock
    input wire rst_n,            // Active-low reset
    input wire [31:0] data_in,   // Input data
    output reg [31:0] data_out   // Output data
);

    // Pipeline stage registers
    reg [31:0] stage1_reg, stage2_reg, stage3_reg;

    // Stage 1: Process input data
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            stage1_reg <= 32'b0;  // Reset stage 1 register
        end else begin
            stage1_reg <= data_in + 1; // Example operation
        end
    end

    // Stage 2: Process stage1 output
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            stage2_reg <= 32'b0;  // Reset stage 2 register
        end else begin
            stage2_reg <= stage1_reg * 2; // Example operation
        end
    end

    // Stage 3: Process stage2 output
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            stage3_reg <= 32'b0;  // Reset stage 3 register
        end else begin
            stage3_reg <= stage2_reg - 5; // Example operation
        end
    end

    // Output data from the final stage
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            data_out <= 32'b0;  // Reset output
        end else begin
            data_out <= stage3_reg;
        end
    end

endmodule

```

### 模板分解与说明

1.**时钟与复位**：

- 每个阶段用 `always @(posedge clk or negedge rst_n)` 实现。
- 在复位时清零所有寄存器，确保流水线处于初始状态。

2.**寄存器定义**：

- 每一级用寄存器存储中间结果（例如 `stage1_reg`, `stage2_reg`, `stage3_reg`）。

**3.逐级处理**：

- 每一级对上一阶段的输出进行处理。
- 逻辑清晰，无跨级依赖。

**4.最终输出**：

- 最后一阶段的结果传递到输出 `data_out`。

## 4.流水线设计的关键点

### 4.1单阶段单段设计

- 在简单设计中，每个阶段的所有逻辑集中在一个 `always` 块中完成，包括数据处理和传递。

- **优点**：逻辑集中，代码简单，适合较小规模的流水线。

- **缺点**：当阶段逻辑复杂时，单块设计可能显得冗长。

**示例**：

```verilog
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        stage1_data <= 0;
        stage2_data <= 0;
    end else begin
        stage1_data <= data_in + 1;
        stage2_data <= stage1_data; 
    end
end

```



### 4.2单阶段多块设计

- 将一个阶段的逻辑拆分为多个 `always` 块，比如数据处理和控制信号分别处理。

- **优点**：逻辑解耦，代码可读性更高，适合复杂设计。

- **缺点**：需要注意时序一致性，避免分块设计引入错误。

**示例**：

```verilog
// Data processing
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        stage1_data <= 0;
    else
        stage1_data <= data_in + 1;
end

// Pass through control signal
always @(posedge clk or negedge rst_n) begin
    if (!rst_n)
        valid_flag <= 0;
    else
        valid_flag <= input_valid;
end

```

### 4.3信号传递的管理

#### 4.3.1逐级传递

即信号随着流水线逐级传递，在每个阶段的寄存器中存储，即使该阶段暂时不使用该变量。

**优点**：

1.**时序对齐明确**：

- 输入变量在每一阶段都同步到当前时钟周期，保证与其他变量的对齐一致性。

2.**灵活性高**：

- 如果中间阶段可能需要使用该变量，直接访问即可，避免后续修改代码时重新增加逻辑。

3.**易于调试**：

- 每一级寄存器均保存了该变量的状态，方便查看时序波形。

```verilog
module pipeline_example (
    input wire clk,
    input wire rst_n,
    input wire [15:0] input_signal, // Input signal used in Stage 3
    output reg [15:0] output_signal
);

    // Stage registers for the input signal
    reg [15:0] stage1_signal, stage2_signal, stage3_signal;

    // Stage 1: Pass input_signal to stage1 register
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            stage1_signal <= 16'b0;
        end else begin
            stage1_signal <= input_signal;
        end
    end

    // Stage 2: Pass stage1_signal to stage2 register
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            stage2_signal <= 16'b0;
        end else begin
            stage2_signal <= stage1_signal;
        end
    end

    // Stage 3: Use stage2_signal
    always @(posedge clk or negedge rst_n) begin
        if (!rst_n) begin
            stage3_signal <= 16'b0;
            output_signal <= 16'b0;
        end else begin
            stage3_signal <= stage2_signal;
            output_signal <= stage3_signal + 1; // Example usage
        end
    end

endmodule

```

#### 4.3.2传递与使用分离

使用单独的`always`块传递数信号

**优点**

1. **逻辑分离**：
   - 将不相关的信号逻辑分离到独立的 `always` 块中，可以避免将无关的操作混在主逻辑中，保持代码清晰。
2. **减少干扰**：
   - 对于本阶段暂时不使用的信号，单独处理可以避免无意中修改信号值或与其他逻辑耦合。
3. **时序一致性**：
   - 即使信号不参与本阶段逻辑，也可以保证它被正确地传递到下一级，维持流水线的时序一致性。
4. **硬件实现**：
   - 每个 `always` 块对应独立的逻辑硬件，分离的 `always` 块不会影响硬件实现效率。

**示例**

```verilog
// Stage 1: Pass-through signals
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        stage1_signal1 <= 0;
        stage1_signal2 <= 0;
    end else begin
        stage1_signal1 <= input_signal1;
        stage1_signal2 <= input_signal2;
    end
end

```

### 4.4信号命名

流水线信号的命名规范对于代码的**可读性**、**维护性**以及团队协作至关重要。

1. **明确信号的作用**

   信号名称应清晰反映信号的**功能**和**作用**，如表示数据、控制信号或有效信号。

   - 数据信号：`data`, `result`, `sum`。
   - 控制信号：`valid`, `ready`, `enable`。
   - 标志信号：`flag`, `error`, `done`。

2. **明确信号的时序**

   信号名称应体现信号所处的流水线阶段或更新的时序。

   - 阶段前缀：如 `stage1_`，表示信号属于第 1 阶段。
   - 阶段后缀：如 `_stage2`，表示信号将用于第 2 阶段。

3. **保持一致性**

   - **命名风格一致**：在整个设计中统一采用前缀或后缀，避免混杂。

   - **语义一致**：同类型的信号（如 `valid` 信号）命名规则应统一。

4. **避免歧义**

   信号名称应避免模糊或冗余，确保不易混淆。

### 4.5结构化注释

为了使注释更具条理性，可以采用简单的标题样式，明确区分每个阶段。例如：

```verilog
// ====================================
// Stage 1: Data Preprocessing
// ====================================
// - Input: 
//   - pixel_data: 8-bit input data
//   - input_valid: Indicates if input is valid
// - Output: 
//   - stage1_data: Processed data
//   - stage1_valid: Data validity flag
// ------------------------------------
// - Notes:
//   - Ensure pixel_data is properly synchronized.
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        stage1_data <= 0;
        stage1_valid <= 0;
    end else begin
        stage1_data <= pixel_data + 1;
        stage1_valid <= input_valid;
    end
end

```

## 总结

通过 TMDS 编码项目的实践，我加深了对流水线设计的理解，尤其是在以下方面：

1. 信号时序和对齐的管理至关重要。
2. 结构化的模块设计和清晰的信号命名规则有助于提高代码可读性和维护性。
3. 流水线设计需要根据实际需求灵活选择单块设计或分块设计。

在未来的设计中，我将继续优化流水线结构，并探索更多复杂项目中的最佳实践。