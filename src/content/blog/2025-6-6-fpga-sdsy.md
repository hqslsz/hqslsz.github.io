---
title: 1
description: 🐸
pubDate: 2000-6-6 # 发表日期，注意格式，可以参考其他文章或config里的date_format
image: /img/projects-bg.jpg # 可选，文章封面图路径，图片放public/image/下
categories:
  - tech
tags:
  - verilog
# badge: Pin # 如果想置顶，可以加上
draft: false # false 表示发布，true 表示草稿
---

# fpga

SWJTU数电课设电子琴

## 核心代码

### 顶层fpga.v

```verilog
// File: fpga.v (Modified for practice mode integration)
// Top-level module for the FPGA Piano with recording, semitones, song playback, and practice mode
// 文件名: fpga.v (为集成练习模式而修改)
// FPGA电子琴的顶层模块，包含录音、半音、歌曲播放和练习模式功能

module fpga (
    // Clock and Reset
    // 时钟和复位信号
    input clk_50mhz,             // 50MHz 系统时钟 (来自开发板 PIN_90)
    input sw0_physical_reset,    // 物理复位按键 (Key0/SW0, PIN_24, 高电平有效，按下为高)

    // Musical Note Keys
    // 音符按键 (C,D,E,F,G,A,B)
    input [6:0] note_keys_physical_in, // Key1-Key7 物理按键输入
                                       // PIN_31(K1), PIN_30(K2), PIN_33(K3), PIN_32(K4)
                                       // PIN_42(K5), PIN_39(K6), PIN_44(K7)

    // Semitone Keys (Key8-Key12)
    // 半音按键 (升降号)
    input key8_sharp1_raw,         // 1# (C#) - PIN_43 (SW8) 原始输入
    input key9_flat3_raw,          // 3b (Eb) - PIN_13 (SW9) 原始输入
    input key10_sharp4_raw,        // 4# (F#) - PIN_6  (SW10) 原始输入
    input key11_sharp5_raw,        // 5# (G#) - PIN_144(SW11) 原始输入
    input key12_flat7_raw,         // 7b (Bb) - PIN_8  (SW12) 原始输入

    // Control Keys
    // 控制按键
    input sw15_octave_up_raw,    // 升八度按键 (Key15/SW15, PIN_10) 原始输入
    input sw13_octave_down_raw,  // 降八度按键 (Key13/SW13, PIN_7) 原始输入
    input sw16_record_raw,       // 录音按键 (Key16/SW16, PIN_142) 原始输入
    input sw17_playback_raw,     // 播放按键 (Key17/SW17, PIN_137) 原始输入
    input key14_play_song_raw,   // 播放预设歌曲按键 (Key14/SW14, PIN_11) 原始输入

    // Outputs
    // 输出信号
    output reg buzzer_out,       // 蜂鸣器输出 (PIN_128)

    // Outputs for 7-Segment Display
    // 7段数码管输出
    output seven_seg_a, output seven_seg_b, output seven_seg_c, output seven_seg_d, // 段选 a-g
    output seven_seg_e, output seven_seg_f, output seven_seg_g, output seven_seg_dp, // 段选 dp (小数点)
    output [7:0] seven_seg_digit_selects // 位选 SEG0-SEG7
);

// --- Internal Reset Logic ---
// --- 内部复位逻辑 ---
wire rst_n_internal; // 内部使用的低电平有效复位信号
assign rst_n_internal = ~sw0_physical_reset; // 物理复位键按下 (高电平) 时，内部复位信号为低

// --- Debouncer Parameter ---
// --- 按键消抖参数 ---
localparam DEBOUNCE_TIME_MS = 20; // 消抖时间（毫秒）
localparam DEBOUNCE_CYCLES_CALC = (DEBOUNCE_TIME_MS * 50000); // 根据50MHz时钟计算出的消抖周期数 (20ms * 50000 cycles/ms = 1,000,000 cycles)

// --- Consolidate All Musical Key Inputs ---
// --- 整合所有音乐按键输入 ---
localparam NUM_BASE_KEYS = 7; // 基础音符按键数量 (CDEFGAB)
localparam NUM_SEMITONE_KEYS = 5; // 半音按键数量
localparam NUM_TOTAL_MUSICAL_KEYS = NUM_BASE_KEYS + NUM_SEMITONE_KEYS; // 总音乐按键数量
localparam RECORDER_KEY_ID_BITS = 4; // 录音模块中按键ID的位数 (用于12个音乐按键 + 休止符0，所以至少需要能表示0-12，4位即可)
localparam RECORDER_OCTAVE_BITS = 2; // 录音模块中八度信息的位数 (如00, 01, 10 分别代表不同八度)

wire [NUM_TOTAL_MUSICAL_KEYS-1:0] all_musical_keys_raw; // 包含所有音乐按键原始输入的总线
// 将物理按键输入映射到总线，方便后续键盘扫描器处理
assign all_musical_keys_raw[0] = note_keys_physical_in[0]; // C -> 在扫描器中ID为1
assign all_musical_keys_raw[1] = note_keys_physical_in[1]; // D -> ID 2
assign all_musical_keys_raw[2] = note_keys_physical_in[2]; // E -> ID 3
assign all_musical_keys_raw[3] = note_keys_physical_in[3]; // F -> ID 4
assign all_musical_keys_raw[4] = note_keys_physical_in[4]; // G -> ID 5
assign all_musical_keys_raw[5] = note_keys_physical_in[5]; // A -> ID 6
assign all_musical_keys_raw[6] = note_keys_physical_in[6]; // B -> ID 7
assign all_musical_keys_raw[7] = key8_sharp1_raw;          // C# -> ID 8
assign all_musical_keys_raw[8] = key9_flat3_raw;           // Eb -> ID 9
assign all_musical_keys_raw[9] = key10_sharp4_raw;         // F# -> ID 10
assign all_musical_keys_raw[10] = key11_sharp5_raw;        // G# -> ID 11
assign all_musical_keys_raw[11] = key12_flat7_raw;         // Bb -> ID 12

// --- Keyboard Scanner Instance ---
// --- 键盘扫描器实例化 ---
wire [RECORDER_KEY_ID_BITS-1:0] current_active_key_id_internal; // 从扫描器输出的当前活动按键ID (0表示无按键, 1-12表示对应按键)
wire       current_key_is_pressed_flag_internal; // 当前是否有按键按下的标志
keyboard_scanner #( .NUM_KEYS(NUM_TOTAL_MUSICAL_KEYS), .DEBOUNCE_TIME_MS(DEBOUNCE_TIME_MS) )
keyboard_scanner_inst (
    .clk(clk_50mhz), .rst_n(rst_n_internal), .keys_in_raw(all_musical_keys_raw), // 输入：时钟、复位、原始按键总线
    .active_key_id(current_active_key_id_internal), .key_is_pressed(current_key_is_pressed_flag_internal) // 输出：活动按键ID、按键按下标志
);

// --- Debouncers for Control Keys ---
// --- 控制按键的消抖器 ---
wire sw15_octave_up_debounced_internal, sw13_octave_down_debounced_internal; // 升降八度消抖后信号
wire sw16_record_debounced_internal, sw17_playback_debounced_internal, key14_play_song_debounced_internal; // 录音、播放、预设歌曲消抖后信号
wire sw17_playback_pulse_internal; // 播放按键的脉冲信号 (用于触发播放开始)

// 为每个控制按键实例化一个消抖模块
debouncer #( .DEBOUNCE_CYCLES(DEBOUNCE_CYCLES_CALC) ) octave_up_deb_inst (.clk(clk_50mhz), .rst_n(rst_n_internal), .key_in_raw(sw15_octave_up_raw),   .key_out_debounced(sw15_octave_up_debounced_internal));
debouncer #( .DEBOUNCE_CYCLES(DEBOUNCE_CYCLES_CALC) ) octave_down_deb_inst(.clk(clk_50mhz), .rst_n(rst_n_internal), .key_in_raw(sw13_octave_down_raw), .key_out_debounced(sw13_octave_down_debounced_internal));
debouncer #( .DEBOUNCE_CYCLES(DEBOUNCE_CYCLES_CALC) ) record_deb_inst(.clk(clk_50mhz), .rst_n(rst_n_internal), .key_in_raw(sw16_record_raw),       .key_out_debounced(sw16_record_debounced_internal));
debouncer #( .DEBOUNCE_CYCLES(DEBOUNCE_CYCLES_CALC) ) playback_deb_inst(.clk(clk_50mhz), .rst_n(rst_n_internal), .key_in_raw(sw17_playback_raw),     .key_out_debounced(sw17_playback_debounced_internal));
debouncer #( .DEBOUNCE_CYCLES(DEBOUNCE_CYCLES_CALC) ) play_song_deb_inst(.clk(clk_50mhz), .rst_n(rst_n_internal), .key_in_raw(key14_play_song_raw),   .key_out_debounced(key14_play_song_debounced_internal));

// 生成播放按键的单脉冲信号 (上升沿检测)
reg sw17_playback_debounced_prev; initial sw17_playback_debounced_prev = 1'b0; // 存储上一周期的播放按键状态，初值为0
always @(posedge clk_50mhz or negedge rst_n_internal) begin // 时钟上升沿或复位信号下降沿触发
    if(!rst_n_internal) sw17_playback_debounced_prev <= 1'b0; // 复位时，前一状态为0
    else sw17_playback_debounced_prev <= sw17_playback_debounced_internal; // 否则，更新为当前播放按键状态
end
assign sw17_playback_pulse_internal = sw17_playback_debounced_internal & ~sw17_playback_debounced_prev; // 当前按下且上一周期未按下，则产生脉冲

// --- Piano Recorder Instance ---
// --- 电子琴录音模块实例化 ---
wire [RECORDER_KEY_ID_BITS-1:0] playback_key_id_feed; // 从录音模块输出的播放按键ID
wire playback_key_is_pressed_feed; // 从录音模块输出的播放按键按下标志
wire playback_octave_up_feed, playback_octave_down_feed; // 从录音模块输出的播放八度升降信号
wire is_recording_status, is_playing_status; // 录音模块状态：是否正在录音，是否正在播放
piano_recorder #( .CLK_FREQ_HZ(50_000_000), .RECORD_INTERVAL_MS(20), .MAX_RECORD_SAMPLES(512), .KEY_ID_BITS(RECORDER_KEY_ID_BITS), .OCTAVE_BITS(RECORDER_OCTAVE_BITS) )
piano_recorder_inst (
    .clk(clk_50mhz), .rst_n(rst_n_internal), // 时钟、复位
    .record_active_level(sw16_record_debounced_internal), .playback_start_pulse(sw17_playback_pulse_internal), // 录音激活(电平)、播放开始(脉冲)
    .live_key_id(current_active_key_id_internal), .live_key_is_pressed(current_key_is_pressed_flag_internal), // 实时按键ID和按下标志 (用于录制)
    .live_octave_up(sw15_octave_up_debounced_internal), .live_octave_down(sw13_octave_down_debounced_internal), // 实时八度升降信号 (用于录制)
    .playback_key_id(playback_key_id_feed), .playback_key_is_pressed(playback_key_is_pressed_feed), // 输出：播放时的按键ID和按下标志
    .playback_octave_up(playback_octave_up_feed), .playback_octave_down(playback_octave_down_feed), // 输出：播放时的八度升降信号
    .is_recording(is_recording_status), .is_playing(is_playing_status) // 输出：当前录音/播放状态
);

// --- Song Player Instance ---
// --- 歌曲播放器模块实例化 ---
wire [RECORDER_KEY_ID_BITS-1:0] song_player_key_id_feed; // 从歌曲播放器输出的按键ID
wire song_player_key_is_pressed_feed; // 从歌曲播放器输出的按键按下标志
wire song_player_octave_up_internal, song_player_octave_down_internal; // 从歌曲播放器输出的八度升降信号
wire is_song_playing_status; // 歌曲播放器状态：是否正在播放歌曲
song_player #( .CLK_FREQ_HZ(50_000_000), .KEY_ID_BITS(RECORDER_KEY_ID_BITS), .OCTAVE_BITS(RECORDER_OCTAVE_BITS) )
song_player_inst (
    .clk(clk_50mhz), .rst_n(rst_n_internal), // 时钟、复位
    .play_active_level(key14_play_song_debounced_internal), // 歌曲播放激活信号(电平)
    .song_key_id(song_player_key_id_feed), .song_key_is_pressed(song_player_key_is_pressed_feed), // 输出：播放歌曲时的按键ID和按下标志
    .song_octave_up_feed(song_player_octave_up_internal), .song_octave_down_feed(song_player_octave_down_internal), // 输出：播放歌曲时的八度升降信号
    .is_song_playing(is_song_playing_status) // 输出：当前是否正在播放歌曲
);

// --- Mode Sequencer Instance (NEW) ---
// --- 模式序列器实例化 (新增，用于练习模式) ---
wire practice_mode_trigger_pulse_internal; // 练习模式触发脉冲信号
mode_sequencer mode_sequencer_inst (
    .clk(clk_50mhz),
    .rst_n(rst_n_internal),
    .current_live_key_id(current_active_key_id_internal), // 输入：来自键盘扫描器的当前实时按键ID
    .current_live_key_pressed(current_key_is_pressed_flag_internal), // 输入：来自键盘扫描器的当前实时按键按下标志
    .practice_mode_active_pulse(practice_mode_trigger_pulse_internal) // 输出：练习模式激活脉冲
);

// --- Practice Mode Enable Logic (NEW) ---
// --- 练习模式使能逻辑 (新增) ---
reg practice_mode_enabled_reg; initial practice_mode_enabled_reg = 1'b0; // 练习模式使能寄存器，默认为禁用
always @(posedge clk_50mhz or negedge rst_n_internal) begin
    if (!rst_n_internal) begin // 复位时
        practice_mode_enabled_reg <= 1'b0; // 禁用练习模式
    end else begin
        if (practice_mode_trigger_pulse_internal) begin // 当接收到练习模式触发脉冲时
            practice_mode_enabled_reg <= ~practice_mode_enabled_reg; // 翻转练习模式的使能状态 (实现开关功能)
        end
    end
end

// --- Practice Player Instance (NEW) ---
// --- 练习播放器实例化 (新增) ---
localparam NUM_PRACTICE_DISPLAY_SEGMENTS = 6; // 练习模式用于显示的数码管段数
wire [2:0] practice_data_s0; // 用于从 practice_player 输出到显示器的 practice_out_seg0 数据
wire [2:0] practice_data_s1;
wire [2:0] practice_data_s2;
wire [2:0] practice_data_s3;
wire [2:0] practice_data_s4;
wire [2:0] practice_data_s5; // 存储练习显示数据的数组
wire practice_correct_event; // 练习正确事件标志
wire practice_wrong_event;   // 练习错误事件标志
wire practice_finished_event; // 练习完成事件标志

// 在 fpga.v 中, practice_player_inst 实例化
practice_player #( .NUM_DISPLAY_SEGMENTS(NUM_PRACTICE_DISPLAY_SEGMENTS) ) practice_player_inst (
    .clk(clk_50mhz),
    .rst_n(rst_n_internal),
    .practice_mode_active(practice_mode_enabled_reg), // 输入：练习模式是否激活
    .current_live_key_id(current_active_key_id_internal), // 输入：当前实时按键ID
    .current_live_key_pressed(current_key_is_pressed_flag_internal), // 输入：当前实时按键按下标志
    //.display_data_practice_seg(practice_seg_data_feed), // 旧的连接方式
    // NEW: 连接到独立的端口
    .display_out_seg0(practice_data_s0), // 输出：练习模式显示数据段0
    .display_out_seg1(practice_data_s1), // 输出：练习模式显示数据段1
    .display_out_seg2(practice_data_s2), // 输出：练习模式显示数据段2
    .display_out_seg3(practice_data_s3), // 输出：练习模式显示数据段3
    .display_out_seg4(practice_data_s4), // 输出：练习模式显示数据段4
    .display_out_seg5(practice_data_s5), // 输出：练习模式显示数据段5
    .correct_note_played_event(practice_correct_event), // 输出：弹奏正确音符事件
    .wrong_note_played_event(practice_wrong_event),     // 输出：弹奏错误音符事件
    .practice_song_finished_event(practice_finished_event) // 输出：练习曲目完成事件
);

// --- Sound/Display Source Multiplexer (MODIFIED for practice mode) ---
// --- 声音/显示源多路选择器 (为练习模式修改) ---
wire [RECORDER_KEY_ID_BITS-1:0] final_key_id_for_sound_and_display; // 最终用于声音和显示的按键ID
wire final_key_is_pressed_for_sound_and_display; // 最终用于声音和显示的按键按下标志
wire final_octave_up_for_sound_and_display, final_octave_down_for_sound_and_display; // 最终用于声音和显示的八度控制信号

// 在练习模式下，声音/显示逻辑会有所不同。
// 目前，假设练习模式本身不直接驱动主蜂鸣器/主音符显示
// (它有自己的显示输出 practice_seg_data_feed 和声音事件标志)。
// 所以，如果练习模式开启，来自按键的主声音生成可能会被禁用或以不同方式处理。

// 优先级: 预设歌曲播放 > 录音回放 > 实时按键 (如果不在练习模式，或者练习模式允许实时声音通过)
// 在 fpga.v 中
// --- 声音/显示源多路选择器 (为练习模式声音修改) ---
// ... (final_key_id_for_sound_and_display 等声明) ...

// 优先级顺序:
// 1. 练习模式: 声音来自实时按键
// 2. 预设歌曲播放器
// 3. 录音回放
// 4. 实时按键 (普通模式)

// 根据当前模式选择最终的按键ID
assign final_key_id_for_sound_and_display =
    (practice_mode_enabled_reg) ? current_active_key_id_internal : // <<< 修改: 练习模式下，声音来自实时按键
    (is_song_playing_status ? song_player_key_id_feed : // 否则，如果正在播放预设歌曲，使用歌曲播放器的按键ID
    (is_playing_status ? playback_key_id_feed : current_active_key_id_internal)); // 否则，如果正在回放录音，使用录音回放的按键ID，最后才是实时按键ID

// 根据当前模式选择最终的按键按下标志
assign final_key_is_pressed_for_sound_and_display =
    (practice_mode_enabled_reg) ? current_key_is_pressed_flag_internal : // <<< 修改: 练习模式下，使用实时按键的按下标志
    (is_song_playing_status ? song_player_key_is_pressed_feed : // 逻辑同上
    (is_playing_status ? playback_key_is_pressed_feed : current_key_is_pressed_flag_internal));

// 练习模式下的八度声音也将来自全局八度按钮 (根据此更改)
// 根据当前模式选择最终的升八度信号
assign final_octave_up_for_sound_and_display =
    (is_song_playing_status && !practice_mode_enabled_reg) ? song_player_octave_up_internal : // 歌曲八度仅在非练习模式下生效
    ((is_playing_status && !practice_mode_enabled_reg) ? playback_octave_up_feed : // 回放八度仅在非练习模式下生效
    sw15_octave_up_debounced_internal); // 实时/练习模式八度 (使用实时按键的八度控制)

// 根据当前模式选择最终的降八度信号
assign final_octave_down_for_sound_and_display =
    (is_song_playing_status && !practice_mode_enabled_reg) ? song_player_octave_down_internal : // 逻辑同上
    ((is_playing_status && !practice_mode_enabled_reg) ? playback_octave_down_feed :
    sw13_octave_down_debounced_internal);

// --- Buzzer Frequency Generation ---
// --- 蜂鸣器频率生成 (PWM驱动) ---
// 定义C4调各音符对应的计数器周期值 (用于50MHz时钟，产生特定频率的PWM)
// 值越大，频率越低；值越小，频率越高。这些值是产生半个周期所需的时钟数。
// 例如 CNT_C4 = 95566, 频率 = 50MHz / (2 * 95566) approx 261.6Hz (C4)
localparam CNT_C4=17'd95566, CNT_CS4=17'd90194, CNT_D4=17'd85135, CNT_DS4=17'd80346, CNT_E4=17'd75830;
localparam CNT_F4=17'd71569, CNT_FS4=17'd67569, CNT_G4=17'd63775, CNT_GS4=17'd60197, CNT_A4=17'd56817;
localparam CNT_AS4=17'd53627,CNT_B4=17'd50619;

reg [17:0] buzzer_counter_reg;     // 蜂鸣器PWM计数器寄存器
reg [17:0] base_note_target_count; // 基础音符的目标计数值 (未调整八度)
reg [17:0] final_target_count_max; // 最终调整八度后的目标计数值 (PWM半周期)

// 组合逻辑：根据最终按键ID确定基础音符的目标计数值
always @(*) begin
    case (final_key_id_for_sound_and_display) // 使用多路选择器输出的按键ID
        4'd1:  base_note_target_count = CNT_C4;  4'd8:  base_note_target_count = CNT_CS4; // C, C#
        4'd2:  base_note_target_count = CNT_D4;  4'd9:  base_note_target_count = CNT_DS4; // D, D# (Eb your ID is 9)
        4'd3:  base_note_target_count = CNT_E4;  // E (你的代码中 ID 9 被用于 Eb, 这里没有单独的 Eb 默认值)
        4'd4:  base_note_target_count = CNT_F4;  4'd10: base_note_target_count = CNT_FS4; // F, F#
        4'd5:  base_note_target_count = CNT_G4;  4'd11: base_note_target_count = CNT_GS4; // G, G#
        4'd6:  base_note_target_count = CNT_A4;  4'd12: base_note_target_count = CNT_AS4; // A, A# (Bb your ID is 12)
        4'd7:  base_note_target_count = CNT_B4;  // B
        default: base_note_target_count = 18'h3FFFF; // 默认静音 (一个较大的计数值，使频率非常低或听不见)
    endcase

    // 根据八度控制信号调整最终的目标计数值
    if (final_octave_up_for_sound_and_display && !final_octave_down_for_sound_and_display) begin // 升八度
        final_target_count_max = (base_note_target_count + 1) / 2 - 1; // 频率加倍 (计数值减半)
    end else if (!final_octave_up_for_sound_and_display && final_octave_down_for_sound_and_display) begin // 降八度
        final_target_count_max = (base_note_target_count + 1) * 2 - 1; // 频率减半 (计数值加倍)
    end else begin // 普通八度
        final_target_count_max = base_note_target_count;
    end
end

initial begin buzzer_out = 1'b0; buzzer_counter_reg = 18'd0; end // 初始化蜂鸣器输出和计数器

// 时序逻辑：生成PWM波形驱动蜂鸣器
always @(posedge clk_50mhz or negedge rst_n_internal) begin
    if(!rst_n_internal) begin // 复位
        buzzer_counter_reg <= 18'd0; buzzer_out <= 1'b0;
    // MODIFIED: Buzzer logic needs to consider practice mode feedback sounds
    // (已修改: 蜂鸣器逻辑需要考虑练习模式的反馈声音)
    // This is a SIMPLIFICATION. A more complex sound muxer might be needed.
    // (这是一个简化处理。可能需要一个更复杂的声音多路选择器。)
    // For now, main buzzer responds to final_key_is_pressed...
    // (目前，主蜂鸣器响应 final_key_is_pressed...)
    // Practice mode correct/wrong events could trigger specific short tones via another PWM or by briefly overriding these.
    // (练习模式的正确/错误事件可以通过另一个PWM或短暂覆盖这些来触发特定的短音。)
    end else if (final_key_is_pressed_for_sound_and_display && final_key_id_for_sound_and_display != 4'd0) begin // 如果有有效按键按下
        if (buzzer_counter_reg >= final_target_count_max) begin // 计数器达到目标值 (半周期结束)
            buzzer_counter_reg <= 18'd0; // 计数器清零
            buzzer_out <= ~buzzer_out;   // 翻转蜂鸣器输出电平 (产生方波)
        end else begin
            buzzer_counter_reg <= buzzer_counter_reg + 1'b1; // 计数器加1
        end
    // NEW: Add simple feedback for practice mode (can be improved)
    // (新增: 为练习模式添加简单的反馈声音，可以改进)
    end else if (practice_correct_event) begin // 练习正确事件 (产生一个简短的高音)
        // This is a placeholder for a proper sound. It will conflict if a note is also sounding.
        // (这是一个占位符，用于表示一个合适的提示音。如果同时有音符在响，可能会产生冲突。)
        // A dedicated sound generator for feedback is better.
        // (一个专用的反馈声音发生器会更好。)
        // For now, let's make it a very short click or high pitch.
        // (目前，我们让它产生一个非常短的咔哒声或高音。)
        // This simple version might not sound good.
        // (这个简单版本的声音可能不太好听。)
        buzzer_counter_reg <= 18'd0; // 例如，清零计数器，如果 final_target_count_max 此时是 CNT_A4，则会发出约440Hz的声音
        // buzzer_out <= ~buzzer_out; // 脉冲可能太短听不见，或者会与上面的逻辑冲突
    end else if (practice_wrong_event) begin // 练习错误事件 (产生一个简短的低音)
        // buzzer_counter_reg <= CNT_C4 * 2; // 示例：一个非常低的频率
        // buzzer_out <= ~buzzer_out; // 同上
    end else begin // 没有按键按下，也没有练习事件
        buzzer_counter_reg <= 18'd0; // 计数器清零
        buzzer_out <= 1'b0;          // 蜂鸣器无输出 (静音)
    end
end

// --- Data Preparation for Display Modules (MODIFIED for practice mode priority on SEG0/SEG7) ---
// --- 为显示模块准备数据 (为练习模式在SEG0/SEG7上的优先级而修改) ---
reg [2:0] base_note_id_for_buffer_and_suffix; // 用于滚动显示缓冲和后缀显示的基础音符ID (1-7 代表 C-B)
reg [1:0] semitone_type_for_suffix;           // 用于后缀显示的半音类型 (00:无, 01:升号#, 10:降号b)
reg       current_note_is_valid_for_display;  // 当前音符是否有效，用于显示

// 这个 always 块主要为滚动显示(非练习模式)和SEG0上的半音后缀准备数据。
always @(*) begin
    base_note_id_for_buffer_and_suffix = 3'd0; // 默认无音符
    semitone_type_for_suffix = 2'b00;          // 默认无半音
    current_note_is_valid_for_display = 1'b0;  // 默认当前音符无效

    // 如果在练习模式下，SEG0/SEG7 可能会显示不同的内容 (在 seven_segment_controller 中处理)
    // 这部分用于非练习模式时，或用于一般按下的音符
    if (final_key_is_pressed_for_sound_and_display && final_key_id_for_sound_and_display != 4'd0) begin // 如果有有效按键按下
        current_note_is_valid_for_display = 1'b1; // 标记当前音符有效
        case (final_key_id_for_sound_and_display) // 根据最终的按键ID解码
            4'd1:  begin base_note_id_for_buffer_and_suffix = 3'd1; semitone_type_for_suffix = 2'b00; end // C (ID 1 -> 哆)
            4'd2:  begin base_note_id_for_buffer_and_suffix = 3'd2; semitone_type_for_suffix = 2'b00; end // D (ID 2 -> 来)
            4'd3:  begin base_note_id_for_buffer_and_suffix = 3'd3; semitone_type_for_suffix = 2'b00; end // E (ID 3 -> 咪)
            4'd4:  begin base_note_id_for_buffer_and_suffix = 3'd4; semitone_type_for_suffix = 2'b00; end // F (ID 4 -> 发)
            4'd5:  begin base_note_id_for_buffer_and_suffix = 3'd5; semitone_type_for_suffix = 2'b00; end // G (ID 5 -> सो)
            4'd6:  begin base_note_id_for_buffer_and_suffix = 3'd6; semitone_type_for_suffix = 2'b00; end // A (ID 6 -> 拉)
            4'd7:  begin base_note_id_for_buffer_and_suffix = 3'd7; semitone_type_for_suffix = 2'b00; end // B (ID 7 -> 西)
            4'd8:  begin base_note_id_for_buffer_and_suffix = 3'd1; semitone_type_for_suffix = 2'b01; end // C# (ID 8 -> 升哆)
            4'd9:  begin base_note_id_for_buffer_and_suffix = 3'd3; semitone_type_for_suffix = 2'b10; end // Eb (ID 9 -> 降咪)
            4'd10: begin base_note_id_for_buffer_and_suffix = 3'd4; semitone_type_for_suffix = 2'b01; end // F# (ID 10 -> 升发)
            4'd11: begin base_note_id_for_buffer_and_suffix = 3'd5; semitone_type_for_suffix = 2'b01; end // G# (ID 11 -> 升索)
            4'd12: begin base_note_id_for_buffer_and_suffix = 3'd7; semitone_type_for_suffix = 2'b10; end // Bb (ID 12 -> 降西)
            default: current_note_is_valid_for_display = 1'b0; // 其他情况，音符无效
        endcase
    end
end

// --- Generate Pulse for New Valid Note to Trigger Scrolling Buffer ---
// --- 为新的有效音符生成脉冲以触滚动显示缓冲区 ---
reg  final_key_is_pressed_for_sound_and_display_prev; // 上一周期最终按键按下状态
reg  [RECORDER_KEY_ID_BITS-1:0] final_key_id_for_sound_and_display_prev; // 上一周期最终按键ID
wire new_note_to_scroll_pulse; // 新音符触发滚动显示的脉冲

initial begin
    final_key_is_pressed_for_sound_and_display_prev = 1'b0; // 初始化
    final_key_id_for_sound_and_display_prev = {RECORDER_KEY_ID_BITS{1'b0}}; // 初始化
end

// 存储上一周期的按键状态和ID
always @(posedge clk_50mhz or negedge rst_n_internal) begin
    if(!rst_n_internal) begin
        final_key_is_pressed_for_sound_and_display_prev <= 1'b0;
        final_key_id_for_sound_and_display_prev <= {RECORDER_KEY_ID_BITS{1'b0}};
    end else begin
        final_key_is_pressed_for_sound_and_display_prev <= final_key_is_pressed_for_sound_and_display;
        if (final_key_is_pressed_for_sound_and_display) begin // 只有当当前有键按下时，才更新上一个ID
            final_key_id_for_sound_and_display_prev <= final_key_id_for_sound_and_display;
        end else begin // 如果当前无键按下，上一个ID也清零
            final_key_id_for_sound_and_display_prev <= {RECORDER_KEY_ID_BITS{1'b0}};
        end
    end
end

// 生成新音符的脉冲信号，条件：
// 1. 不在练习模式下
// 2. (当前键按下 且 上一周期未按下) 或者 (当前键按下 且 当前ID与上一周期ID不同)
// 3. 并且 当前音符是有效的
assign new_note_to_scroll_pulse =
    !practice_mode_enabled_reg && // 仅当不在练习模式时才滚动
    (
        (final_key_is_pressed_for_sound_and_display && !final_key_is_pressed_for_sound_and_display_prev) || // 新按键按下
        (final_key_is_pressed_for_sound_and_display && (final_key_id_for_sound_and_display != final_key_id_for_sound_and_display_prev)) // 或按键改变
    ) && current_note_is_valid_for_display; // 且当前音符有效

// --- Instantiate Scrolling Display Buffer ---
// --- 实例化滚动显示缓冲区 ---
wire [2:0] scroll_data_seg1_feed; wire [2:0] scroll_data_seg2_feed; // 滚动数据显示SEG1, SEG2
wire [2:0] scroll_data_seg3_feed; wire [2:0] scroll_data_seg4_feed; // 滚动数据显示SEG3, SEG4
wire [2:0] scroll_data_seg5_feed; wire [2:0] scroll_data_seg6_feed; // 滚动数据显示SEG5, SEG6

scrolling_display_buffer scroller_inst (
    .clk(clk_50mhz),
    .rst_n(rst_n_internal),
    .new_note_valid_pulse(new_note_to_scroll_pulse), // 输入：新有效音符脉冲 (触发数据移入)
    .current_base_note_id_in(base_note_id_for_buffer_and_suffix), // 输入：当前基础音符ID
    .display_data_seg1(scroll_data_seg1_feed), // 输出：用于数码管SEG1的数据
    .display_data_seg2(scroll_data_seg2_feed), // 输出：用于数码管SEG2的数据
    .display_data_seg3(scroll_data_seg3_feed), // ...
    .display_data_seg4(scroll_data_seg4_feed),
    .display_data_seg5(scroll_data_seg5_feed),
    .display_data_seg6(scroll_data_seg6_feed)
);

// --- Wires for final data to seven_segment_controller (NEW) ---
// --- 连接到 seven_segment_controller 的最终数据的连线 (新增) ---
wire [2:0] final_to_sev_seg1_data; wire [2:0] final_to_sev_seg2_data;
wire [2:0] final_to_sev_seg3_data; wire [2:0] final_to_sev_seg4_data;
wire [2:0] final_to_sev_seg5_data; wire [2:0] final_to_sev_seg6_data;

// --- Multiplex data for SEG1-SEG6 display (NEW) ---
// --- 为 SEG1-SEG6 显示复用数据 (新增) ---
// 如果练习模式使能，则数码管 SEG1-SEG6 显示练习数据；否则，显示滚动数据。
assign final_to_sev_seg1_data = practice_mode_enabled_reg ? practice_data_s0 : scroll_data_seg1_feed;
assign final_to_sev_seg2_data = practice_mode_enabled_reg ? practice_data_s1 : scroll_data_seg2_feed;
assign final_to_sev_seg3_data = practice_mode_enabled_reg ? practice_data_s2 : scroll_data_seg3_feed;
assign final_to_sev_seg4_data = practice_mode_enabled_reg ? practice_data_s3 : scroll_data_seg4_feed;
assign final_to_sev_seg5_data = practice_mode_enabled_reg ? practice_data_s4 : scroll_data_seg5_feed;
assign final_to_sev_seg6_data = practice_mode_enabled_reg ? practice_data_s5 : scroll_data_seg6_feed;

// --- Instantiate Seven Segment Controller (MODIFIED inputs) ---
// --- 实例化七段数码管控制器 (输入已修改) ---
seven_segment_controller seven_segment_display_inst (
    .clk(clk_50mhz),
    .rst_n(rst_n_internal),

    // Inputs for SEG0 (Suffix / Practice Mode Indicator)
    // SEG0的输入 (后缀 / 练习模式指示符)
    // MODIFIED: If in practice, show 'P', else show semitone.
    // (已修改: 如果在练习模式，显示 'P'，否则显示半音符号)
    .semitone_type_in(practice_mode_enabled_reg ? 2'b11 : semitone_type_for_suffix), // 2'b11 可以是你控制器中 'P' 的编码
    .semitone_display_active_flag(practice_mode_enabled_reg ? 1'b1 : current_note_is_valid_for_display), // 练习模式下SEG0常亮'P'，否则根据音符有效性显示

    // Inputs for SEG1-SEG6 (Muxed data)
    // SEG1-SEG6的输入 (复用后的数据)
    .scrolled_note_seg1_in(final_to_sev_seg1_data),
    .scrolled_note_seg2_in(final_to_sev_seg2_data),
    .scrolled_note_seg3_in(final_to_sev_seg3_data),
    .scrolled_note_seg4_in(final_to_sev_seg4_data),
    .scrolled_note_seg5_in(final_to_sev_seg5_data),
    .scrolled_note_seg6_in(final_to_sev_seg6_data),

    // Inputs for SEG7 (Octave / Practice Feedback)
    // SEG7的输入 (八度 / 练习反馈)
    // MODIFIED: If in practice, show feedback (e.g., based on practice_correct_event), else octave.
    // (已修改: 如果在练习模式，显示反馈 (例如，基于 practice_correct_event)，否则显示八度)
    .octave_up_active(practice_mode_enabled_reg ? practice_correct_event : (final_octave_up_for_sound_and_display && !final_octave_down_for_sound_and_display)), // 练习模式显示正确提示，否则显示升八度
    .octave_down_active(practice_mode_enabled_reg ? practice_wrong_event : (final_octave_down_for_sound_and_display && !final_octave_up_for_sound_and_display)), // 练习模式显示错误提示，否则显示降八度

    // Seven Segment Outputs
    // 七段数码管输出
    .seg_a(seven_seg_a), .seg_b(seven_seg_b), .seg_c(seven_seg_c), .seg_d(seven_seg_d),
    .seg_e(seven_seg_e), .seg_f(seven_seg_f), .seg_g(seven_seg_g), .seg_dp(seven_seg_dp),
    .digit_selects(seven_seg_digit_selects)
);

endmodule
```

### debouncer.v

```verilog
// File: debouncer.v
// 文件名: debouncer.v (按键消抖模块)
module debouncer #(
    parameter DEBOUNCE_CYCLES = 1000000 // Default for 20ms at 50MHz
                                        // 参数：消抖周期数，默认为1,000,000 (对应50MHz时钟下20ms)
) (
    input clk,                 // 时钟输入
    input rst_n,             // Active low reset (低电平有效复位)
    input key_in_raw,        // Raw input from physical key (active high)
                               // 来自物理按键的原始输入 (高电平有效)
    output reg key_out_debounced // Debounced key state (active high)
                                  // 消抖后的按键状态 (高电平有效)
);

// For DEBOUNCE_CYCLES = 1,000,000, counter width is 20 bits.
// (如果 DEBOUNCE_CYCLES = 1,000,000, 计数器需要20位宽 (2^20 > 10^6))
reg [19:0] count_reg;     // Counter for debounce timing (消抖计时计数器)
reg key_temp_state;       // Temporary state to track changes (跟踪按键变化的临时状态)

initial begin
    key_out_debounced = 1'b0; // Key inactive state (since active high) (按键无效状态，因为是高电平有效)
    key_temp_state    = 1'b0; // Assume key is not pressed initially (假设按键初始未按下)
    count_reg         = 0;    // 计数器初始化为0
end

always @(posedge clk or negedge rst_n) begin // 时钟上升沿或复位信号下降沿触发
    if (!rst_n) begin // 如果复位信号有效 (低电平)
        key_out_debounced <= 1'b0; // 输出消抖后的按键状态为无效
        key_temp_state    <= 1'b0; // 临时状态为无效
        count_reg         <= 0;    // 计数器清零
    end else begin // 正常工作状态
        if (key_in_raw != key_temp_state) begin // 如果原始按键输入与上一次记录的临时状态不同 (发生抖动或真实状态变化)
            // Raw input differs from last seen temporary state, means a potential change
            // (原始输入与上次观察到的临时状态不同，意味着可能发生变化)
            key_temp_state <= key_in_raw; // 更新临时状态为当前原始输入
            count_reg      <= 0;          // 重置计数器 (因为状态不稳定)
        end else begin // 如果原始按键输入与临时状态相同 (状态稳定，或抖动结束，或已变化并等待确认)
            // Raw input is the same as the temporary state (stable or changed and waiting)
            // (原始输入与临时状态相同 - 稳定，或已改变并等待)
            if (count_reg < DEBOUNCE_CYCLES - 1) begin // 如果计数器未达到设定的消抖周期数
                count_reg <= count_reg + 1'b1; // 计数器加1
            end else begin // 计数器已达到设定的消抖周期数 (表示按键状态已稳定)
                // Counter reached max, the key_temp_state is now considered stable
                // (计数器达到最大值，key_temp_state 现在被认为是稳定的)
                key_out_debounced <= key_temp_state; // 将稳定的临时状态输出为消抖后的按键状态
            end
        end
    end
end

endmodule
```

### keyboard_scanner.v

```verilog
// File: keyboard_scanner.v
// Scans multiple keys, debounces them, and outputs the ID of the highest priority pressed key.
// 文件名: keyboard_scanner.v
// (扫描多个按键，对其进行消抖，并输出优先级最高的已按下按键的ID)

module keyboard_scanner #(
    parameter NUM_KEYS = 7, // Default value, will be overridden by the top module (e.g., to 12)
                            // 参数：按键数量，默认值为7，会被顶层模块覆盖 (例如改为12)
    parameter DEBOUNCE_TIME_MS = 20 // 参数：消抖时间(毫秒)
) (
    input clk,                        // 时钟输入
    input rst_n,                      // Active low reset (低电平有效复位)
    input [NUM_KEYS-1:0] keys_in_raw, // Raw inputs from keys (e.g., keys_in_raw[0] for Key1)
                                      // 来自按键的原始输入 (例如 keys_in_raw[0] 对应按键1)

    // Output ID can be 0 (no key) or 1 up to NUM_KEYS.
    // So, it needs to represent NUM_KEYS + 1 distinct values.
    // The width required is $clog2(NUM_KEYS + 1).
    // Example: NUM_KEYS = 7 -> $clog2(8) = 3 bits (for 0-7)
    // Example: NUM_KEYS = 12 -> $clog2(13) = 4 bits (for 0-12)
    // (输出ID可以是0(无按键)或1到NUM_KEYS。因此需要表示NUM_KEYS+1个不同的值。所需的位宽是$clog2(NUM_KEYS+1)。)
    output reg [$clog2(NUM_KEYS + 1) - 1 : 0] active_key_id, // 活动按键ID (0表示无按键，1-NUM_KEYS表示对应按键)
    output reg key_is_pressed           // High if any key is currently pressed (debounced)
                                        // 如果当前有任何按键被按下(已消抖)，则为高电平
);

// Calculate debounce cycles based on 50MHz clock (passed via DEBOUNCE_TIME_MS)
// (根据50MHz时钟和DEBOUNCE_TIME_MS计算消抖周期数)
localparam DEBOUNCE_CYCLES_CALC = (DEBOUNCE_TIME_MS * 50000); // Assuming 50MHz clock (假设50MHz时钟)

// Array of debounced key states
// (存储所有按键消抖后状态的线网数组)
wire [NUM_KEYS-1:0] keys_debounced_signals;

// Instantiate debouncers for each key
// (为每个按键实例化消抖器)
genvar i; // generate块的循环变量
generate // Verilog generate块，用于批量实例化
    for (i = 0; i < NUM_KEYS; i = i + 1) begin : debounce_gen_block // 循环NUM_KEYS次
        debouncer #(
            .DEBOUNCE_CYCLES(DEBOUNCE_CYCLES_CALC) // 传递消抖周期参数
        ) inst_debouncer ( // 实例化一个debouncer模块
            .clk(clk),
            .rst_n(rst_n),
            .key_in_raw(keys_in_raw[i]), // 第i个按键的原始输入
            .key_out_debounced(keys_debounced_signals[i]) // 第i个按键消抖后的输出
        );
    end
endgenerate

// Declare loop variable for Verilog-2001 compatibility if needed by older tools
// For modern tools, 'j' can be declared directly in the for loop if 'automatic' is not desired for synthesis.
// (如果旧工具需要，为Verilog-2001兼容性声明循环变量'j'。对于现代工具，如果不想让'j'成为'automatic'类型以用于综合，可以直接在for循环中声明。)
integer j; // for循环中使用的整数型变量 (在组合逻辑块中使用)

// Logic to determine active_key_id and key_is_pressed
// Priority: If multiple keys are pressed, lowest index key (keys_in_raw[0]) wins.
// (确定 active_key_id 和 key_is_pressed 的逻辑)
// (优先级：如果多个按键同时按下，索引最低的按键(keys_in_raw[0])获胜)
always @(*) begin // 组合逻辑块
    key_is_pressed = 1'b0;      // Initialize: assume no key is pressed yet (初始化：假设当前没有按键按下)
    // Initialize active_key_id to 0 (no key pressed). Ensure correct width.
    // (初始化 active_key_id 为0 (无按键按下)。确保正确的位宽。)
    active_key_id = {$clog2(NUM_KEYS + 1){1'b0}}; // 活动按键ID初始化为0，使用复制操作符保证位宽

    // Iterate from lowest index (Key1, which is keys_debounced_signals[0])
    // to highest. The first one found will set the outputs.
    // (从最低索引(按键1，即keys_debounced_signals[0])遍历到最高索引。找到的第一个被按下的按键将设置输出。)
    for (j = 0; j < NUM_KEYS; j = j + 1) begin // 遍历所有消抖后的按键信号
        if (keys_debounced_signals[j]) begin // If this key 'j' is pressed (如果第j个按键被按下)
            if (!key_is_pressed) begin       // AND if we haven't already found a lower-index pressed key
                                             // (并且如果还没有找到索引更低的已按下按键 - 实现优先级)
                key_is_pressed = 1'b1;       // 设置按键按下标志为真
                active_key_id = j + 1;     // Assign its ID (j=0 is ID 1, j=1 is ID 2, etc.)
                                           // (分配其ID，j=0对应ID 1，j=1对应ID 2，依此类推)
            end
        end
    end
end

endmodule
```

### seven_segment_controller.v

```verilog
// File: seven_segment_controller.v
// Modified to display:
// - SEG0: Current Semitone Suffix (#/b) or 'P' for Practice
// - SEG1-SEG6: Scrolled Note Digits (from scrolling_display_buffer) or Practice Data
// - SEG7: Octave Status or Practice Feedback
// 文件名: seven_segment_controller.v
// (已修改以显示:
//  - SEG0: 当前半音后缀(#/b) 或 'P'表示练习模式
//  - SEG1-SEG6: 滚动的音符数字(来自scrolling_display_buffer) 或 练习数据
//  - SEG7: 八度状态 或 练习反馈)
module seven_segment_controller (
    input clk,    // 时钟
    input rst_n,  // 低电平有效复位

    // Inputs for SEG0 (Suffix / Practice Mode Indicator)
    // SEG0 (后缀 / 练习模式指示符) 的输入
    input [1:0] semitone_type_in,        // 半音类型输入 (00:无, 01:升号(#), 10:降号(b), 11:练习模式'P')
    input semitone_display_active_flag,  // SEG0后缀显示激活标志 (当为有效音符或练习模式时为真)

    // Inputs for SEG1-SEG6 (Scrolled Note Digits from buffer or Practice data)
    // SEG1-SEG6 (来自缓冲区的滚动音符数字 或 练习数据) 的输入
    // Each input is [2:0], 0 for blank, 1-7 for note C-B.
    // (每个输入是3位宽, 0表示空白, 1-7表示音符C-B，即哆-西)
    input [2:0] scrolled_note_seg1_in, // 数码管SEG1的音符数据
    input [2:0] scrolled_note_seg2_in, // 数码管SEG2的音符数据
    input [2:0] scrolled_note_seg3_in, // ...
    input [2:0] scrolled_note_seg4_in,
    input [2:0] scrolled_note_seg5_in,
    input [2:0] scrolled_note_seg6_in,

    // Inputs for SEG7 (Octave / Practice Feedback)
    // SEG7 (八度 / 练习反馈) 的输入
    input octave_up_active,     // 升八度激活标志 (或练习正确反馈)
    input octave_down_active,   // 降八度激活标志 (或练习错误反馈)

    // 数码管段选信号 (a-g, dp)
    output reg seg_a, output reg seg_b, output reg seg_c, output reg seg_d,
    output reg seg_e, output reg seg_f, output reg seg_g, output reg seg_dp,
    output reg [7:0] digit_selects // 数码管位选信号 (SEG0-SEG7)
);

// Segment patterns (common cathode)
// 段码模式 (共阴数码管，段选为1点亮)
//         gfedcba
localparam PATTERN_0    = 7'b0111111; // 0
localparam PATTERN_1    = 7'b0000110; // 1
localparam PATTERN_2    = 7'b1011011; // 2
localparam PATTERN_3    = 7'b1001111; // 3
localparam PATTERN_4    = 7'b1100110; // 4
localparam PATTERN_5    = 7'b1101101; // 5
localparam PATTERN_6    = 7'b1111101; // 6
localparam PATTERN_7    = 7'b0000111; // 7 (这里用作音符B的显示，实际显示1-7代表C-B)
localparam PATTERN_BLANK= 7'b0000000; // 空白
localparam PATTERN_H    = 7'b1110110; // H (用于显示升号 #)
localparam PATTERN_b    = 7'b1111100; // b (小写b，用于显示降号 b)
localparam PATTERN_P    = 7'b1110011; // P (用于练习模式指示) - 你可以根据需要修改，原代码中用semitone_type_in=2'b11来代表P

// Octave/Feedback patterns for SEG7
// SEG7的八度/反馈显示模式
localparam OCTAVE_UP_PATTERN    = 7'b0110000; // U (代表升八度或练习正确) - 可以自定义，例如箭头向上
localparam OCTAVE_NORMAL_PATTERN= 7'b0001110; // L (代表普通八度或无反馈) - 可以自定义，例如横杠
localparam OCTAVE_DOWN_PATTERN  = 7'b0111001; // D (代表降八度或练习错误) - 可以自定义，例如箭头向下

// Decoded segment data for each display position
// 每个显示位置解码后的段数据
reg [6:0] seg_data_suffix;     // For SEG0 (后缀或'P')
reg [6:0] seg_data_scrolled_notes [1:6]; // Array for SEG1-SEG6 data (用于SEG1-SEG6数据的数组)
reg [6:0] seg_data_octave;     // For SEG7 (八度或反馈)

// Combinational logic to decode note ID to segment pattern
// (组合逻辑：将音符ID解码为段码模式)
// 输入 note_id: 0为空白, 1=C(显示1), 2=D(显示2)... 7=B(显示7)
function [6:0] decode_note_to_segments (input [2:0] note_id);
    case (note_id)
        3'd1: decode_note_to_segments = PATTERN_1; // C
        3'd2: decode_note_to_segments = PATTERN_2; // D
        3'd3: decode_note_to_segments = PATTERN_3; // E
        3'd4: decode_note_to_segments = PATTERN_4; // F
        3'd5: decode_note_to_segments = PATTERN_5; // G
        3'd6: decode_note_to_segments = PATTERN_6; // A
        3'd7: decode_note_to_segments = PATTERN_7; // B
        default: decode_note_to_segments = PATTERN_BLANK; // 包括 0 (空白)
    endcase
endfunction

// Decoder for Semitone Suffix (SEG0) or Practice Indicator 'P'
// (SEG0的半音后缀 或 练习指示符'P' 的解码器)
always @(*) begin
    if (!semitone_display_active_flag) begin // 如果SEG0显示未激活
        seg_data_suffix = PATTERN_BLANK;
    end else begin
        case (semitone_type_in)
            2'b01:  seg_data_suffix = PATTERN_H;    // 升号 #
            2'b10:  seg_data_suffix = PATTERN_b;    // 降号 b
            2'b11:  seg_data_suffix = PATTERN_P;    // 练习模式 'P' (假设你的顶层模块传2'b11代表P)
            default: seg_data_suffix = PATTERN_BLANK; // 其他 (如2'b00基础音) 显示空白
        endcase
    end
end

// Decoder for Scrolled Notes/Practice Data (SEG1-SEG6)
// (SEG1-SEG6的滚动音符/练习数据的解码器)
// integer k; // 如果用Verilog-2001或更早标准，需要在此声明
always @(*) begin
    // Verilog-2001 及以上版本可以直接在 for 循环中声明 k
    // for (integer k = 1; k <= 6; k = k + 1) begin
    //    seg_data_scrolled_notes[k] = decode_note_to_segments( (k==1)? scrolled_note_seg1_in : ... ); // 这样做比较麻烦
    // end
    // 直接赋值更清晰
    seg_data_scrolled_notes[1] = decode_note_to_segments(scrolled_note_seg1_in);
    seg_data_scrolled_notes[2] = decode_note_to_segments(scrolled_note_seg2_in);
    seg_data_scrolled_notes[3] = decode_note_to_segments(scrolled_note_seg3_in);
    seg_data_scrolled_notes[4] = decode_note_to_segments(scrolled_note_seg4_in);
    seg_data_scrolled_notes[5] = decode_note_to_segments(scrolled_note_seg5_in);
    seg_data_scrolled_notes[6] = decode_note_to_segments(scrolled_note_seg6_in);
end

// Decoder for Octave/Practice Feedback (SEG7)
// (SEG7的八度/练习反馈的解码器)
always @(*) begin
    if (octave_up_active && !octave_down_active) seg_data_octave = OCTAVE_UP_PATTERN;       // 升八度或练习正确
    else if (!octave_up_active && octave_down_active) seg_data_octave = OCTAVE_DOWN_PATTERN; // 降八度或练习错误
    else seg_data_octave = OCTAVE_NORMAL_PATTERN;                                            // 普通八度或无反馈
end

// Muxing Logic for 8 digits (SEG0 to SEG7) - Dynamic Scanning
// (8位数码管 (SEG0-SEG7) 的多路复用逻辑 - 动态扫描)
localparam NUM_DISPLAY_SLOTS = 8; // 显示位数
// 扫描频率控制: 50MHz / MUX_COUNT_MAX_PER_DIGIT = 单个位选通时间倒数
// 整个8位数码管刷新一遍的时间 = 8 * MUX_COUNT_MAX_PER_DIGIT / 50MHz
// MUX_COUNT_MAX_PER_DIGIT = 104000 -> 2.08ms per digit -> 8 * 2.08ms = 16.64ms per refresh -> ~60Hz refresh rate
localparam MUX_COUNT_MAX_PER_DIGIT = 50000; // 修改此值可以调整刷新率，例如50000对应1ms每位，总刷新率125Hz

reg [$clog2(MUX_COUNT_MAX_PER_DIGIT)-1:0] mux_counter_reg; // 多路复用扫描计数器
reg [2:0] current_digit_slot_reg; // 当前选通的数码管位 (0-7)

initial begin
    seg_a = 1'b0; seg_b = 1'b0; seg_c = 1'b0; seg_d = 1'b0; // 初始化段选为灭
    seg_e = 1'b0; seg_f = 1'b0; seg_g = 1'b0; seg_dp = 1'b0; // dp不使用，但也初始化
    digit_selects = 8'hFF; // 初始化位选全灭 (假设共阴数码管，位选高电平有效，则00有效，FF无效；若位选低有效，则FF有效，00无效。根据你的板子调整，你的代码是高电平有效)
                            // 你的代码中 digit_selects[i] <= 1'b1; 表示选通，所以初始应该是 digit_selects = 8'h00; (全不选通)
    mux_counter_reg = 0;
    current_digit_slot_reg = 3'd0;
end

always @(posedge clk or negedge rst_n) begin // 时钟上升沿或复位下降沿触发
    if (!rst_n) begin // 复位
        mux_counter_reg <= 0;
        current_digit_slot_reg <= 3'd0;
        digit_selects <= 8'h00; // 复位时所有位都不选通
        {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= PATTERN_BLANK; // 段选全灭
        seg_dp <= 1'b0; // 小数点灭
    end else begin
        // 默认情况下先关闭所有段和位选，防止上一个周期的鬼影
        // (在case语句之前执行，确保在切换位选时，段数据不会错误地短暂显示在旧的位上)
        digit_selects <= 8'h00; // 关闭所有位选
        {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= PATTERN_BLANK; // 段数据清空
        seg_dp <= 1'b0;

        if (mux_counter_reg >= MUX_COUNT_MAX_PER_DIGIT - 1) begin // 当扫描计数器达到每个位的显示时间
            mux_counter_reg <= 0; // 计数器清零
            // current_digit_slot_reg <= (current_digit_slot_reg == NUM_DISPLAY_SLOTS - 1) ? 3'd0 : current_digit_slot_reg + 1'b1;
            if (current_digit_slot_reg == NUM_DISPLAY_SLOTS - 1) begin // 如果已扫描到最后一位
                current_digit_slot_reg <= 3'd0; // 回到第一位
            end else begin
                current_digit_slot_reg <= current_digit_slot_reg + 1'b1; // 扫描下一位
            end
        end else begin
            mux_counter_reg <= mux_counter_reg + 1; // 计数器加1
        end

        // 根据当前选通的数码管位 (current_digit_slot_reg) 选择要显示的段数据和位选信号
        case (current_digit_slot_reg)
            3'd0: begin // SEG0: Semitone Suffix / Practice 'P'
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_suffix; // 显示SEG0的数据
                digit_selects[0] <= 1'b1; // 选通SEG0 (假设高电平有效)
            end
            3'd1: begin // SEG1: Scrolled Note 1 / Practice Data
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_scrolled_notes[1];
                digit_selects[1] <= 1'b1; // 选通SEG1
            end
            3'd2: begin // SEG2: Scrolled Note 2 / Practice Data
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_scrolled_notes[2];
                digit_selects[2] <= 1'b1;
            end
            3'd3: begin // SEG3: Scrolled Note 3 / Practice Data
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_scrolled_notes[3];
                digit_selects[3] <= 1'b1;
            end
            3'd4: begin // SEG4: Scrolled Note 4 / Practice Data
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_scrolled_notes[4];
                digit_selects[4] <= 1'b1;
            end
            3'd5: begin // SEG5: Scrolled Note 5 / Practice Data
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_scrolled_notes[5];
                digit_selects[5] <= 1'b1;
            end
            3'd6: begin // SEG6: Scrolled Note 6 / Practice Data
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_scrolled_notes[6];
                digit_selects[6] <= 1'b1;
            end
            3'd7: begin // SEG7: Octave Status / Practice Feedback
                {seg_g, seg_f, seg_e, seg_d, seg_c, seg_b, seg_a} <= seg_data_octave;
                digit_selects[7] <= 1'b1; // 选通SEG7
            end
            default: digit_selects <= 8'h00; // 理论上不会到这里，保险起见
        endcase
        // seg_dp <= 1'b0; // 小数点不使用，保持熄灭 (可以根据需要点亮，例如在特定模式)
    end
end
endmodule
```

### piano_recorder.v

```verilog
// File: piano_recorder.v
// Module for recording and playing back piano key presses.

// File: piano_recorder.v
// Module for recording and playing back piano key presses.
// 文件名: piano_recorder.v
// (用于录制和回放钢琴按键的模块)

module piano_recorder #(
    parameter CLK_FREQ_HZ      = 50_000_000, // System clock frequency (系统时钟频率)
    parameter RECORD_INTERVAL_MS = 20,       // Interval for sampling/playback (e.g., 20ms) (采样/回放的时间间隔，例如20毫秒)
    parameter MAX_RECORD_SAMPLES = 512,    // Max number of samples to record (最大录制采样点数)
    parameter KEY_ID_BITS      = 3,        // Bits for key ID (0 for none, 1-N for keys) - Default, will be overridden
                                           // (按键ID的位数(0表示无, 1-N表示按键) - 默认值, 会被顶层模块覆盖)
                                           // 你的顶层模块传入的是 RECORDER_KEY_ID_BITS = 4
    parameter OCTAVE_BITS      = 2         // Bits for octave state (00:normal, 01:up, 10:down)
                                           // (八度状态的位数 (00:普通, 01:升, 10:降))
) (
    input clk,
    input rst_n, // Active low reset (低电平有效复位)

    // Control signals (expected to be debounced externally)
    // (控制信号，期望外部已消抖)
    input record_active_level,     // High when record button (e.g., SW16) is held down
                                   // (当录音按钮 (例如SW16) 按下时为高电平)
    input playback_start_pulse,    // A single clock cycle pulse to start playback (e.g., on SW17 press)
                                   // (启动回放的单时钟周期脉冲 (例如SW17按下时))

    // Inputs from main piano logic (live playing)
    // (来自主钢琴逻辑的输入 - 实时演奏)
    input [KEY_ID_BITS-1:0] live_key_id,       // Current key ID being pressed (0 if none) (当前按下的按键ID (0表示无))
    input live_key_is_pressed,                 // Flag: is a live key currently pressed? (标志：当前是否有实时按键按下?)
    input live_octave_up,                      // Flag: is live octave up active? (标志：实时升八度是否激活?)
    input live_octave_down,                    // Flag: is live octave down active? (标志：实时降八度是否激活?)

    // Outputs to drive buzzer and display during playback
    // (回放期间驱动蜂鸣器和显示的输出)
    output reg [KEY_ID_BITS-1:0] playback_key_id,       // 回放的按键ID
    output reg playback_key_is_pressed,                 // 回放时是否有按键按下
    output wire playback_octave_up,   // Changed from reg to wire (回放时的升八度信号，从reg改为wire)
    output wire playback_octave_down, // Changed from reg to wire (回放时的降八度信号，从reg改为wire)

    // Status outputs (optional, for LEDs or debugging)
    // (状态输出，可选，用于LED或调试)
    output reg is_recording, // 是否正在录音
    output reg is_playing    // 是否正在播放
);

// --- Derived Parameters ---
// --- 派生参数 ---
localparam RECORD_INTERVAL_CYCLES = (RECORD_INTERVAL_MS * (CLK_FREQ_HZ / 1000)); // Cycles per interval (每个采样/回放间隔的时钟周期数)
localparam ADDR_WIDTH = $clog2(MAX_RECORD_SAMPLES); // Width for memory address (存储器地址位宽)
// Data format per sample: {octave_state[1:0], key_is_pressed (1), key_id[KEY_ID_BITS-1:0]}
// (每个采样点的数据格式: {八度状态[1:0], 按键是否按下(1位), 按键ID[KEY_ID_BITS-1:0]})
localparam DATA_WIDTH = OCTAVE_BITS + 1 + KEY_ID_BITS; // 每个采样点的数据总位宽

// --- Memory for Recording ---
// --- 用于录音的存储器 ---
// Quartus will infer this as RAM (M9K blocks if available and appropriate size)
// (Quartus会将其推断为RAM，如果可用且大小合适，则使用M9K块)
reg [DATA_WIDTH-1:0] recorded_data_memory [0:MAX_RECORD_SAMPLES-1]; // 录音数据存储器
reg [ADDR_WIDTH-1:0] record_write_ptr;    // Points to the next empty slot for recording (指向下一个用于录音的空闲位置 - 写指针)
reg [ADDR_WIDTH-1:0] playback_read_ptr;   // Points to the current sample to play (指向当前要播放的采样点 - 读指针)
reg [ADDR_WIDTH-1:0] last_recorded_ptr;   // Stores the address of the last valid recorded sample + 1 (i.e., length)
                                          // (存储最后一个有效录制采样点的地址+1，即录制长度)

// --- Timers and Counters ---
// --- 定时器和计数器 ---
reg [$clog2(RECORD_INTERVAL_CYCLES)-1:0] sample_timer_reg; // 采样/回放间隔定时器

// --- State Machine ---
// --- 状态机 ---
localparam S_IDLE      = 2'b00; // 空闲状态
localparam S_RECORDING = 2'b01; // 录音状态
localparam S_PLAYBACK  = 2'b10; // 回放状态
reg [1:0] current_state_reg;  // 当前状态寄存器

// --- Internal signals for octave encoding/decoding ---
// --- 用于八度编码/解码的内部信号 ---
wire [OCTAVE_BITS-1:0] live_octave_encoded; // 实时八度编码后的值
assign live_octave_encoded = (live_octave_up && !live_octave_down) ? 2'b01 :      // Up (升八度)
                             (!live_octave_up && live_octave_down) ? 2'b10 :      // Down (降八度)
                             2'b00;                                              // Normal (or both pressed) (普通八度或同时按下)

reg [OCTAVE_BITS-1:0] playback_octave_encoded; // Changed from wire to reg (回放八度编码后的值，从wire改为reg，因为在状态机中被赋值)
assign playback_octave_up   = (playback_octave_encoded == 2'b01); // 回放时升八度输出
assign playback_octave_down = (playback_octave_encoded == 2'b10); // 回放时降八度输出

initial begin // 初始化块
    is_recording = 1'b0;
    is_playing = 1'b0;
    playback_key_id = {KEY_ID_BITS{1'b0}}; // Ensure correct width for 0 (确保0有正确的位宽)
    playback_key_is_pressed = 1'b0;
    playback_octave_encoded = {OCTAVE_BITS{1'b0}}; // Initialize to normal (初始化为普通八度)
    current_state_reg = S_IDLE;
    record_write_ptr = {ADDR_WIDTH{1'b0}};
    playback_read_ptr = {ADDR_WIDTH{1'b0}};
    last_recorded_ptr = {ADDR_WIDTH{1'b0}};
    sample_timer_reg = 0; // Assuming its width is sufficient for 0 (假设其位宽足以容纳0)
end

always @(posedge clk or negedge rst_n) begin // 时序逻辑块
    if (!rst_n) begin // 复位逻辑
        is_recording <= 1'b0;
        is_playing <= 1'b0;
        playback_key_id <= {KEY_ID_BITS{1'b0}};
        playback_key_is_pressed <= 1'b0;
        playback_octave_encoded <= {OCTAVE_BITS{1'b0}}; // Reset to normal (复位到普通八度)
        current_state_reg <= S_IDLE;
        record_write_ptr <= {ADDR_WIDTH{1'b0}};
        playback_read_ptr <= {ADDR_WIDTH{1'b0}};
        last_recorded_ptr <= {ADDR_WIDTH{1'b0}};
        sample_timer_reg <= 0;
    end else begin // 正常操作
        // Default actions, can be overridden by states
        // (默认操作，可以被状态覆盖)
        if (current_state_reg != S_PLAYBACK) begin // Only reset playback outputs if not actively playing
                                                  // (仅当非播放状态时才重置回放输出)
             playback_key_is_pressed <= 1'b0;
             playback_key_id <= {KEY_ID_BITS{1'b0}};
             playback_octave_encoded <= {OCTAVE_BITS{1'b0}};
        end
        if (current_state_reg != S_RECORDING) begin // 非录音状态
            is_recording <= 1'b0;
        end
        if (current_state_reg != S_PLAYBACK) begin // 非播放状态
            is_playing <= 1'b0;
        end

        // 状态机
        case (current_state_reg)
            S_IDLE: begin // 空闲状态
                sample_timer_reg <= 0; // Reset timer in IDLE (在IDLE状态重置定时器)

                if (record_active_level) begin // SW16 pressed to start recording (SW16按下，开始录音)
                    current_state_reg <= S_RECORDING;      // 进入录音状态
                    record_write_ptr <= {ADDR_WIDTH{1'b0}}; // Start recording from the beginning (从头开始录音)
                    is_recording <= 1'b1;                  // 设置正在录音标志
                    last_recorded_ptr <= {ADDR_WIDTH{1'b0}}; // Reset length of current recording (重置当前录音长度)
                end else if (playback_start_pulse && last_recorded_ptr > 0) begin // SW17 pressed and there's something to play
                                                                               // (SW17按下并且有内容可播放)
                    current_state_reg <= S_PLAYBACK;         // 进入回放状态
                    playback_read_ptr <= {ADDR_WIDTH{1'b0}}; // Start playback from the beginning (从头开始播放)
                    is_playing <= 1'b1;                      // 设置正在播放标志
                    sample_timer_reg <= RECORD_INTERVAL_CYCLES -1; // Preload to play first sample immediately
                                                                // (预加载以立即播放第一个采样点，因为定时器会在下一个周期才比较)
                end
            end

            S_RECORDING: begin // 录音状态
                is_recording <= 1'b1; // Keep is_recording high (保持录音标志为高)
                if (!record_active_level || record_write_ptr >= MAX_RECORD_SAMPLES) begin // SW16 released or memory full
                                                                                       // (SW16释放或存储器已满)
                    current_state_reg <= S_IDLE;            // 返回空闲状态
                    // is_recording will be set to 0 by default action or IDLE entry
                    // (is_recording 会被默认操作或IDLE入口的逻辑置0)
                    last_recorded_ptr <= record_write_ptr; // Save how much we recorded (number of samples)
                                                          // (保存录制了多少采样点)
                end else begin // 继续录音
                    if (sample_timer_reg == RECORD_INTERVAL_CYCLES - 1) begin // 达到一个采样间隔
                        sample_timer_reg <= 0; // 重置采样间隔定时器
                        // Store: {octave_state[1:0], live_key_is_pressed, live_key_id[KEY_ID_BITS-1:0]}
                        // (存储: {八度状态, 是否按下, 按键ID})
                        recorded_data_memory[record_write_ptr] <= {live_octave_encoded, live_key_is_pressed, live_key_id};

                        if (record_write_ptr < MAX_RECORD_SAMPLES - 1 ) begin // 如果存储器未满
                           record_write_ptr <= record_write_ptr + 1; // 写指针加1
                        end else begin // Memory is now full (last slot used) (存储器已满，最后一个槽位已用)
                           current_state_reg <= S_IDLE; // 返回空闲状态
                           last_recorded_ptr <= MAX_RECORD_SAMPLES; // Record that memory is full (记录存储器已满)
                        end
                    end else begin // 未达到采样间隔
                        sample_timer_reg <= sample_timer_reg + 1; // 采样间隔定时器加1
                    end
                end
            end

            S_PLAYBACK: begin // 回放状态
                is_playing <= 1'b1; // Keep is_playing high (保持播放标志为高)
                // Check if playback should stop
                // (检查是否应停止播放)
                if (playback_read_ptr >= last_recorded_ptr || playback_read_ptr >= MAX_RECORD_SAMPLES ) begin // 读指针达到录制末尾或存储器末尾
                    current_state_reg <= S_IDLE; // 返回空闲状态
                    // is_playing and playback outputs will be reset by default action or IDLE entry
                    // (is_playing 和 playback 输出会被默认操作或IDLE入口的逻辑重置)
                end else begin // 继续播放
                    if (sample_timer_reg == RECORD_INTERVAL_CYCLES - 1) begin // 达到一个回放间隔
                        sample_timer_reg <= 0; // 重置回放间隔定时器
                        // Read data: {octave_state, key_pressed, key_id}
                        // (读取数据: {八度状态, 是否按下, 按键ID})
                        {playback_octave_encoded, playback_key_is_pressed, playback_key_id} <= recorded_data_memory[playback_read_ptr];

                        if (playback_read_ptr < MAX_RECORD_SAMPLES - 1 && playback_read_ptr < last_recorded_ptr -1 ) begin // 如果未到录制数据末尾
                            playback_read_ptr <= playback_read_ptr + 1; // 读指针加1
                        end else begin // Reached end of data to play or last valid sample (已到达要播放的数据末尾或最后一个有效采样点)
                            current_state_reg <= S_IDLE; // 返回空闲状态
                        end
                    end else begin // 未达到回放间隔
                        sample_timer_reg <= sample_timer_reg + 1; // 回放间隔定时器加1
                    end
                end
            end
            default: current_state_reg <= S_IDLE; // 默认状态，防止意外，返回空闲
        endcase
    end
end
endmodule
```

### song_player.v

```verilog
// File: song_player.v
// (歌曲播放器模块)
module song_player #(
    parameter CLK_FREQ_HZ = 50_000_000, // 系统时钟频率
    parameter KEY_ID_BITS = 4,         // For C, C#, D ... B (12 notes + REST) (用于C,C#...B (12个音符+休止符))
    parameter OCTAVE_BITS = 2          // To represent Low, Middle, High octaves (用于表示低、中、高八度)
) (
    input clk,
    input rst_n,                       // 低电平有效复位
    input play_active_level,           // 高电平播放，低电平停止 (播放激活电平)

    output reg [KEY_ID_BITS-1:0] song_key_id,      // 歌曲播放的按键ID
    output reg song_key_is_pressed,                // 歌曲播放时是否有按键按下
    output reg song_octave_up_feed,    // New output for octave up (新增的升八度输出)
    output reg song_octave_down_feed,  // New output for octave down (新增的降八度输出)
    output reg is_song_playing         // 歌曲正在播放的状态指示
);

    // --- 音符定义 (KEY_ID 1-12) ---
    // (你的顶层模块中将音符ID映射为1-12，这里保持一致)
    localparam NOTE_C   = 4'd1; localparam NOTE_CS  = 4'd8;  // C, C#
    localparam NOTE_D   = 4'd2; localparam NOTE_DS  = 4'd9;  // D, D# (Eb)
    localparam NOTE_E   = 4'd3;                             // E
    localparam NOTE_F   = 4'd4; localparam NOTE_FS  = 4'd10; // F, F#
    localparam NOTE_G   = 4'd5; localparam NOTE_GS  = 4'd11; // G, G# (Ab)
    localparam NOTE_A   = 4'd6; localparam NOTE_AS  = 4'd12; // A, A# (Bb)
    localparam NOTE_B   = 4'd7;                             // B
    localparam REST     = 4'd0; // 休止符 (ID为0)

    // --- 八度定义 ---
    localparam OCTAVE_LOW  = 2'b10; // Signal to activate octave_down (激活降八度的信号)
    localparam OCTAVE_MID  = 2'b00; // Signal for normal (middle) octave (普通(中)八度的信号)
    localparam OCTAVE_HIGH = 2'b01; // Signal to activate octave_up (激活升八度的信号)

    // --- 时长和乐谱数据定义 ---
    localparam DURATION_BITS = 4;     // 用于表示时长单位的位数
    // 乐谱数据宽度 = 八度位 + 按键ID位 + 时长位
    localparam SONG_DATA_WIDTH = OCTAVE_BITS + KEY_ID_BITS + DURATION_BITS;

    // !!! REPLACE THIS WITH THE CORRECT SONG_LENGTH FROM YOUR MIDI CONVERSION !!!
    // (!!! 请将此替换为您MIDI转换得到的正确SONG_LENGTH !!!)
    localparam SONG_LENGTH = 232; // EXAMPLE - Use the actual length from your transcription (示例 - 使用你转录的实际长度)

    // --- 节拍和基础时长单位 ---
    // !!! ENSURE THIS MATCHES THE VALUE USED FOR YOUR MIDI TRANSCRIPTION !!!
    // (!!! 确保这与您MIDI转录所用的值匹配 !!!)
    localparam BASIC_NOTE_DURATION_MS = 70; // 基础音符时长 (毫秒)
    localparam BASIC_NOTE_DURATION_CYCLES = (BASIC_NOTE_DURATION_MS * (CLK_FREQ_HZ / 1000)); // 基础音符时长的时钟周期数
    localparam MAX_DURATION_UNITS_VAL = (1 << DURATION_BITS) - 1; // 最大时长单位值 (例如4位时是15)

    // --- 状态机定义 ---
    localparam S_IDLE   = 1'b0; // 空闲状态
    localparam S_PLAYING= 1'b1; // 播放状态

    // --- 内部寄存器声明 ---
    // 存储乐谱的ROM (Read-Only Memory)
    reg [SONG_DATA_WIDTH-1:0] song_rom [0:SONG_LENGTH-1];

    reg [$clog2(SONG_LENGTH)-1:0] current_note_index; // 当前音符在乐谱中的索引
    // 音符时长计时器，需要能计时到最长音符的周期数
    reg [$clog2(BASIC_NOTE_DURATION_CYCLES * MAX_DURATION_UNITS_VAL + 1)-1:0] note_duration_timer;
    reg [DURATION_BITS-1:0] current_note_duration_units; // 当前音符的时长单位
    reg [KEY_ID_BITS-1:0] current_note_id_from_rom;    // 从ROM读取的当前音符ID
    reg [OCTAVE_BITS-1:0] current_octave_code_from_rom; // 从ROM读取的当前八度编码
    reg state; // 当前状态 (S_IDLE 或 S_PLAYING)
    reg play_active_level_prev; // 上一周期播放激活电平 (用于检测上升沿)

    // ########################################################################## //
    // #                                                                        # //
    // #    <<<<< REPLACE THE ENTIRE 'initial begin ... end' BLOCK BELOW >>>>>  # //
    // #    <<<<< WITH THE ONE CONTAINING YOUR TRANSCRIBED song_rom DATA >>>>>  # //
    // #    (<<<<< 请将下面的整个 'initial begin ... end' 块替换为 >>>>> )      # //
    // #    (<<<<< 包含您转录的 song_rom 数据的那个             >>>>> )      # //
    // #                                                                        # //
    // ########################################################################## //
    initial begin
        // THIS IS A PLACEHOLDER - REPLACE IT WITH YOUR ACTUAL SONG_ROM INITIALIZATION
        // (这是一个占位符 - 请用您实际的 SONG_ROM 初始化替换它)
        // Example:
        // song_rom[0]  = {OCTAVE_LOW,  NOTE_D,   4'd2};
        // song_rom[1]  = {OCTAVE_MID,  REST,     4'd2};
        // ... many more lines ...
        // song_rom[SONG_LENGTH-1] = {OCTAVE_MID, REST, 4'd4}; // Last note or rest

        // Ensure all song_rom entries are initialized, especially if your
        // transcription doesn't fill the entire SONG_LENGTH.
        // (确保所有 song_rom 条目都已初始化，特别是如果您的转录没有填满整个 SONG_LENGTH。)
        integer i;
        for (i = 0; i < SONG_LENGTH; i = i + 1) begin // 遍历初始化ROM
            // If you provide all entries explicitly, this loop can be minimal or removed
            // (如果您显式提供了所有条目，则此循环可以最小化或删除)
            // If your transcription is shorter than SONG_LENGTH, fill the rest:
            // (如果您的转录比 SONG_LENGTH 短，请填充其余部分：)
            if (i >= 232) begin // Assuming your transcription has 232 entries (0 to 231) (假设你的转录有232个条目)
                 song_rom[i] = {OCTAVE_MID, REST, 4'd1}; // Default fill (默认填充)
            end
            // If your transcription has fewer entries than the example (232), adjust the 'if' condition.
            // (如果您的转录条目少于示例(232)，请调整'if'条件。)
            // Or, just ensure your transcription defines ALL song_rom[0] through song_rom[SONG_LENGTH-1].
            // (或者，只需确保您的转录定义了所有 song_rom[0] 到 song_rom[SONG_LENGTH-1]。)
        end
        // Make sure song_rom[0] to song_rom[231] (or however many entries you have) are defined
        // by the MIDI transcription part. For example:
        // (确保 song_rom[0] 到 song_rom[231] (或您拥有的任意数量的条目) 由MIDI转录部分定义。例如：)
        song_rom[0]  = {OCTAVE_LOW,  NOTE_D,   4'd2}; // MIDI 50 (D3), Dur: 0.1429s (数据格式: {八度, 音符ID, 时长单位})
        song_rom[1]  = {OCTAVE_MID,  REST,     4'd2}; // Rest dur: 0.1413s
        song_rom[2]  = {OCTAVE_LOW,  NOTE_G,   4'd2}; // MIDI 55 (G3), Dur: 0.1429s
        // ... (The 232 lines of song_rom data you generated previously) ...
        // (之前生成的232行song_rom数据)
        song_rom[231] = {OCTAVE_LOW, NOTE_FS,  4'd8}; // MIDI 54 (F#3), Dur: 0.5715s

        // Initialize outputs and internal state registers (This part should be AT THE END of your initial block)
        // (初始化输出和内部状态寄存器 (这部分应在您的initial块的末尾))
        song_key_id = {KEY_ID_BITS{1'b0}};
        song_key_is_pressed = 1'b0;
        song_octave_up_feed = 1'b0;
        song_octave_down_feed = 1'b0;
        is_song_playing = 1'b0;
        state = S_IDLE;
        current_note_index = 0;
        note_duration_timer = 0;
        current_note_duration_units = 0;
        current_note_id_from_rom = {KEY_ID_BITS{1'b0}};
        current_octave_code_from_rom = OCTAVE_MID;
        play_active_level_prev = 1'b0;
    end
    // ################# END OF BLOCK TO BE REPLACED ######################## //
    // ################# (需要替换的块结束) ######################## //

    // --- 主要状态机和逻辑 ---
    always @(posedge clk or negedge rst_n) begin // 时序逻辑
        if (!rst_n) begin // 复位状态
            song_key_id <= {KEY_ID_BITS{1'b0}};
            song_key_is_pressed <= 1'b0;
            song_octave_up_feed <= 1'b0;
            song_octave_down_feed <= 1'b0;
            is_song_playing <= 1'b0;
            state <= S_IDLE;
            current_note_index <= 0;
            note_duration_timer <= 0;
            current_note_duration_units <= 0;
            current_note_id_from_rom <= {KEY_ID_BITS{1'b0}};
            current_octave_code_from_rom <= OCTAVE_MID;
            play_active_level_prev <= 1'b0;
        end else begin // 正常操作
            play_active_level_prev <= play_active_level; // Store current key level for next cycle edge detection (存储当前按键电平，用于下一周期边沿检测)

            // Top priority stop condition: if play button becomes low and currently playing, stop immediately
            // (最高优先级停止条件：如果播放按钮变为低电平且当前正在播放，则立即停止)
            if (!play_active_level && state == S_PLAYING) begin
                state <= S_IDLE;             // 进入空闲状态
                song_key_is_pressed <= 1'b0; // Silence (静音)
                song_octave_up_feed <= 1'b0;   // Reset on stop (停止时重置)
                song_octave_down_feed <= 1'b0; // Reset on stop (停止时重置)
                is_song_playing <= 1'b0;   // Update status (更新状态)
            end

            // 状态机逻辑
            case (state)
                S_IDLE: begin // 空闲状态
                    song_key_is_pressed <= 1'b0; // Ensure silence in IDLE (在IDLE状态确保静音)
                    song_octave_up_feed <= 1'b0;
                    song_octave_down_feed <= 1'b0;
                    is_song_playing <= 1'b0;   // Ensure playing status is false in IDLE (在IDLE状态确保播放状态为否)

                    // If play button is pressed (detect rising edge)
                    // (如果播放按钮按下 - 检测上升沿)
                    if (play_active_level && !play_active_level_prev) begin
                        if (SONG_LENGTH > 0) begin // Only play if there's a song (仅当有歌曲时才播放)
                            state <= S_PLAYING;     // Enter playing state (进入播放状态)
                            current_note_index <= 0;  // Play from the beginning of the score (从乐谱开头播放)
                            // Read the first note
                            // (读取第一个音符)
                            {current_octave_code_from_rom, current_note_id_from_rom, current_note_duration_units} = song_rom[0];

                            song_key_id <= current_note_id_from_rom; // 输出音符ID
                            song_key_is_pressed <= (current_note_id_from_rom != REST); // 如果不是休止符，则按下
                            song_octave_up_feed <= (current_octave_code_from_rom == OCTAVE_HIGH); // 设置升八度
                            song_octave_down_feed <= (current_octave_code_from_rom == OCTAVE_LOW); // 设置降八度

                            note_duration_timer <= 0; // Reset note duration timer (重置音符时长计时器)
                            is_song_playing <= 1'b1;  // Set playing status to true (设置播放状态为是)
                        end
                    end
                end // S_IDLE 结束

                S_PLAYING: begin // 播放状态
                    // Only continue processing playback logic if the play button is still held down
                    // (只有当播放按钮仍然按下时才继续处理播放逻辑)
                    if (play_active_level) begin
                        is_song_playing <= 1'b1; // Keep playing status true (保持播放状态为是)

                        if (current_note_duration_units == 0) begin // If current note has 0 duration (should ideally not happen from good ROM)
                                                                    // (如果当前音符时长为0 - 理想情况下好的ROM不应发生)
                            // Defensive: skip to next note or stop if at end
                            // (防御性编程：跳到下一个音符，如果到末尾则停止)
                            if (current_note_index < SONG_LENGTH - 1) begin // 如果不是最后一个音符
                                current_note_index <= current_note_index + 1; // 指向下一个音符
                                {current_octave_code_from_rom, current_note_id_from_rom, current_note_duration_units} = song_rom[current_note_index + 1];
                                song_key_id <= current_note_id_from_rom;
                                song_key_is_pressed <= (current_note_id_from_rom != REST);
                                song_octave_up_feed <= (current_octave_code_from_rom == OCTAVE_HIGH);
                                song_octave_down_feed <= (current_octave_code_from_rom == OCTAVE_LOW);
                                note_duration_timer <= 0;
                            end else begin // Already at the last note, song finished (or invalid duration on last note)
                                           // (已经是最后一个音符，歌曲结束 - 或最后一个音符时长无效)
                                state <= S_IDLE; // 返回空闲
                            end
                        end else if (note_duration_timer >= (BASIC_NOTE_DURATION_CYCLES * current_note_duration_units) - 1'b1 ) begin // Current note's duration is over
                                                                                                                                    // (当前音符播放时长已到)
                            // Switch to the next note
                            // (切换到下一个音符)
                            if (current_note_index < SONG_LENGTH - 1) begin // If not the last note (如果不是最后一个音符)
                                current_note_index <= current_note_index + 1; // 指向下一个音符
                                // Read next note including octave for the *next* cycle
                                // (为下一个周期读取下一个音符，包括八度)
                                {current_octave_code_from_rom, current_note_id_from_rom, current_note_duration_units} = song_rom[current_note_index + 1]; // This reads for the upcoming note (这读取的是即将播放的音符)

                                song_key_id <= current_note_id_from_rom;
                                song_key_is_pressed <= (current_note_id_from_rom != REST);
                                song_octave_up_feed <= (current_octave_code_from_rom == OCTAVE_HIGH);
                                song_octave_down_feed <= (current_octave_code_from_rom == OCTAVE_LOW);
                                note_duration_timer <= 0; // Reset timer for the new note (为新音符重置计时器)
                            end else begin // Already at the last note, song finished (已经是最后一个音符，歌曲结束)
                                state <= S_IDLE; // 返回空闲
                                // Optional: keep the last note sounding until button release or explicitly silence here.
                                // (可选：保持最后一个音符发声直到按钮释放，或在此处明确静音。)
                                // Current logic will go to IDLE, which silences.
                                // (当前逻辑会进入IDLE，从而静音。)
                            end
                        end else begin // Current note is not finished playing (当前音符还未播完)
                            note_duration_timer <= note_duration_timer + 1; // Continue timing (继续计时)
                            // Outputs (key_id, is_pressed, octave_feeds) remain for the current note
                            // (输出 (key_id, is_pressed, octave_feeds) 保持为当前音符的值)
                        end
                    end else begin // play_active_level 变为低电平
                        // If play_active_level becomes low while in S_PLAYING
                        // (如果在S_PLAYING状态时play_active_level变为低)
                        state <= S_IDLE;          // Force back to IDLE (强制回到IDLE)
                        song_key_is_pressed <= 1'b0; // Silence (静音)
                        song_octave_up_feed <= 1'b0;
                        song_octave_down_feed <= 1'b0;
                        is_song_playing <= 1'b0;    // Update status (更新状态)
                    end
                end // S_PLAYING 结束

                default: state <= S_IDLE; // Unexpected state, go to IDLE (意外状态则回到IDLE)
            endcase // case(state) 结束
        end // else (if !rst_n) 结束
    end // always 结束
endmodule // 模块结束
```

### scrolling_display_buffer.v

```verilog
// File: scrolling_display_buffer.v
// Module to manage a 6-digit scrolling buffer for note display (SEG1-SEG6)
// 文件名: scrolling_display_buffer.v
// (用于管理6位数码管滚动显示的缓冲区模块 - 对应物理数码管SEG1-SEG6)

module scrolling_display_buffer (
    input clk,
    input rst_n, // 低电平有效复位

    input new_note_valid_pulse,         // Single clock pulse when a new valid note is pressed
                                        // (当新的有效音符被按下时产生的单时钟周期脉冲)
    input [2:0] current_base_note_id_in,  // The base note ID (1-7) of the new note (0 for blank)
                                        // (新音符的基础音符ID (1-7对应C-B, 0表示空白))

    output reg [2:0] display_data_seg1, // Data for physical SEG1 (rightmost of scrolling area)
                                        // (用于物理数码管SEG1的数据 - 滚动区域的最右边)
    output reg [2:0] display_data_seg2, // ...
    output reg [2:0] display_data_seg3,
    output reg [2:0] display_data_seg4,
    output reg [2:0] display_data_seg5,
    output reg [2:0] display_data_seg6  // Data for physical SEG6 (leftmost of scrolling area)
                                        // (用于物理数码管SEG6的数据 - 滚动区域的最左边)
);

// Internal buffer registers for 6 display segments (SEG1 to SEG6)
// (6个显示段 (SEG1-SEG6) 的内部缓冲寄存器)
// seg_buffer[0] corresponds to display_data_seg1 (rightmost of scrolling)
// (seg_buffer[0] 对应 display_data_seg1 - 滚动区域最右侧)
// seg_buffer[5] corresponds to display_data_seg6 (leftmost of scrolling)
// (seg_buffer[5] 对应 display_data_seg6 - 滚动区域最左侧)
reg [2:0] seg_buffer [0:5]; // Each element stores a 3-bit note ID (0 for blank)
                            // (每个元素存储一个3位的音符ID，0表示空白)

integer i; // Loop variable (循环变量)

initial begin // 初始化块
    display_data_seg1 = 3'd0; // 初始化所有输出为0 (空白)
    display_data_seg2 = 3'd0;
    display_data_seg3 = 3'd0;
    display_data_seg4 = 3'd0;
    display_data_seg5 = 3'd0;
    display_data_seg6 = 3'd0;
    for (i = 0; i < 6; i = i + 1) begin
        seg_buffer[i] = 3'd0; // Initialize buffer to blank (初始化缓冲区为空白)
    end
end

always @(posedge clk or negedge rst_n) begin // 时序逻辑: 处理缓冲区滚动
    if (!rst_n) begin // 复位
        // Reset all buffer positions to 0 (blank)
        // (将所有缓冲位置复位为0 - 空白)
        for (i = 0; i < 6; i = i + 1) begin
            seg_buffer[i] <= 3'd0;
        end
    end else begin // 正常操作
        if (new_note_valid_pulse) begin // 当有新的有效音符脉冲到来时
            // Scroll existing data: seg_buffer[5] (SEG6) <- seg_buffer[4] (SEG5), etc.
            // (滚动现有数据：seg_buffer[5](SEG6的显示内容) 接收 seg_buffer[4](SEG5的显示内容) 的值，依此类推)
            // The oldest data at seg_buffer[5] is shifted out.
            // (seg_buffer[5] 中最老的数据被移出)
            seg_buffer[5] <= seg_buffer[4]; // SEG6_data <--- SEG5_data
            seg_buffer[4] <= seg_buffer[3]; // SEG5_data <--- SEG4_data
            seg_buffer[3] <= seg_buffer[2]; // SEG4_data <--- SEG3_data
            seg_buffer[2] <= seg_buffer[1]; // SEG3_data <--- SEG2_data
            seg_buffer[1] <= seg_buffer[0]; // SEG2_data <--- SEG1_data

            // Load new note into the first position (SEG1)
            // (将新音符加载到第一个位置 - SEG1)
            seg_buffer[0] <= current_base_note_id_in; // SEG1_data <--- New Note (新音符)
        end
        // No else: if new_note_valid_pulse is not high, the buffer holds its value.
        // (无else：如果new_note_valid_pulse不为高，则缓冲区保持其值不变)
    end
end

// Assign buffer contents to outputs continuously
// (持续地将缓冲区内容赋给输出)
// (Combinational assignment from buffer regs to output regs for clarity,
// or could directly use seg_buffer[x] in seven_segment_controller if preferred as wires)
// (为清晰起见，从缓冲寄存器到输出寄存器的组合分配，
//  或者如果希望作为连线，则可以在 seven_segment_controller 中直接使用 seg_buffer[x])
// For direct output from registers, assign in the always block or use separate assigns.
// (要从寄存器直接输出，请在 always 块中分配或使用单独的 assign 语句。)
// To ensure outputs are also reset correctly and avoid latches if outputs were wires,
// it's cleaner to assign them from the registered buffer values.
// (为确保输出也能正确复位，并且如果输出是连线类型时避免产生锁存器，
//  从寄存的缓冲值分配它们是更清晰的做法。)

// We declared outputs as 'reg' and will assign them directly.
// (我们将输出声明为 'reg'，并将直接为它们赋值。)
// Let's ensure they are updated in the clocked block or from the buffer
// in a combinational way if they were wires.
// (让我们确保它们在时钟块中更新，或者如果它们是连线，则以组合方式从缓冲区更新。)
// Since outputs are 'reg', we update them based on the 'seg_buffer'.
// (由于输出是 'reg'，我们根据 'seg_buffer' 更新它们。)
// It's often good practice for module outputs driven by internal registers
// to be assigned combinatorially *from* those registers,
// or the outputs themselves are the registers.
// (通常好的做法是，由内部寄存器驱动的模块输出，应从这些寄存器进行组合赋值，
//  或者输出本身就是这些寄存器。)
// Here, we've made outputs 'reg', so we should assign them in an always block.
// (在这里，我们将输出设为 'reg'，所以我们应该在一个 always 块中为它们赋值。)
// A simpler way if outputs are 'reg' is to directly assign in the clocked block AFTER buffer update.
// (如果输出是 'reg'，一个更简单的方法是在缓冲区更新后，直接在时钟块中赋值。)
// 你这里用了第二个 always 块来赋值输出，这也是可以的，确保了时序性。

always @(posedge clk or negedge rst_n) begin // 时序逻辑: 更新输出寄存器
    if (!rst_n) begin // 复位
        display_data_seg1 <= 3'd0;
        display_data_seg2 <= 3'd0;
        display_data_seg3 <= 3'd0;
        display_data_seg4 <= 3'd0;
        display_data_seg5 <= 3'd0;
        display_data_seg6 <= 3'd0;
    end else begin // 正常操作
        // Update outputs from the buffer contents
        // (从缓冲区内容更新输出)
        display_data_seg1 <= seg_buffer[0]; // SEG1的显示数据来自 buffer[0]
        display_data_seg2 <= seg_buffer[1]; // SEG2的显示数据来自 buffer[1]
        display_data_seg3 <= seg_buffer[2]; // ...
        display_data_seg4 <= seg_buffer[3];
        display_data_seg5 <= seg_buffer[4];
        display_data_seg6 <= seg_buffer[5]; // SEG6的显示数据来自 buffer[5]
    end
end

endmodule
```

### mode_sequencer.v

```verilog
// File: mode_sequencer.v (Corrected and Refined)
// (模式序列器模块 - 已修正和优化)
module mode_sequencer (
    input clk,
    input rst_n, // 低电平有效复位

    // Input from the main piano logic (debounced, highest priority key ID)
    // (来自主钢琴逻辑的输入 - 已消抖的、优先级最高的按键ID)
    input [3:0] current_live_key_id,         // 4-bit ID: 1-12 for notes, 0 for none
                                             // (4位ID: 1-12代表音符, 0代表无)
    input       current_live_key_pressed,    // Is a live key currently pressed?
                                             // (当前是否有实时按键按下?)

    // Output to indicate practice mode activation
    // (输出以指示练习模式激活)
    output reg  practice_mode_active_pulse   // A single clock pulse when sequence is matched
                                             // (当序列匹配时产生的单时钟周期脉冲)
);

// --- Parameters for the sequence ---
// --- 序列参数 ---
localparam SEQ_LENGTH = 7; // Length of your sequence "2317616" (你的序列 "2317616" 的长度)

// CORRECTED: Single line assignment for TARGET_SEQUENCE, ensure your Verilog version in Quartus is 2001 or SystemVerilog
// (已修正: TARGET_SEQUENCE的单行赋值，确保您Quartus中的Verilog版本是2001或SystemVerilog)
// 使用函数来获取目标序列的值，更灵活
function [3:0] get_target_sequence_val (input integer index); // 输入索引，返回对应序列值
    case (index) // 序列 "2317616" 对应按键ID: D(2), E(3), C(1), B(7), A(6), C(1), A(6)
        0: get_target_sequence_val = 4'd2; // 第0个是 D (ID 2)
        1: get_target_sequence_val = 4'd3; // 第1个是 E (ID 3)
        2: get_target_sequence_val = 4'd1; // 第2个是 C (ID 1)
        3: get_target_sequence_val = 4'd7; // 第3个是 B (ID 7)
        4: get_target_sequence_val = 4'd6; // 第4个是 A (ID 6)
        5: get_target_sequence_val = 4'd1; // 第5个是 C (ID 1)
        6: get_target_sequence_val = 4'd6; // 第6个是 A (ID 6)
        default: get_target_sequence_val = 4'dx; // Or some other default (或其他默认值，表示无效索引)
    endcase
endfunction

// Timeout for sequence input (e.g., 2 seconds between key presses)
// (序列输入的超时时间，例如两次按键之间2秒)
localparam TIMEOUT_MS = 2000; // 2 seconds (2秒)
localparam CLK_FREQ_HZ = 50_000_000; // 时钟频率
localparam TIMEOUT_CYCLES = (TIMEOUT_MS * (CLK_FREQ_HZ / 1000)); // 超时对应的时钟周期数

// --- Internal state and registers ---
// --- 内部状态和寄存器 ---
// Corrected width for current_match_index to safely hold 0 to SEQ_LENGTH states (e.g., 0-6 for match, 7 for 'done' or use 0 to SEQ_LENGTH-1 as index)
// (修正了 current_match_index 的位宽，以安全地容纳0到SEQ_LENGTH个状态 (例如，0-6用于匹配，7用于'完成'，或使用0到SEQ_LENGTH-1作为索引))
// It needs to hold values from 0 up to SEQ_LENGTH-1 as an index.
// (它需要容纳从0到SEQ_LENGTH-1的值作为索引。)
// $clog2(SEQ_LENGTH) gives bits for 0 to SEQ_LENGTH-1. If SEQ_LENGTH=7, needs 3 bits (0-6).
// ($clog2(SEQ_LENGTH) 给出0到SEQ_LENGTH-1所需的位数。如果SEQ_LENGTH=7，需要3位 (0-6)。)
reg [$clog2(SEQ_LENGTH > 1 ? SEQ_LENGTH : 2)-1:0] current_match_index; // 当前匹配的序列索引 (例如，SEQ_LENGTH=7时为 [2:0])

reg [3:0] last_pressed_key_id_prev_cycle; // Stores key_id from previous cycle to detect new presses
                                          // (存储上一周期的按键ID，用于检测新的按键按下)
reg [$clog2(TIMEOUT_CYCLES > 1 ? TIMEOUT_CYCLES : 2)-1:0] timeout_counter_reg; // 超时计数器
reg sequence_input_active_flag; // Flag to indicate if we are in the middle of inputting a sequence
                                // (标志：指示我们是否正在输入序列的过程中)

initial begin // 初始化
    practice_mode_active_pulse = 1'b0;
    current_match_index = 0; // 初始匹配索引为0
    last_pressed_key_id_prev_cycle = 4'd0; // No key initially (初始无按键)
    timeout_counter_reg = 0;
    sequence_input_active_flag = 1'b0; // 初始不在序列输入中
end

always @(posedge clk or negedge rst_n) begin // 时序逻辑
    if (!rst_n) begin // 复位
        practice_mode_active_pulse <= 1'b0;
        current_match_index <= 0;
        last_pressed_key_id_prev_cycle <= 4'd0;
        timeout_counter_reg <= 0;
        sequence_input_active_flag <= 1'b0;
    end else begin
        // Default: pulse is low unless explicitly set high for one cycle
        // (默认：脉冲为低，除非显式地将其置高一个周期)
        practice_mode_active_pulse <= 1'b0;

        // Timeout logic
        // (超时逻辑)
        if (sequence_input_active_flag) begin // 如果正在输入序列
            if (timeout_counter_reg >= TIMEOUT_CYCLES - 1) begin // 如果超时计数器达到设定值
                // Timeout occurred, reset sequence matching
                // (发生超时，重置序列匹配)
                current_match_index <= 0;
                sequence_input_active_flag <= 1'b0; // 退出序列输入状态
                timeout_counter_reg <= 0;
            end else begin
                timeout_counter_reg <= timeout_counter_reg + 1'b1; // 超时计数器加1
            end
        end else begin // 如果不在序列输入中
            timeout_counter_reg <= 0; // Reset timer (重置定时器)
        end

        // Key press detection and sequence matching logic
        // (按键检测和序列匹配逻辑)
        // A new key press is when current_live_key_pressed is true,
        // and current_live_key_id is different from last_pressed_key_id_prev_cycle,
        // and current_live_key_id is not 0 (rest).
        // (新的按键按下事件：当前有键按下，且当前按键ID与上一周期的不同，并且当前按键ID不是0(休止符))
        if (current_live_key_pressed && current_live_key_id != 4'd0 && current_live_key_id != last_pressed_key_id_prev_cycle) begin
            // This is a new, valid musical key press event
            // (这是一个新的、有效的音乐按键按下事件)
            timeout_counter_reg <= 0;                 // Reset timeout timer on new key press (新按键按下时重置超时计数器)
            sequence_input_active_flag <= 1'b1;       // We are now actively inputting/checking a sequence (我们现在正在积极输入/检查序列)

            if (current_live_key_id == get_target_sequence_val(current_match_index)) begin // 如果当前按键与序列中对应位置的按键匹配
                // Correct key for the current step in the sequence
                // (序列中当前步骤的正确按键)
                if (current_match_index == SEQ_LENGTH - 1) begin // 如果是序列的最后一个按键
                    // Last key of the sequence matched!
                    // (序列的最后一个键匹配成功!)
                    practice_mode_active_pulse <= 1'b1; // Fire the pulse! (发出激活脉冲!)
                    current_match_index <= 0;           // Reset for next time (为下一次重置)
                    sequence_input_active_flag <= 1'b0; // Sequence complete, no longer active (序列完成，不再激活)
                end else begin // 不是最后一个按键，但目前为止正确
                    // Not the last key, but correct so far. Advance.
                    // (不是最后一个键，但到目前为止是正确的。前进。)
                    current_match_index <= current_match_index + 1'b1; // 匹配索引加1，准备匹配下一个
                end
            end else begin // Incorrect key pressed for the sequence (按下了序列中不正确的键)
                // If the incorrect key is the start of a new target sequence, restart matching from step 1
                // (如果错误的键是新目标序列的开始，则从步骤1重新开始匹配)
                if (current_live_key_id == get_target_sequence_val(0)) begin // 如果按下的键是序列的第一个键
                    current_match_index <= 1; // Matched the first element of the sequence (匹配了序列的第一个元素，所以索引变为1)
                end else begin // 错误的键，并且它也不是新序列的开始
                    current_match_index <= 0; // Wrong key, and it's not the start of a new sequence, reset. (错误的键，重置匹配索引)
                    sequence_input_active_flag <= 1'b0; // 退出序列输入状态
                end
            end
        end

        // Update last_pressed_key_id_prev_cycle for the next clock cycle
        // (为下一个时钟周期更新 last_pressed_key_id_prev_cycle)
        // If a key is pressed, store its ID. If no key is pressed, store 0.
        // (如果按键被按下，存储其ID。如果未按下键，则存储0。)
        if (current_live_key_pressed && current_live_key_id != 4'd0) begin // 如果有有效音乐按键按下
            last_pressed_key_id_prev_cycle <= current_live_key_id; // 记录当前按键ID
        end else if (!current_live_key_pressed) begin // Key has been released (按键已释放)
            last_pressed_key_id_prev_cycle <= 4'd0; // 记录为无按键
        end
        // If key is held (current_live_key_id == last_pressed_key_id_prev_cycle),
        // last_pressed_key_id_prev_cycle remains, and the main `if` condition above won't trigger for "new press".
        // (如果按键被按住 (current_live_key_id == last_pressed_key_id_prev_cycle)，
        // last_pressed_key_id_prev_cycle 保持不变，上面的主要 `if` 条件不会因“新按键”而触发。)
    end
end

endmodule
```

### practice_player.v

```verilog
// File: practice_player.v (Reconstructed Complete Version)
// (练习播放器模块 - 重构的完整版本)
module practice_player #(
    parameter NUM_DISPLAY_SEGMENTS = 6 // 用于练习提示的数码管段数
) (
    input clk,
    input rst_n, // 低电平有效复位

    input practice_mode_active,         // 练习模式是否激活
    input [3:0] current_live_key_id,    // 当前实时按下的按键ID
    input current_live_key_pressed,   // 当前是否有实时按键按下

    // 输出到数码管的数据 (每个seg对应一个音符的显示ID, 0表示空白, 1-7表示C-B)
    output reg [2:0] display_out_seg0, // 数码管最右边(或按你的布局是第0个)的显示数据
    output reg [2:0] display_out_seg1,
    output reg [2:0] display_out_seg2,
    output reg [2:0] display_out_seg3,
    output reg [2:0] display_out_seg4,
    output reg [2:0] display_out_seg5, // 数码管最左边(或按你的布局是第5个)的显示数据

    output reg correct_note_played_event,  // 正确音符按下事件 (单周期脉冲)
    output reg wrong_note_played_event,    // 错误音符按下事件 (单周期脉冲)
    output reg practice_song_finished_event // 练习曲目完成事件 (单周期脉冲)
);

// --- Parameters ---
// --- 参数 ---
localparam PRACTICE_SONG_LENGTH = 14; // 练习曲目的音符数量

// --- Functions ---
// --- 函数 ---
// 获取练习曲目中指定索引的音符ID
function [3:0] get_practice_song_note (input integer index);
    if (index >= PRACTICE_SONG_LENGTH || index < 0) begin // 索引越界检查
        get_practice_song_note = 4'd0; // 返回休止符或无效音符
    end else begin
        case (index) // 练习曲谱 "两只老虎" (C C G G A A G - F F E E D D C)
                     // 对应音符ID: 1 1 5 5 6 6 5 - 4 4 3 3 2 2 1
            0:  get_practice_song_note = 4'd1; 1:  get_practice_song_note = 4'd1; // C C
            2:  get_practice_song_note = 4'd5; 3:  get_practice_song_note = 4'd5; // G G
            4:  get_practice_song_note = 4'd6; 5:  get_practice_song_note = 4'd6; // A A
            6:  get_practice_song_note = 4'd5; 7:  get_practice_song_note = 4'd4; // G F
            8:  get_practice_song_note = 4'd4; 9:  get_practice_song_note = 4'd3; // F E
            10: get_practice_song_note = 4'd3; 11: get_practice_song_note = 4'd2; // E D
            12: get_practice_song_note = 4'd2; 13: get_practice_song_note = 4'd1; // D C
            default: get_practice_song_note = 4'd0; // 默认返回休止符
        endcase
    end
endfunction

// 将音乐按键ID (1-12) 转换为用于数码管显示的ID (1-7代表C-B，其他转为0或特定显示)
// 这里只处理了基础音1-7的映射，半音没有直接显示为数字。
function [2:0] musical_to_display_id (input [3:0] musical_id);
    case (musical_id)
        4'd1:  musical_to_display_id = 3'd1; // C -> 1
        4'd2:  musical_to_display_id = 3'd2; // D -> 2
        4'd3:  musical_to_display_id = 3'd3; // E -> 3
        4'd4:  musical_to_display_id = 3'd4; // F -> 4
        4'd5:  musical_to_display_id = 3'd5; // G -> 5
        4'd6:  musical_to_display_id = 3'd6; // A -> 6
        4'd7:  musical_to_display_id = 3'd7; // B -> 7
        // 对于半音 (IDs 8-12)，这里没有映射，默认会显示空白(0)。
        // 如果想显示半音的根音，可以添加：
        // 4'd8:  musical_to_display_id = 3'd1; // C# -> 显示 C(1)
        // 4'd9:  musical_to_display_id = 3'd3; // Eb -> 显示 E(3) (或者D(2)如果你认为是D#)
        // ...
        default: musical_to_display_id = 3'd0; // 其他 (如休止符, 半音) 显示空白
    endcase
endfunction

// --- Internal Registers ---
// --- 内部寄存器 ---
reg [$clog2(PRACTICE_SONG_LENGTH + 1)-1:0] current_note_index_in_song; // 当前在练习曲目中的音符索引
reg current_live_key_pressed_prev; // 上一周期实时按键是否按下 (用于检测新按键)
reg [$clog2(PRACTICE_SONG_LENGTH + 1)-1:0] next_note_idx_calculated; // Moved declaration to module level (移至模块级别声明)

// --- Wires ---
// --- 连线 ---
wire new_key_press_event; // 新按键按下事件标志

// --- Tasks ---
// --- 任务 ---
// 更新数码管显示缓冲区，从指定的歌曲索引起始，显示NUM_DISPLAY_SEGMENTS个音符
task update_display_buffer (input [$clog2(PRACTICE_SONG_LENGTH + 1)-1:0] base_song_idx_for_display);
    integer i; // 循环变量
    integer song_idx_to_show; // 要显示的歌曲音符的实际索引
    reg [2:0] temp_display_buffer [NUM_DISPLAY_SEGMENTS-1:0]; // 临时显示缓冲区
    begin // Task body begin (任务体开始)
        for (i = 0; i < NUM_DISPLAY_SEGMENTS; i = i + 1) begin // For loop begin (for循环开始)
            song_idx_to_show = base_song_idx_for_display + i; // 计算要显示的音符在完整曲谱中的索引
            if (song_idx_to_show < PRACTICE_SONG_LENGTH) begin // If begin (如果索引在曲谱范围内)
                temp_display_buffer[i] = musical_to_display_id(get_practice_song_note(song_idx_to_show)); // 获取并转换音符ID
            end else begin // Else for if begin (如果索引超出曲谱范围)
                temp_display_buffer[i] = 3'd0; // 显示空白
            end // If else end (if-else结束)
        end // For loop end (for循环结束)
        // 将临时缓冲区的值赋给输出端口
        display_out_seg0 <= temp_display_buffer[0]; display_out_seg1 <= temp_display_buffer[1];
        display_out_seg2 <= temp_display_buffer[2]; display_out_seg3 <= temp_display_buffer[3];
        display_out_seg4 <= temp_display_buffer[4]; display_out_seg5 <= temp_display_buffer[5];
    end // Task body end (任务体结束)
endtask

// --- Initial block ---
// --- 初始化块 ---
initial begin
    current_note_index_in_song = 0; // 初始指向歌曲第一个音符
    correct_note_played_event = 1'b0; wrong_note_played_event = 1'b0;
    practice_song_finished_event = 1'b0;
    display_out_seg0 = 3'd0; display_out_seg1 = 3'd0; display_out_seg2 = 3'd0;
    display_out_seg3 = 3'd0; display_out_seg4 = 3'd0; display_out_seg5 = 3'd0;
    current_live_key_pressed_prev = 1'b0; // 初始认为上一周期无按键按下
    next_note_idx_calculated = 0; // Initialize module level reg (初始化模块级寄存器)
end

// --- Combinational logic for new_key_press_event ---
// --- new_key_press_event 的组合逻辑 ---
// 当current_live_key_pressed为高且上周期为低时，产生新按键事件
assign new_key_press_event = current_live_key_pressed && !current_live_key_pressed_prev;

// --- Sequential logic for current_live_key_pressed_prev ---
// --- current_live_key_pressed_prev 的时序逻辑 ---
// 每个时钟周期更新上一周期的按键状态
always @(posedge clk or negedge rst_n) begin
    if (!rst_n) begin
        current_live_key_pressed_prev <= 1'b0;
    end else begin
        current_live_key_pressed_prev <= current_live_key_pressed;
    end
end

// --- Main sequential logic block ---
// --- 主要时序逻辑块 ---
always @(posedge clk or negedge rst_n) begin
    // 'next_note_idx_calculated' is a module-level reg, assigned before use in this block.
    // ('next_note_idx_calculated' 是一个模块级寄存器，在此块中使用之前已赋值。)
    if (!rst_n) begin // 复位逻辑
        current_note_index_in_song <= 0; // 重置歌曲索引
        correct_note_played_event <= 1'b0; wrong_note_played_event <= 1'b0;
        practice_song_finished_event <= 1'b0;
        update_display_buffer(0); // 复位时更新显示，从歌曲开头显示
    end else begin // 正常操作
        // 默认将事件脉冲清零，只在事件发生时置高一周期
        correct_note_played_event <= 1'b0;
        wrong_note_played_event <= 1'b0;
        // practice_song_finished_event 不在此处清零，它由非练习模式时清除

        if (practice_mode_active) begin // 如果练习模式激活
            if (new_key_press_event && current_live_key_id != 4'd0) begin // 如果有新的有效音乐按键按下
                if (current_note_index_in_song < PRACTICE_SONG_LENGTH) begin // 如果当前练习还未结束
                    if (current_live_key_id == get_practice_song_note(current_note_index_in_song)) begin // 如果按下的键与当前期望的练习音符匹配
                        correct_note_played_event <= 1'b1; // 产生正确音符事件脉冲

                        next_note_idx_calculated = current_note_index_in_song + 1; // Assignment (计算下一个音符的索引)

                        if (current_note_index_in_song == PRACTICE_SONG_LENGTH - 1) begin // 如果当前是最后一个音符
                            practice_song_finished_event <= 1'b1; // 产生练习完成事件脉冲
                        end // end if (current_note_index_in_song == ...)

                        current_note_index_in_song <= next_note_idx_calculated; // Usage (更新当前音符索引到下一个)
                        update_display_buffer(next_note_idx_calculated);      // Usage (更新数码管显示，从下一个音符开始)
                    end else begin // else for if (current_live_key_id == ...) (按下的键不匹配)
                        wrong_note_played_event <= 1'b1; // 产生错误音符事件脉冲
                        // 错误时不推进练习曲谱，等待用户按对
                    end // end if (current_live_key_id == ...) else
                end // end if (current_note_index_in_song < ...)
                  // 如果 current_note_index_in_song >= PRACTICE_SONG_LENGTH (练习已完成)，则不再响应按键
            end // end if (new_key_press_event && ...)
        end else begin // else for if (practice_mode_active) (如果练习模式未激活)
            if (current_note_index_in_song != 0) begin // 如果之前在练习，现在退出了，则重置练习索引
                 current_note_index_in_song <= 0;
                 update_display_buffer(0); // 并更新显示
            end // end if (current_note_index_in_song != 0)
            if (practice_song_finished_event) begin // 如果练习完成标志仍然是高，清零它
                practice_song_finished_event <= 1'b0;
            end // end if (practice_song_finished_event)
        end // end if (practice_mode_active) else
    end // end if (!rst_n) else
end // end always

endmodule
```
