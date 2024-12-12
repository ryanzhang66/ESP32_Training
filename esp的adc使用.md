# ESP32的ADC使用

## IDF案例代码学习

## ArduinoIDE案例代码学习
- `AnalogRead.ino`  
    单个ADC读取示例代码
    ```ino
    void setup() {
    Serial.begin(115200);    // 以115200比特每秒初始化串口通信：
    analogReadResolution(12);// 将分辨率设置为12位（0-4095）
    }

    void loop() {
    // 读取引脚2的模拟/毫伏值：
    int analogValue = analogRead(2);
    int analogVolts = analogReadMilliVolts(2);

    // 打印读取的值：
    Serial.printf("ADC模拟值 = %d\n", analogValue);
    Serial.printf("ADC毫伏值 = %d\n", analogVolts);

    delay(100);  // 在读取之间延迟以清晰显示串口读取值
    }
    ```
- `AnalogReadContinuous.ino`  
    定义每个引脚进行多少次转换，读取数据将是所有转换的平均值
    ```ino
    // 定义每个引脚进行多少次转换，读取数据将是所有转换的平均值
    #define CONVERSIONS_PER_PIN 5

    // 声明将用于ADC连续模式的ADC引脚数组 - 仅支持ADC1引脚
    // 选择的引脚数量可以从1到所有ADC1引脚。
    #ifdef CONFIG_IDF_TARGET_ESP32
    uint8_t adc_pins[] = {36, 39, 34, 35};  // ESP32的一些ADC1引脚
    #else
    uint8_t adc_pins[] = {1, 2, 3, 4};  // ESP32S2/S3 + ESP32C3/C6 + ESP32H2的ADC1公共引脚
    #endif

    // 计算数组中声明了多少个引脚 - 作为ADC连续模式设置函数的输入需要
    uint8_t adc_pins_count = sizeof(adc_pins) / sizeof(uint8_t);

    // 在中断服务例程中转换完成时将设置的标志
    volatile bool adc_coversion_done = false;

    // ADC连续读取的结果结构
    adc_continuous_data_t *result = NULL;

    // 当ADC转换完成时触发的ISR函数
    void ARDUINO_ISR_ATTR adcComplete() {
    adc_coversion_done = true;
    }

    void setup() {
    // 以115200比特每秒初始化串口通信：
    Serial.begin(115200);

    // 可选：为ESP32设置分辨率为9-12位（默认是12位）
    analogContinuousSetWidth(12);

    // 可选：设置不同的衰减（默认是ADC_11db）
    analogContinuousSetAtten(ADC_11db);

    // 设置ADC连续模式，输入包括：
    // 引脚数组、引脚数量、每个引脚每个周期进行多少次转换、采样频率、回调函数
    analogContinuous(adc_pins, adc_pins_count, CONVERSIONS_PER_PIN, 20000, &adcComplete);

    // 启动ADC连续转换
    analogContinuousStart();
    }

    void loop() {
    // 检查转换是否完成，并尝试读取数据
    if (adc_coversion_done == true) {
        // 将ISR标志重置为false
        adc_coversion_done = false;
        // 从ADC读取数据
        if (analogContinuousRead(&result, 0)) {

        // 可选：停止ADC连续转换，以便有更多时间处理（打印）数据
        analogContinuousStop();

        for (int i = 0; i < adc_pins_count; i++) {
            Serial.printf("\nADC引脚 %d 数据:", result[i].pin);
            Serial.printf("\n   平均原始值 = %d", result[i].avg_read_raw);
            Serial.printf("\n   平均毫伏值 = %d", result[i].avg_read_mvolts);
        }

        // 延迟以更好地显示ADC数据
        delay(1000);

        // 可选：如果ADC被停止，则启动ADC转换并等待回调函数将adc_coversion_done标志设置为true
        analogContinuousStart();
        } else {
        Serial.println("读取数据时发生错误。将核心调试级别设置为错误或更低以获取更多信息。");
        }
    }
    }

    ```