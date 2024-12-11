# ESP32使用GPIO
## 案例代码学习
- 案例路径：`Espressif\frameworks\esp-idf-v5.1.5\examples\peripherals\gpio\generic_gpio`
- 项目结构：
    ```
    │  CMakeLists.txt  
    │  pytest_generic_gpio_example.py
    │  README.md
    │
    └─main
            CMakeLists.txt
            gpio_example_main.c
            Kconfig.projbuild
    ```
    - `CMakeLists.txt `
        ```cmake
        # The following lines of boilerplate have to be in your project's CMakeLists
        # in this exact order for cmake to work correctly
        cmake_minimum_required(VERSION 3.16)  #这一行指定了CMake所需的最低版本为3.16。如果使用的CMake版本低于这个版本，CMake会报错并停止构建。

        #这一行包含了指定路径下的CMake配置文件。$ENV{IDF_PATH}是一个环境变量，指向ESP-IDF框架的安装路径。
        include($ENV{IDF_PATH}/tools/cmake/project.cmake)

        #这一行定义了当前项目的名称为generic_gpio
        project(generic_gpio)
        ```
    - `pytest_generic_gpio_example.py`
        ```python
        #主要用于验证ESP-IDF GPIO示例程序的行为

        # SPDX-FileCopyrightText: 2022 Espressif Systems (Shanghai) CO LTD
        # SPDX-License-Identifier: CC0-1.0

        from typing import Callable

        import pytest
        from pytest_embedded import Dut


        @pytest.mark.supported_targets  #标记这是一项针对已支持目标的测试
        @pytest.mark.generic  #标记这是一项通用测试
        def test_generic_gpio_example(  #定义了一个测试函数
            dut: Dut, log_minimum_free_heap_size: Callable[..., None]  #两个参数：dut和log_minimum_free_heap_size
        ) -> None:
            log_minimum_free_heap_size()  #打印最小空闲堆栈大小
            dut.expect(r'cnt: \d+')
        ```
    - `README.md`
        - 介绍了案例的功能和使用方法。
    - `main`
        - `CMakeLists.txt`
            ```cmake
            # 将名为gpio_example_main.c的源文件注册为一个ESP-IDF组件

            # idf_component_register: 这是一个宏，用于在ESP-IDF框架中注册一个组件。
            # SRCS是一个参数，用来指定组件源文件的列表
            # INCLUDE_DIRS指定了包含目录，即编译时需要查找头文件的目录

            idf_component_register(SRCS "gpio_example_main.c"
                    INCLUDE_DIRS ".")
            ```
        - `gpio_example_main.c`
            ```c
            /**
            * 简介：
            * 该测试代码展示了如何配置 GPIO 以及如何使用 GPIO 中断。
            *
            * GPIO 状态：
            * GPIO18: 输出（ESP32C2/ESP32H2 使用 GPIO8 作为第二个输出引脚）
            * GPIO19: 输出（ESP32C2/ESP32H2 使用 GPIO9 作为第二个输出引脚）
            * GPIO4:  输入，上拉，触发上升沿和下降沿中断
            * GPIO5:  输入，上拉，仅触发上升沿中断。
            *
            * 注意：这些是示例中默认使用的 GPIO 引脚。您可以
            * 在 menuconfig 中更改 IO 引脚。
            *
            * 测试：
            * 将 GPIO18(8) 连接到 GPIO4
            * 将 GPIO19(9) 连接到 GPIO5
            * 在 GPIO18(8)/19(9) 上产生脉冲，以触发 GPIO4/5 的中断
            *
            */
            #include <stdio.h>
            #include <string.h>
            #include <stdlib.h>
            #include <inttypes.h>
            #include "freertos/FreeRTOS.h"
            #include "freertos/task.h"
            #include "freertos/queue.h"
            #include "driver/gpio.h"

            #define GPIO_OUTPUT_IO_0    CONFIG_GPIO_OUTPUT_0
            #define GPIO_OUTPUT_IO_1    CONFIG_GPIO_OUTPUT_1
            #define GPIO_OUTPUT_PIN_SEL  ((1ULL<<GPIO_OUTPUT_IO_0) | (1ULL<<GPIO_OUTPUT_IO_1))
            /*
            * 假设 GPIO_OUTPUT_IO_0=18, GPIO_OUTPUT_IO_1=19
            * 在二进制表示中，
            * 1ULL<<GPIO_OUTPUT_IO_0 等于 0000000000000000000001000000000000000000
            * 1ULL<<GPIO_OUTPUT_IO_1 等于 0000000000000000000010000000000000000000
            * GPIO_OUTPUT_PIN_SEL         0000000000000000000011000000000000000000
            * */  
            #define GPIO_INPUT_IO_0     CONFIG_GPIO_INPUT_0
            #define GPIO_INPUT_IO_1     CONFIG_GPIO_INPUT_1
            #define GPIO_INPUT_PIN_SEL  ((1ULL<<GPIO_INPUT_IO_0) | (1ULL<<GPIO_INPUT_IO_1))   //同理
            #define ESP_INTR_FLAG_DEFAULT 0

            static QueueHandle_t gpio_evt_queue = NULL;  

            static void IRAM_ATTR gpio_isr_handler(void* arg)
            {
                uint32_t gpio_num = (uint32_t) arg;
                xQueueSendFromISR(gpio_evt_queue, &gpio_num, NULL);
            }

            static void gpio_task_example(void* arg)
            {
                uint32_t io_num;
                for(;;) {
                    if(xQueueReceive(gpio_evt_queue, &io_num, portMAX_DELAY)) {
                        printf("GPIO[%"PRIu32"] intr, val: %d\n", io_num, gpio_get_level(io_num));
                    }
                }
            }

            void app_main(void)
            {
                gpio_config_t io_conf = {};      // 将配置结构体初始化为零
                io_conf.intr_type = GPIO_INTR_DISABLE;      // 禁用中断
                io_conf.mode = GPIO_MODE_OUTPUT;        // 设置为输出模式
                io_conf.pin_bit_mask = GPIO_OUTPUT_PIN_SEL;// 设置要配置的引脚的位掩码，例如 GPIO18/19
                io_conf.pull_down_en = 0;// 禁用下拉模式
                io_conf.pull_up_en = 0;// 禁用上拉模式
                gpio_config(&io_conf);// 根据给定的设置配置 GPIO

                io_conf.intr_type = GPIO_INTR_POSEDGE;// 设置上升沿中断
                io_conf.pin_bit_mask = GPIO_INPUT_PIN_SEL;// 设置位掩码，引脚使用 GPIO4/5
                io_conf.mode = GPIO_MODE_INPUT;// 设置为输入模式
                io_conf.pull_up_en = 1;// 启用上拉模式
                gpio_config(&io_conf);

                gpio_set_intr_type(GPIO_INPUT_IO_0, GPIO_INTR_ANYEDGE);// 更改某个引脚的GPIO中断类型

                gpio_evt_queue = xQueueCreate(10, sizeof(uint32_t));// 创建一个队列以处理来自中断服务例程的GPIO事件
                xTaskCreate(gpio_task_example, "gpio_task_example", 2048, NULL, 10, NULL);// 启动GPIO任务

                gpio_install_isr_service(ESP_INTR_FLAG_DEFAULT);// 安装GPIO中断服务
                gpio_isr_handler_add(GPIO_INPUT_IO_0, gpio_isr_handler, (void*) GPIO_INPUT_IO_0);// 为特定GPIO引脚添加中断处理程序
                gpio_isr_handler_add(GPIO_INPUT_IO_1, gpio_isr_handler, (void*) GPIO_INPUT_IO_1);// 为特定GPIO引脚添加中断处理程序

                gpio_isr_handler_remove(GPIO_INPUT_IO_0);// 移除特定GPIO引脚的中断处理程序
                gpio_isr_handler_add(GPIO_INPUT_IO_0, gpio_isr_handler, (void*) GPIO_INPUT_IO_0);// 重新为特定GPIO引脚添加中断处理程序

                printf("Minimum free heap size: %"PRIu32" bytes\n", esp_get_minimum_free_heap_size());

                int cnt = 0;
                while(1) {
                    printf("cnt: %d\n", cnt++);
                    vTaskDelay(1000 / portTICK_PERIOD_MS);
                    gpio_set_level(GPIO_OUTPUT_IO_0, cnt % 2);
                    gpio_set_level(GPIO_OUTPUT_IO_1, cnt % 2); 
                }
            }
            ```
        - `Kconfig.projbuild`  
            这个主要用于设置项目的示例配置选项，在IDF可以直接配置烧录此示例项目，我们正常写项目一般不需要这个文件。
            ```
            menu "Example Configuration"

            orsource "$IDF_PATH/examples/common_components/env_caps/$IDF_TARGET/Kconfig.env_caps"

            config GPIO_OUTPUT_0
                int "GPIO output pin 0"
                range ENV_GPIO_RANGE_MIN ENV_GPIO_OUT_RANGE_MAX
                default 8 if IDF_TARGET_ESP32C2 || IDF_TARGET_ESP32H2
                default 18
                help
                    GPIO pin number to be used as GPIO_OUTPUT_IO_0.

            config GPIO_OUTPUT_1
                int "GPIO output pin 1"
                range ENV_GPIO_RANGE_MIN ENV_GPIO_OUT_RANGE_MAX
                default 9 if IDF_TARGET_ESP32C2 || IDF_TARGET_ESP32H2
                default 19
                help
                    GPIO pin number to be used as GPIO_OUTPUT_IO_1.

            config GPIO_INPUT_0
                int "GPIO input pin 0"
                range ENV_GPIO_RANGE_MIN ENV_GPIO_IN_RANGE_MAX
                default 4
                help
                    GPIO pin number to be used as GPIO_INPUT_IO_0.

            config GPIO_INPUT_1
                int "GPIO input pin 1"
                range ENV_GPIO_RANGE_MIN ENV_GPIO_IN_RANGE_MAX
                default 5
                help
                    GPIO pin number to be used as GPIO_INPUT_IO_1.

            endmenu
            ```
