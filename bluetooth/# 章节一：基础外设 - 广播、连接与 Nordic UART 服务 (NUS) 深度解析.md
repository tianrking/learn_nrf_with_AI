# 章节一：基础外设 - 广播、连接与 Nordic UART 服务 (NUS) 深度解析

**主示例 (Main Example)**: `peripheral_uart` (位于 `~/ncs/v3.0.1/nrf/samples/bluetooth/peripheral_uart/`)
**辅助理解 (Auxiliary Example for Understanding)**: `shell_bt_nus` (可用于通过 Shell 命令行与 NUS 服务交互，帮助验证和理解其行为)

## 1. 核心蓝牙知识点 (Core BLE Concepts) - 深度剖析

### 1.1. 通用访问配置文件 (GAP - Generic Access Profile)

GAP 层定义了设备如何发现彼此、建立连接以及管理这些连接。它是设备间互操作的基础。

* **角色 (Role)**: 本示例中 nRF52832 扮演 **外设 (Peripheral)**。
    * **定义**: 外设通常是资源受限的设备，它通过广播 (Advertising) 来宣告自己的存在和能力，并接受来自中心设备 (Central) 的连接请求。
    * **应用场景**: 物联网 (IoT) 中的传感器节点 (Sensor Nodes)、可穿戴设备 (Wearables)、健康监测器 (Health Monitors)、智能家居设备 (Smart Home Appliances)、遥控器 (Remote Controls)、信标 (Beacons) 等。

* **广播 (Advertising)**: 外设周期性地发送**广播数据包 (Advertising Packets / Advertising PDUs - Protocol Data Units)**，使其能被扫描设备发现。
    * **广播数据 (Advertising Data - AD)**: 包含在主要广播数据包中，长度通常限制在 31 字节 (对于传统广播 Legacy Advertising)。
        * **Flags (AD Type: `0x01`)**: 表明设备的能力和模式。
            * `BT_LE_AD_GENERAL` (通用可发现模式 - General Discoverable Mode): 表示设备希望被任何扫描设备发现。
            * `BT_LE_AD_LIMITED_DISCOVERABLE_MODE` (有限可发现模式 - Limited Discoverable Mode): 设备仅在有限时间内可被发现，通常用于特定操作后 (如按键触发)，以节省功耗或增强隐私。
            * `BT_LE_AD_NO_BREDR` (不支持经典蓝牙 - BR/EDR Not Supported): 表明这是一个纯 BLE 设备。
        * **设备名称 (Device Name - AD Type: `0x09` for Complete Local Name, `0x08` for Shortened Local Name)**: 人类可读的设备标识，如本例中的 "Nordic\_UART\_Service"。
            * **配置**: 在 `prj.conf` 中通过 `CONFIG_BT_DEVICE_NAME` 设置。
            * **应用**: 如果完整名称过长，可使用缩短名称以在有限的广播包空间内放入更多其他信息。对于空间极度敏感的应用，甚至可以不在广播包中包含名称，而是在扫描响应或 GATT 设备信息服务 (DIS) 中提供。
        * **配置与应用 - 广播间隔 (Advertising Interval)**:
            * **定义**: 两个连续广播事件 (Advertising Event) 开始之间的时间。一个广播事件可能包含在多个广播信道 (37, 38, 39) 上发送数据包。
            * **配置**: 在 `prj.conf` 中通过 Kconfig 选项如 `CONFIG_BT_GAP_ADV_FAST_INT_MIN_1` (例如默认 30ms, 即 `0x0030` = 48 * 0.625ms), `CONFIG_BT_GAP_ADV_FAST_INT_MAX_1` (例如默认 60ms, 即 `0x0060` = 96 * 0.625ms) 和 `CONFIG_BT_GAP_ADV_FAST_TIMEOUT` (快速广播持续时间) 等进行配置。也可以在代码中通过填充 `struct bt_le_adv_param` 结构体并传递给 `bt_le_adv_start` 来进行更精细的控制。
            * **应用**:
                * **快速发现 (Shorter Interval, e.g., 20ms-100ms)**: 设备能更快被中心设备发现。适用于需要快速建立连接的应用，如无线鼠标首次配对、或需要立即响应的交互式设备。缺点是功耗较高。
                * **节能 (Longer Interval, e.g., 1s-10.24s)**: 显著降低平均功耗，适用于电池供电且不需要频繁或立即连接的设备，如周期性上报数据的环境传感器、或只需要偶尔被发现的设备。缺点是发现时间变长。
                * **动态调整策略**: 许多设备采用初始快速广播一段时间，若未连接则切换到慢速广播的策略 (`CONFIG_BT_GAP_AUTO_UPDATE_CONN_PARAMS` 相关的 Kconfig 选项可以配置这种行为)，以平衡响应速度和功耗。
    * **扫描响应数据 (Scan Response Data - SRD)**: 当中心设备执行**主动扫描 (Active Scan)** (即发送**扫描请求 (Scan Request PDU)**) 时，外设可以回复此数据包，长度也通常限制在 31 字节。
        * 通常包含在广播数据包中因空间限制而未放入的额外信息，如本例中的 **NUS 服务 UUID (Service UUID)**。
        * **配置与应用**: 可以在扫描响应中放入**制造商特定数据 (Manufacturer Specific Data - AD Type: `0xFF`)**，这对于实现自定义的广播协议或信标 (Beacon) 非常有用 (如 Apple iBeacon, Google Eddystone)。即使不建立连接，中心设备也能通过解析这些数据获取有用的信息。
    * **广播类型 (Advertising Type)**:
        * `BT_LE_ADV_IND` (可连接可扫描的非定向广播 - Connectable and Scannable Undirected Advertising): 最常用的类型，允许任何设备扫描和连接。
        * `BT_LE_ADV_DIRECT_IND` (可连接的定向广播 - Connectable Directed Advertising): 用于快速重新连接到已知的特定中心设备。广播时间非常短。
        * `BT_LE_ADV_SCAN_IND` (可扫描的非定向广播 - Scannable Undirected Advertising): 允许主动扫描获取扫描响应数据，但不可连接。适用于仅广播数据的信标。
        * `BT_LE_ADV_NONCONN_IND` (不可连接不可扫描的非定向广播 - Non-connectable and Non-scannable Undirected Advertising): 纯粹的单向数据广播，如某些类型的传感器数据或最简单的信标。

* **连接 (Connection)**: 当中心设备决定连接某个广播中的外设时，它会发送**连接请求 (Connection Request PDU)**。连接成功后，外设通常会停止广播 (除非配置为多连接或特定广播模式)，双方进入连接状态，可以进行双向数据通信。
    * **连接参数 (Connection Parameters)**: 在连接建立时由中心设备提议，包括连接间隔 (Connection Interval)、从设备延迟 (Peripheral Latency / Slave Latency) 和监控超时 (Supervision Timeout)。这些参数对数据吞吐量、延迟和功耗有显著影响。我们将在后续章节详细讨论。

### 1.2. 属性协议 (ATT - Attribute Protocol) 和 通用属性配置文件 (GATT - Generic Attribute Profile)

GATT 是建立在 ATT 之上的数据交换框架，定义了连接后设备间如何组织和交换数据。它采用**客户端-服务器 (Client-Server)** 模型。

* **GATT 服务器 (GATT Server)**: 本示例中的 nRF52832 (外设) 扮演此角色。它维护一个包含服务 (Services) 和特征 (Characteristics) 的属性表 (Attribute Table)，并响应来自 GATT 客户端的请求。
* **GATT 客户端 (GATT Client)**: 连接到服务器的中心设备 (如手机 App) 扮演此角色。它通过发送 ATT 请求来发现服务、读写特征值等。
* **服务 (Service)**: GATT 服务器上属性的逻辑集合，代表设备的一个特定功能或特性。每个服务有一个唯一的 **通用唯一标识符 (UUID - Universally Unique Identifier)**。
    * 本示例核心是 **Nordic UART Service (NUS)**，这是一个由 Nordic Semiconductor 定义的、广泛用于模拟串口通信的蓝牙服务。其服务 UUID 为 `6E400001-B5A3-F393-E0A9-E50E24DCCA9E` (在代码中用 `BT_UUID_NUS_VAL` 宏表示)。
    * **配置与应用**:
        * 可以设计和实现自己的**自定义服务 (Custom Service)**，使用自行生成的 128 位 UUID (例如通过 `uuidgen` 工具生成)。这允许你为特定应用创建独有的数据结构和功能接口。例如，一个智能灯泡可以有一个“灯光控制服务”，包含开关、亮度、颜色等特征。
* **特征 (Characteristic)**: 服务中具体的数据项，是 GATT 数据交换的基本单元。
    * 每个特征同样拥有一个唯一的 UUID，一个**值 (Value)** (实际数据存储的地方)，以及一组**属性 (Properties)** 来定义该特征值如何被访问 (如读 (Read)、写 (Write)、通知 (Notify)、指示 (Indicate)) 和**权限 (Permissions)** (如是否需要加密或认证才能访问)。
    * **NUS 的特征**:
        * **RX 特征 (Receive Characteristic)**:
            * UUID: `6E400002-B5A3-F393-E0A9-E50E24DCCA9E` (`BT_UUID_NUS_RX_VAL`)。
            * **视角**: 从中心设备 (Client) 的角度看，它是用于“接收”数据的通道；因此，对于外设 (Server) 来说，这个特征是用来被中心设备写入数据的。
            * **属性 (Properties)**: 通常具有 **可写 (Writeable)** 属性，特别是 **无响应写入 (Write Without Response)** 属性，允许客户端快速发送数据而无需等待服务器的 ATT 响应，从而提高吞吐量。也可以配置为**有响应写入 (Write With Response)**，可靠性更高但速度较慢。
        * **TX 特征 (Transmit Characteristic)**:
            * UUID: `6E400003-B5A3-F393-E0A9-E50E24DCCA9E` (`BT_UUID_NUS_TX_VAL`)。
            * **视角**: 从中心设备 (Client) 的角度看，它是用于“发送”数据的通道；因此，对于外设 (Server) 来说，这个特征是用来向中心设备发送数据的。
            * **属性 (Properties)**: 通常具有 **可通知 (Notifiable)** 属性。这意味着服务器可以在其值发生变化时，主动将新值通过 **GATT 通知 (GATT Notification)** 推送给已订阅的客户端。通知是单向的，服务器发送后不等待客户端的确认。
        * **配置与应用 (特征属性修改)**:
            * 为 RX 特征添加**读 (Read)** 属性：允许客户端在写入数据前先读取 RX 特征的当前状态或缓冲区信息（如果服务设计如此）。
            * 为 TX 特征使用**指示 (Indicate)** 属性代替通知：指示与通知类似，都是服务器主动向客户端发送数据，但指示要求客户端在收到数据后发送一个 **ATT 确认 (ATT Confirmation)** 给服务器。这提供了更可靠的数据传输，但由于需要额外的确认包，吞吐量会低于通知。适用于传输非常重要且不允许丢失的数据。
            * 定义特征的**最大长度 (Maximum Length)**。
* **描述符 (Descriptor)**: 提供关于特征的附加元数据 (Metadata)。
    * **客户端特性配置描述符 (CCCD - Client Characteristic Configuration Descriptor)**:
        * UUID: `0x2902` (标准16位 UUID)。
        * **作用**: 对于具有可通知 (Notifiable) 或可指示 (Indicatable) 属性的特征，必须包含此描述符。GATT 客户端通过向此描述符写入特定的值来开启或关闭来自服务器的通知/指示。
            * 写入 `0x0001`: 使能通知 (Enable Notifications)。
            * 写入 `0x0002`: 使能指示 (Enable Indications)。
            * 写入 `0x0000`: 禁止通知和指示 (Disable)。
        * CCCD 的值是每个客户端连接特有的，即一个客户端使能了通知，不代表其他连接到此服务器的客户端也自动使能了通知。
    * **配置与应用 (其他描述符)**:
        * **特征用户描述描述符 (Characteristic User Description Descriptor - UUID: `0x2901`)**: 可以为特征提供一个人类可读的字符串描述 (例如 "Serial Data Input Channel")，帮助客户端开发者理解特征的用途。
        * **特征表示格式描述符 (Characteristic Presentation Format Descriptor - UUID: `0x2904`)**: 定义特征值的格式 (如 uint8, string, float) 和单位 (如摄氏度, 米/秒)。

## 2. 代码实现分析 (`main.c` 结合 `prj.conf`) - 深度剖析

### 2.1. 初始化流程 (`main()` 函数及相关初始化函数)

`main()` 函数是应用程序的入口点，负责初始化硬件外设、蓝牙协议栈、相关服务以及应用程序逻辑。

1.  **GPIO 初始化 (`configure_gpio()`)**:
    * **源码关联**: `dk_leds_init()`。
    * **`prj.conf` 关联**: `CONFIG_DK_LIBRARY=y` (使能 Nordic DK 板级支持库)。
    * **参数配置与应用**:
        * `RUN_STATUS_LED` (默认为 `DK_LED1`) 和 `CON_STATUS_LED` (默认为 `DK_LED2`) 是宏定义，可以通过修改这些宏将其指向不同的 LED (如果开发板上有更多 LED 可用，并已在设备树中正确配置)。
        * **应用扩展**:
            * 可以设计更复杂的 LED 闪烁模式或颜色组合 (如果使用 RGB LED) 来指示更丰富的设备状态，例如：
                * 错误状态 (Error State): LED 快速红色闪烁。
                * 数据传输中 (Data Transferring): LED 蓝色呼吸灯效果。
                * 低电量警告 (Low Battery Warning): LED 黄色慢速闪烁。
            * 如果同时启用了按键 (`CONFIG_BT_NUS_SECURITY_ENABLED` 相关的 `dk_buttons_init`)，LED 可以与按键事件联动，提供用户交互反馈。

2.  **UART 初始化 (`uart_init()`)**:
    * **源码关联**:
        * `static const struct device *uart = DEVICE_DT_GET(DT_CHOSEN(nordic_nus_uart));` (通过设备树获取 UART 设备实例)。
        * `err = uart_callback_set(uart, uart_cb, NULL);` (注册 UART 事件回调)。
        * `err = uart_rx_enable(uart, rx->data, sizeof(rx->data), UART_WAIT_FOR_RX);` (启动 UART 接收)。
    * **`prj.conf` 关联**: `CONFIG_UART_ASYNC_API=y` (启用异步 UART API), `CONFIG_NRFX_UARTE0=y` (启用 UARTE0 驱动，通常 `nordic_nus_uart` 在设备树中指向 `&uarte0` 节点)。
    * **参数配置与应用**:
        * **设备树配置 (`DT_CHOSEN(nordic_nus_uart)`)**:
            * **修改**: 可以在项目的 `.overlay` 文件中 (例如 `nrf52832dk_nrf52832.overlay`) 修改 `chosen nordic_nus_uart` 节点，将其指向不同的 UART 硬件实例 (如 `&uart1`，前提是 nRF52832 有第二个 UART 接口且未被其他功能占用，如调试串口)。
                ```devicetree
                // Example .overlay content
                / {
                    chosen {
                        nordic_nus_uart = &uart1; // Change NUS to use uart1
                    };
                };

                &uart1 { // Ensure uart1 is configured
                    compatible = "nordic,nrf-uarte";
                    status = "okay";
                    current-speed = <115200>;
                    // Add other necessary properties like tx-pin, rx-pin, rts-pin, cts-pin if not default
                };
                ```
            * **应用**: 如果默认的 `&uarte0` 被用作系统控制台 (Console) 或 RTT 日志的物理输出 (虽然 RTT 通常不走物理 UART)，或者需要与一个不能更改接口的外部设备通信，将 NUS 服务配置到另一个 UART 接口可以避免冲突，提高系统设计的灵活性。
        * **UART 通信参数 (Baud Rate, Parity, Data bits, Stop bits)**:
            * **配置**: 这些参数通常不在 `prj.conf` 中直接配置，而是在设备树 (`.dts` 或特定板级的 `.overlay`) 中为相应的 UART 节点 (如 `&uarte0`) 进行配置。例如，`current-speed = <115200>;` 设置波特率为 115200。其他如 `parity` (e.g., `none`, `even`, `odd`), `data-bits` (e.g., `<8>`), `stop-bits` (e.g., `<1>`) 也可以配置。
            * **应用**:
                * **匹配外部设备**: 必须将 UART 参数配置为与连接的外部串口设备 (如传感器模块、GPS 模块、PC 串口终端) 完全一致，否则会导致通信乱码或失败。
                * **吞吐量与兼容性**: 更高的波特率 (如 1Mbps) 可以获得更高的数据传输速率，但对信号质量和线缆要求也更高，且某些旧设备可能不支持。较低的波特率 (如 9600bps) 兼容性好，但速率慢。
        * `CONFIG_UART_LINE_CTRL` (在 `prj.conf` 中配置):
            * **作用**: 如果使能 (通常用于 USB CDC ACM 虚拟串口)，`uart_init` 中的代码会处理 DTR (Data Terminal Ready) 等控制信号。
            * **应用**: 在与 PC 的 USB 虚拟串口通信时，某些终端软件会使用 DTR 信号来表示串口是否“打开”或“准备好”。等待 DTR 信号可以确保在 PC 端串口软件未打开时，设备不会盲目发送数据。也可以通过这些信号实现简单的硬件流控。
        * `UART_BUF_SIZE` (来自 `prj.conf` 的 `CONFIG_BT_NUS_UART_BUFFER_SIZE`):
            * **定义**: `struct uart_data_t` 结构体中 `data` 数组的大小，即单次 UART 接收或发送操作的缓冲区大小。
            * **应用**:
                * **增大此值**: 可以减少处理小数据块的次数，可能略微提高 UART 传输效率，尤其是在传输大数据块时。但会显著增加每次 `k_malloc` 分配的内存量，消耗更多 RAM。
                * **减小此值**: 节省 RAM，但如果数据到达频繁，可能导致更频繁的 `uart_cb` 回调和 `k_malloc`/`k_free` 操作，增加 CPU 开销。
                * **权衡**: 需要根据预期的平均数据包大小、实时性要求和 RAM 限制进行权衡。如果通常传输的是短命令，小缓冲区即可；如果传输文件或大数据流，可能需要更大的缓冲区（并配合流控）。
        * `UART_WAIT_FOR_RX` (来自 `prj.conf` 的 `CONFIG_BT_NUS_UART_RX_WAIT_TIME`):
            * **定义**: `uart_rx_enable` 的第三个参数，表示 UART 驱动在多长时间内没有新数据到达时触发接收超时，从而将当前已接收的数据上报给应用层。
            * **应用**:
                * **较小值 (e.g., 10-50ms)**: 响应更及时，即使数据不是以换行符结束，也能较快地被处理。适用于需要低延迟响应的交互式应用。
                * **较大值 (e.g., 100-500ms)**: 允许 UART 驱动累积更多数据再上报，可能减少 CPU 中断和回调次数，但会增加数据处理的延迟。适用于数据传输不频繁或对实时性要求不高的场景。
                * **`SYS_FOREVER_MS`**: 如果希望 UART 驱动仅在缓冲区满或接收到特定终止符 (如换行，如果驱动支持) 时才上报数据，可以设置一个非常大的超时值 (但要注意这可能导致数据长时间滞留在驱动缓冲区)。在本例中，代码逻辑是在收到换行符时显式调用 `uart_rx_disable` 来触发数据处理。

3.  **蓝牙安全认证回调注册**:
    * 在当前 `prj.conf` 中，由于 `CONFIG_BT_NUS_SECURITY_ENABLED` (或更通用的 `CONFIG_BT_SMP`) **未被定义**，`main.c` 中相关的 `bt_conn_auth_cb_register()` 和 `bt_conn_auth_info_cb_register()` 调用被 `#ifdef` 条件编译排除了。因此，当前的蓝牙连接是**不安全 (Unsecured)** 的，数据以明文传输。
    * **参数配置与应用 (如果启用安全)**:
        * **在 `prj.conf` 中启用**:
            ```conf
            CONFIG_BT_SMP=y                # 启用安全管理器协议 (Security Manager Protocol)
            CONFIG_BT_BONDING=y            # 启用绑定 (Bonding)，需要 CONFIG_BT_SETTINGS=y
            CONFIG_BT_SETTINGS_CCC_LAZY_LOADING=y # 推荐，惰性加载CCCD设置
            # 可选：启用LE Secure Connections (更强的安全性)
            CONFIG_BT_SMP_LESC_SUPPORT=y
            ```
        * **配置 I/O 能力 (I/O Capabilities)**: 在 `conn_auth_callbacks` 结构体中（如果被注册），可以设置回调函数来声明设备的 I/O 能力，例如是否有显示屏 (`passkey_display`)、是否有键盘 (`passkey_entry` - 虽然外设通常不主动输入)、是否有确认按钮 (`passkey_confirm`)。也可以通过 Kconfig (如 `CONFIG_BT_FIXED_PASSKEY`) 设置固定密钥。
            * `conn_auth_callbacks.passkey_display`: 若设备有显示屏，此回调被调用以显示配对码。
            * `conn_auth_callbacks.passkey_confirm`: 若使用 Numeric Comparison，此回调被调用，用户通过按键确认屏幕显示的密钥是否匹配。
            * `conn_auth_callbacks.cancel`: 配对取消时调用。
        * **决定配对方法 (Pairing Method)**: 基于双方的 I/O 能力和安全设置，SMP 会选择一种配对方法：
            * **Just Works**: 无需用户交互。安全性最低，易受中间人攻击 (MITM)。适用于对安全要求极低的场景。
            * **Passkey Entry**: 一方显示 6 位数字密钥，另一方输入。提供 MITM 保护。适用于至少一方有键盘和显示屏的场景。
            * **Numeric Comparison (LESC 专属)**: 双方设备上显示相同的 6 位数字，用户确认是否匹配。提供 MITM 保护。适用于双方都有显示屏和确认能力的设备。
            * **Out Of Band (OOB)**: 通过蓝牙之外的渠道 (如 NFC, QR码) 交换临时密钥 (TK) 或其他鉴权数据。可以提供非常强的安全性。
        * **应用**:
            * **保护敏感数据**: 对于通过 NUS 传输的数据如果是私密的或关键的 (例如配置参数、控制命令、个人信息)，必须启用安全连接。推荐使用 **LE Secure Connections (LESC)** 配合 **Passkey Entry** 或 **Numeric Comparison** 以获得较强的 MITM 保护。
            * **用户便利性与安全平衡**: 如果产品没有显示或输入能力，可以考虑 Just Works (若数据不敏感) 或 OOB (若能实现)。
            * **绑定 (Bonding)**: 启用绑定后，配对成功产生的长期密钥 (LTK) 会被存储。下次连接时，设备可以直接使用 LTK 恢复加密会话，无需重复完整的配对过程，大大提升用户体验。

4.  **使能蓝牙协议栈 (`bt_enable(NULL)`)**:
    * `NULL` 参数表示使用 Zephyr 蓝牙协议栈的默认初始化参数。也可以传入一个 `bt_ready_cb_t` 类型的回调函数，在协议栈准备就绪后被调用，但本例中使用信号量 `ble_init_ok` 进行同步。

5.  **加载持久化设置 (`settings_load()`)**:
    * **`prj.conf` 关联**: `CONFIG_BT_SETTINGS=y` (已配置)，依赖 `CONFIG_FLASH=y`, `CONFIG_FLASH_PAGE_LAYOUT=y`, `CONFIG_FLASH_MAP=y`。
    * **作用**: 从非易失性存储 (Flash) 中加载之前存储的蓝牙设置，最主要的是**绑定信息 (Bonding Information)**，包括已配对设备的长期密钥 (LTK)、身份解析密钥 (IRK)、连接签名解析密钥 (CSRK) 以及可能的GATT CCCD 配置。
    * **应用**:
        * **无缝重连**: 当一个已绑定的中心设备尝试重新连接时，如果外设已加载了绑定信息，它们可以快速恢复加密会_VOIS_SESSION，无需用户再次干预进行配对。
        * **CCCD 持久化**: `CONFIG_BT_SETTINGS_CCC_LAZY_LOADING=y` 或 `CONFIG_BT_SETTINGS_CCC_STORE_ON_WRITE=y` 可以让服务器记住客户端对通知/指示的订阅状态。这样即使设备重启，之前订阅了通知的客户端重新连接后，服务器仍能自动开始发送通知，而无需客户端再次写入 CCCD。这对某些应用场景非常重要。
        * **禁用场景**: 如果不希望设备记住任何绑定信息，或者每次连接都强制重新配对 (例如某些测试场景或特定安全要求的应用)，可以禁用 `CONFIG_BT_SETTINGS`。

6.  **初始化 NUS 服务 (`bt_nus_init(&nus_cb)`)**:
    * **`prj.conf` 关联**: `CONFIG_BT_NUS=y` (已配置)。
    * **作用**: 调用此函数会在 GATT 服务器的属性表中注册 NUS 服务及其两个特征 (TX 和 RX) 和相关的描述符 (如 TX 特征的 CCCD)。同时，它将传入的 `nus_cb` 结构体中的回调函数与 NUS 服务的特定事件关联起来。
    * `nus_cb.received = bt_receive_cb;`: 这是最重要的回调。当GATT客户端向本设备的 NUS RX 特征执行写操作时，蓝牙协议栈会最终调用此 `bt_receive_cb` 函数，并将接收到的数据作为参数传入。

7.  **初始化并启动广播 (`advertising_start()` 通过 `adv_work_handler`)**:
    * **源码关联**:
        ```c
        k_work_init(&adv_work, adv_work_handler); // 初始化工作项
        advertising_start();                     // 提交工作项以启动广播
        // ...
        static void adv_work_handler(struct k_work *work) {
            // BT_LE_ADV_CONN_FAST_2 是一个使用Kconfig预定义参数的便捷宏
            int err = bt_le_adv_start(BT_LE_ADV_CONN_FAST_2, ad, ARRAY_SIZE(ad), sd, ARRAY_SIZE(sd));
            // ...
        }
        ```
    * **`prj.conf` 关联**: `CONFIG_BT_PERIPHERAL=y`。
    * **参数配置与应用**:
        * **`BT_LE_ADV_CONN_FAST_2`**: 这是一个 Zephyr 提供的预定义广播参数集，通常表示使用较短的广播间隔以实现快速连接。这些参数的实际值 (如最小/最大广播间隔、广播超时) 是通过 Kconfig 选项定义的，例如：
            * `CONFIG_BT_GAP_ADV_FAST_INT_MIN_1` (例如，默认 30ms，即 `0x0030` = 48 * 0.625ms)
            * `CONFIG_BT_GAP_ADV_FAST_INT_MAX_1` (例如，默认 60ms，即 `0x0060` = 96 * 0.625ms)
            * `CONFIG_BT_GAP_ADV_FAST_TIMEOUT` (例如，默认 30 秒，快速广播持续时间)
            * Zephyr 通常还支持 `BT_LE_ADV_CONN_SLOW_2` 等，对应不同的慢速广播参数。
        * **应用 (广播策略)**:
            * **平衡发现速度与功耗**: 许多设备采用的策略是：启动后先以**快速间隔 (Fast Interval)** 广播一段时间 (由 `CONFIG_BT_GAP_ADV_FAST_TIMEOUT` 控制)，如果在这段时间内没有设备连接，则自动切换到**慢速间隔 (Slow Interval)** (由 `CONFIG_BT_GAP_ADV_SLOW_INT_MIN_1`, `MAX_1` 等定义) 以节省功耗。这种行为通常由 `CONFIG_BT_GAP_AUTO_UPDATE_CONN_PARAMS` (虽然名字带 conn_params，但广义上影响 GAP 行为) 和相关的超时 Kconfig 选项控制。
            * **始终快速广播**: 如果应用场景要求设备始终能被快速发现，并且功耗不是首要考虑因素 (例如，市电供电设备)，可以将快速广播超时设置得非常长或者禁用慢速广播。
            * **始终慢速广播**: 如果设备对功耗极其敏感，并且不需要立即被发现 (例如，一个后台运行的、仅偶尔需要连接的传感器)，可以直接配置为使用慢速广播间隔。
        * **自定义广播参数结构体 (`struct bt_le_adv_param`)**:
            * 除了使用预定义的参数集 (如 `BT_LE_ADV_CONN_FAST_2`)，开发者可以通过填充一个 `struct bt_le_adv_param` 结构体，并将其作为 `bt_le_adv_start` 的第一个参数，来实现对广播行为更精细的控制。
            * `struct bt_le_adv_param` 包含的字段：
                * `id`: 使用的身份标识 (通常为 `BT_ID_DEFAULT`)。
                * `sid`: 广播集标识 (Advertising Set ID)，用于扩展广播和周期性广播。对于传统广播，通常为0。
                * `secondary_max_skip`: 用于扩展广播。
                * `options`: 位掩码，用于设置广播选项，如：
                    * `BT_LE_ADV_OPT_CONNECTABLE`: 广播是可连接的。
                    * `BT_LE_ADV_OPT_USE_NAME`: 尝试在广播数据中包含设备名称 (如果空间足够且 `ad` 中未指定)。
                    * `BT_LE_ADV_OPT_ONE_TIME`: 广播仅在一次连接后停止 (如果使用定向广播)。
                    * `BT_LE_ADV_OPT_USE_IDENTITY`: 使用身份地址而不是 RPA (可解析私有地址) 进行广播。
                    * `BT_LE_ADV_OPT_DIR_ADDR_RPA`: 用于定向广播时，对端地址是否为 RPA。
                * `interval_min`, `interval_max`: 最小和最大广播间隔 (单位为 0.625ms)。链路层会在这两个值之间选择一个实际间隔。
                * `peer`: 用于定向广播的目标对端地址。
            * **应用 (自定义广播类型)**:
                * **信标 (Beacon) 实现**: 设置 `options` 为非可连接 (清除 `BT_LE_ADV_OPT_CONNECTABLE`)，并可能使用 `BT_LE_ADV_SCAN_IND` (允许扫描响应但不可连接) 或 `BT_LE_ADV_NONCONN_IND` (完全不可扫描也不可连接的纯广播)。然后在 `ad` 中放入 iBeacon 或 Eddystone 格式的制造商特定数据。
                * **传感器数据纯广播**: 某些低功耗传感器可以将少量数据直接放入 `BT_LE_ADV_NONCONN_IND` 类型的广播包中，中心设备只需进行被动扫描 (Passive Scan) 即可获取数据，完全无需建立连接，极大节省双方功耗。

8.  **主循环 (`for (;;)` in `main()`)**:
    * 在所有初始化完成后，`main` 函数进入一个无限循环，周期性地闪烁 LED1 并调用 `k_sleep(K_MSEC(RUN_LED_BLINK_INTERVAL))`。
    * `k_sleep()` 会使当前线程 (即 `main` 线程) 进入休眠状态，将 CPU 控制权让渡给其他优先级更高或已就绪的线程 (如系统工作队列线程、蓝牙协议栈的内部线程、`ble_write_thread` 等)。这对于 RTOS 系统至关重要，避免了 `main` 线程空占 CPU。实际的蓝牙事件处理和数据交互都是由中断服务程序 (ISR)、蓝牙协议栈注册的回调函数以及应用程序创建的其他线程 (如 `ble_write_thread`) 异步驱动的。

### 2.2. 广播的实现 (GAP Peripheral Role) - 更多细节

* **广播数据 `ad[]` (Advertising Data) 和扫描响应数据 `sd[]` (Scan Response Data) 的内容与结构**:
    * 每个 `bt_data` 结构体包含三个字段：`type` (AD Type), `data_len` (数据长度), `data` (指向数据的指针)。
    * **`ad[]` (广播数据包内容)**:
        ```c
        static const struct bt_data ad[] = {
            // AD Element 1: Flags
            BT_DATA_BYTES(BT_DATA_FLAGS, (BT_LE_AD_GENERAL | BT_LE_AD_NO_BREDR)),
            // AD Element 2: Complete Local Name
            BT_DATA(BT_DATA_NAME_COMPLETE, DEVICE_NAME, DEVICE_NAME_LEN),
        };
        ```
        * `BT_DATA_BYTES` 是一个宏，用于方便地定义包含字节数组数据的 AD 元素。
        * `BT_DATA` 是一个宏，用于方便地定义包含字符串 (如设备名称) 或其他数据块的 AD 元素。
    * **`sd[]` (扫描响应数据包内容)**:
        ```c
        static const struct bt_data sd[] = {
            // AD Element 1: Complete List of 128-bit Service Class UUIDs
            BT_DATA_BYTES(BT_DATA_UUID128_ALL, BT_UUID_NUS_VAL),
        };
        ```
    * **参数配置与应用**:
        * **Flags (`BT_DATA_FLAGS`)**:
            * **`BT_LE_AD_LIMITED_DISCOVERABLE_MODE`**: 可以替换 `BT_LE_AD_GENERAL`。设备只在有限时间内可被发现 (通常为 180 秒)。适用于特定场景，例如设备刚开机或用户按下某个按键后的一小段时间内允许配对或连接，之后自动停止广播或进入不可发现模式，以节省功耗或增强隐私。
        * **设备名称 (`BT_DATA_NAME_COMPLETE` 或 `BT_DATA_NAME_SHORTENED`)**:
            * **`CONFIG_BT_DEVICE_NAME`**: 在 `prj.conf` 中修改此值可以直接改变广播的设备名称。
            * 如果设备名称很长，导致 `ad` 数组的总长度超过 31 字节的限制 (对于传统广播)，可以使用 `BT_DATA_NAME_SHORTENED` 来广播一个缩短的设备名称，并将完整名称放在扫描响应 `sd` 中，或者通过GATT设备信息服务 (DIS) 提供。
            * **应用**: 根据产品需求定制设备名称。对于空间极其宝贵的广播包，可以考虑不广播名称，而仅在扫描响应中广播，或者让中心设备在连接后通过读取GATT中的设备信息服务 (Device Information Service - DIS) 来获取设备全名、型号、制造商等信息。
        * **服务 UUID (`BT_DATA_UUID..._ALL` 或 `BT_DATA_UUID..._SOME`)**:
            * 在扫描响应 `sd[]` 中广播 NUS 服务的 128 位 UUID，可以帮助中心设备在扫描阶段就识别出支持 NUS 的设备，从而进行有效的设备过滤或优先连接。
            * **应用**: 如果设备支持多个重要的服务，可以在 `ad[]` 或 `sd[]` (如果空间允许) 中列出它们的服务 UUID。可以使用 `BT_DATA_UUID16_ALL` (或 `_SOME`) 来广播一个或多个 16 位标准服务 UUID (如心率服务 `0x180D`, 电池服务 `0x180F`)，或使用 `BT_DATA_UUID128_ALL` (或 `_SOME`) 来广播 128 位自定义服务 UUID。这使得中心设备无需连接就能大致了解设备提供的功能。
        * **动态修改广播内容**:
            * 虽然本示例中 `ad` 和 `sd` 数组是静态定义的 (`static const`)，但实际应用中，可以通过在连接断开后，先调用 `bt_le_adv_stop()` 停止当前广播，然后修改一个非 `const` 的 `bt_data` 数组的内容 (或指向新的 `bt_data` 数组)，最后再调用 `bt_le_adv_start()` 来实现动态更新广播数据。
            * **应用**:
                * 根据设备当前状态 (例如电池电量低、有报警事件发生、传感器读数达到某个阈值) 在广播数据中嵌入少量状态信息，使得中心设备无需连接即可获取关键状态。
                * 实现一个“查找我的设备”功能，当用户触发查找时，设备开始以特定的广播数据 (可能包含一个唯一的标识符或特定的制造商数据) 进行广播，并可能同时发出声音或闪烁LED。

### 2.3. 连接与断开处理 (`conn_callbacks`) - 深度细节

Bluetooth 连接和断开事件由通过 `BT_CONN_CB_DEFINE(conn_callbacks)` 宏注册的一组回调函数来处理。

* **`connected(struct bt_conn *conn, uint8_t err)` 回调**:
    * `conn`: 指向代表此新连接的 `bt_conn` 连接对象的指针。此对象包含了连接的所有状态信息，如对端地址、连接参数、安全级别等。
    * `err`: 连接状态。`0` 表示成功，非 `0` 表示连接失败的错误码 (定义在 `zephyr/bluetooth/hci_err.h` 中，可通过 `bt_hci_err_to_str(err)` 转换为字符串)。
    * `current_conn = bt_conn_ref(conn);`: **引用计数 (Reference Counting)** 是 Zephyr 中管理内核对象生命周期的重要机制。`bt_conn` 对象由蓝牙协议栈创建和管理。当回调函数被调用时，传入的 `conn` 指针是有效的。但是，如果应用程序需要在回调之外的地方 (例如在其他线程或后续事件中) 继续使用这个连接对象，就必须通过 `bt_conn_ref(conn)` 来增加其引用计数。这告诉协议栈，应用程序正在使用此对象，请不要释放它。
    * **后续操作**: 通常在连接成功后，会进行服务发现 (如果是中心设备)、使能通知/指示、或准备数据交换。外设通常会停止广播 (这是默认行为，除非特别配置为在连接后继续广播，这在需要同时被多个中心设备发现或连接的场景下有用)。

* **`disconnected(struct bt_conn *conn, uint8_t reason)` 回调**:
    * `conn`: 指向刚刚断开的连接的 `bt_conn` 对象。
    * `reason`: 断开原因的 HCI 错误码 (如 `BT_HCI_ERR_REMOTE_USER_TERM_CONN` 表示对端用户终止连接，`BT_HCI_ERR_CONN_TIMEOUT` 表示连接超时)。
    * `bt_conn_unref(current_conn);`: 当应用程序不再需要之前通过 `bt_conn_ref()` 持有的连接对象引用时 (例如，连接已断开，不再通过此对象发送数据)，**必须**调用 `bt_conn_unref(current_conn)` 来减少引用计数。当连接对象的引用计数降为 0 时，蓝牙协议栈才能安全地回收该对象占用的资源。**忘记调用 `bt_conn_unref` 会导致连接对象无法被释放，最终可能导致系统因无法创建新的连接而出错 (内存泄漏或连接资源耗尽)。**
    * `current_conn = NULL;`: 将全局指针清空，避免悬挂指针。
    * `dk_set_led_off(CON_STATUS_LED);`: 更新 LED 状态。

* **`recycled_cb(void)` 回调**:
    * **作用**: 当一个 `bt_conn` 对象的所有引用 (包括协议栈内部的和应用程序通过 `bt_conn_ref` 持有的) 都被释放，并且该对象所占用的内存和资源已被协议栈完全回收，可以被用于新的连接时，这个回调函数会被触发。
    * **意义**: 它标志着与前一个连接相关的清理工作已彻底完成。在本示例中，在此回调中调用 `advertising_start()` 来重新开始广播是一种非常稳健的做法，确保了在尝试开始新的广播之前，不会有任何与旧连接相关的状态冲突或资源问题。这在设备需要频繁断开和重新连接的场景下尤为重要。

* **参数配置与应用 (扩展回调功能)**:
    * **在 `connected` 回调中**:
        * **请求连接参数更新**: 外设可以在连接建立后，如果发现中心设备给出的初始连接参数不理想 (例如，连接间隔太短导致功耗过高，或太长导致延迟过大)，可以通过 `bt_conn_le_param_update()` 函数向中心设备请求更新连接参数。
            * **应用**: 智能手表在与手机同步大量数据时可能请求较短的连接间隔以提高吞吐量；同步完成后，则请求较长的连接间隔和一定的从设备延迟以节省功耗。
        * **启动认证/加密**: 如果服务或特征需要安全保护，可以在连接后立即调用 `bt_conn_set_security(conn, BT_SECURITY_L2)` (或更高安全级别) 来发起安全程序 (配对/加密)。
        * **执行应用层握手**: 发送特定的初始化数据包或版本信息给中心设备。
    * **在 `disconnected` 回调中**:
        * **保存未完成状态**: 如果有正在进行的数据传输或操作因连接意外断开而中断，可以在此回调中保存相关状态，以便下次连接时恢复或提示用户。
        * **触发报警**: 对于某些关键应用，意外的连接断开可能需要触发本地报警 (声音、震动、其他方式通知用户)。
        * **尝试重新连接 (中心设备)**: 如果是中心设备，可以在断开后尝试自动重新连接到之前绑定的外设。

### 2.4. GATT 服务 (NUS) 的具体工作方式 - 深度细节

Nordic UART Service (NUS) 的核心功能在于模拟一个双向的串口数据通道。

* **数据接收 (BLE RX -> UART TX) - 通过 `bt_receive_cb` 回调处理**:
    * 当中心设备 (GATT 客户端) 向 NUS 的 **RX 特征 (Characteristic)** (UUID: `...0002...`) 执行 GATT 写操作 (Write Operation) 时，蓝牙协议栈最终会调用已注册的 `bt_receive_cb` 函数。
    * **动态内存分配 (`k_malloc` for `struct uart_data_t`)**:
        * `CONFIG_HEAP_MEM_POOL_SIZE=2048` (在 `prj.conf` 中) 定义了系统堆 (System Heap) 的大小。
        * 代码中 `struct uart_data_t *tx = k_malloc(sizeof(*tx));` 为每一个要通过 UART 发送的数据块动态分配内存。如果从 BLE 接收数据的速率非常快，或者 `uart_tx()` 函数处理较慢导致 `fifo_uart_tx_data` 队列中累积了大量待发送的数据块，频繁的 `k_malloc()` 调用可能会耗尽可用的堆内存，导致 `tx` 指针为 `NULL` (分配失败)。
        * **应用 (健壮性设计)**:
            * **必须检查 `k_malloc` 的返回值**: 如果 `tx == NULL`，表示内存分配失败。此时应采取适当的错误处理措施，例如：
                * 记录错误日志 (`LOG_WRN("Not able to allocate UART send data buffer");`)。
                * 简单地丢弃当前从 BLE 收到的数据包 (可能导致数据丢失)。
                * 如果应用层协议允许，可以通过 BLE 回复一个错误状态给中心设备。
                * 实现更复杂的**流控制 (Flow Control)** 机制，例如，暂时停止从 BLE 读取更多数据，或者使用一个预分配的内存池 (Memory Pool) 代替堆分配，以获得更可预测的内存行为。
    * **FIFO 队列 (`fifo_uart_tx_data`)**:
        * `K_FIFO_DEFINE(fifo_uart_tx_data)`: 这个宏在编译时定义了一个先进先出 (FIFO) 队列。其容量（能容纳多少个 `uart_data_t*` 指针）通常是固定的，并可能受到系统 Kconfig (如 `CONFIG_MAX_DOMAIN_DATA_MSG_SIZE` 或 Zephyr 内核默认值) 的影响。
        * **作用**: 当 `uart_tx()` 因为 UART 硬件忙于发送上一个数据包而不能立即发送当前数据块时，包含待发送数据的 `uart_data_t` 结构体指针会被放入此队列 (`k_fifo_put(&fifo_uart_tx_data, tx);`)。
        * 当 UART 发送完成一个数据块后，`uart_cb` 中的 `UART_TX_DONE` 事件处理逻辑会尝试从 `fifo_uart_tx_data` 队列中取出下一个待发送的数据块 (`buf = k_fifo_get(&fifo_uart_tx_data, K_NO_WAIT);`) 并通过 `uart_tx()` 发送。
        * **应用**:
            * **缓冲突发数据**: 如果 BLE 端瞬间写入大量短数据，这个 FIFO 可以起到缓冲作用，平滑 UART 的输出。
            * **队列满处理**: 如果生产数据的速度 (来自BLE) 持续远快于消费数据的速度 (UART硬件发送)，此 FIFO 队列最终可能会满。`k_fifo_put` 在队列满时如果使用 `K_NO_WAIT` 会立即返回错误 (通常是 `-ENOMEM` 或类似)。示例代码中没有显式处理 `k_fifo_put` 的返回值，但在高负载下这可能是一个潜在问题点。更健壮的设计可能需要检查返回值，并在队列满时采取措施 (如丢弃数据、等待、或向上层反馈)。
            * **队列大小调整**: 如果需要处理更大的数据突发，可能需要通过 Kconfig 调整相关配置以间接增加 FIFO 的容量，或者考虑使用自定义大小的内核消息队列 (Message Queue) 代替简单 FIFO。
    * **数据流总结**: 数据从 BLE 中心设备写入 NUS RX 特征 -> 触发 `bt_receive_cb` -> (动态分配 `uart_data_t` 内存) -> 数据拷贝 -> 尝试 `uart_tx()` 直接发送 -> 若失败，则数据指针入队 `fifo_uart_tx_data` -> (UART空闲时) `uart_cb` 的 `UART_TX_DONE` 事件处理逻辑从队列取出数据并通过 `uart_tx()` 发送 -> (数据发送完成后) `uart_cb` 的 `UART_TX_DONE` 再次触发，释放已发送的 `uart_data_t` 内存。

* **数据发送 (UART RX -> BLE TX) - 通过 `ble_write_thread` 线程和 `uart_cb` 回调处理**:
    * **步骤1: UART 数据接收并入队 (`uart_cb` 中处理 `UART_RX_BUF_RELEASED` 事件)**:
        ```c
        // 在 uart_cb 函数的 switch(evt->type) 中:
        case UART_RX_BUF_RELEASED:
            LOG_DBG("UART_RX_BUF_RELEASED");
            buf = CONTAINER_OF(evt->data.rx_buf.buf, struct uart_data_t, data[0]);
            if (buf->len > 0) { // 确保缓冲区中有有效数据
                k_fifo_put(&fifo_uart_rx_data, buf); // 将包含 UART 数据的缓冲区指针放入 fifo_uart_rx_data
            } else {
                k_free(buf); // 如果没有数据 (例如，只是一个空的缓冲区被释放)，则释放内存
            }
            break;
        ```
        * **背景**: Zephyr 的异步 UART API 通常采用双缓冲或环形缓冲机制。当 UART 驱动程序接收数据填充了一个缓冲区后 (例如，接收到换行符导致 `uart_rx_disable` 被调用，或者内部缓冲区满，或者超时 `UART_WAIT_FOR_RX`)，它会触发一个事件。当应用程序处理完这个缓冲区的数据，并为 UART 驱动提供了新的空闲缓冲区后，包含已接收数据的旧缓冲区会被“释放”回给应用程序，此时触发 `UART_RX_BUF_RELEASED` 事件。
        * **流程**: 在这个事件回调中，包含从物理 UART 接收到的有效数据的 `uart_data_t` 结构体指针 (`buf`) 被放入 `fifo_uart_rx_data` 队列，等待 `ble_write_thread` 线程进行处理。
    * **步骤2: `ble_write_thread` 线程从队列取出数据并通过 BLE 发送**:
        ```c
        K_THREAD_DEFINE(ble_write_thread_id, STACKSIZE, ble_write_thread, NULL, NULL,
                        NULL, PRIORITY, 0, 0);

        void ble_write_thread(void)
        {
            k_sem_take(&ble_init_ok, K_FOREVER); // 阻塞等待蓝牙初始化完成信号
            struct uart_data_t nus_data = { .len = 0, }; // 用于聚合数据的临时缓冲区

            for (;;) { // 无限循环
                struct uart_data_t *buf = k_fifo_get(&fifo_uart_rx_data, K_FOREVER); // 阻塞等待从 fifo_uart_rx_data 获取数据

                int plen = MIN(sizeof(nus_data.data) - nus_data.len, buf->len); // 计算本次可以拷贝到 nus_data 的长度
                int loc = 0; // buf 中的当前读取位置

                while (plen > 0) { // 如果 buf 中的数据还没读完
                    memcpy(&nus_data.data[nus_data.len], &buf->data[loc], plen); // 拷贝数据到 nus_data
                    nus_data.len += plen;
                    loc += plen;

                    // 当 nus_data 缓冲区满，或者遇到换行符时，准备发送
                    if (nus_data.len >= sizeof(nus_data.data) ||
                       (nus_data.data[nus_data.len - 1] == '\n') ||
                       (nus_data.data[nus_data.len - 1] == '\r')) {
                        if (bt_nus_send(current_conn, nus_data.data, nus_data.len)) { // 通过 NUS TX 特征发送通知
                            LOG_WRN("Failed to send data over BLE connection");
                        }
                        nus_data.len = 0; // 清空 nus_data 准备下次聚合
                    }
                    plen = MIN(sizeof(nus_data.data) - nus_data.len, buf->len - loc); // 计算下一次可以拷贝的长度
                }
                k_free(buf); // 释放从 FIFO 取出的 uart_data_t 结构体（它是由 uart_cb 中 k_malloc 分配的）
            }
        }
        ```
        * **线程创建与同步**: `K_THREAD_DEFINE` 创建了一个名为 `ble_write_thread_id` 的内核线程，其执行函数是 `ble_write_thread`，具有指定的堆栈大小 (`STACKSIZE` 来自 `CONFIG_BT_NUS_THREAD_STACK_SIZE`) 和优先级 (`PRIORITY`)。线程开始时会调用 `k_sem_take(&ble_init_ok, K_FOREVER)`，这将使线程阻塞，直到 `main()` 函数中的 `bt_enable()` 成功完成并调用 `k_sem_give(&ble_init_ok)`。这确保了在尝试使用蓝牙功能之前，蓝牙协议栈已经初始化完毕。
        * **数据获取**: 线程在其主循环中调用 `k_fifo_get(&fifo_uart_rx_data, K_FOREVER)`。这是一个阻塞操作，如果 `fifo_uart_rx_data` 队列为空，线程将在此挂起，直到队列中有数据被放入 (由 `uart_cb` 完成)。
        * **数据发送 (`bt_nus_send()`)**:
            * `bt_nus_send(current_conn, nus_data.data, nus_data.len)`: 这个函数是 Nordic 提供的 NUS 服务库的一部分。它会构建一个 GATT 通知 (Notification)，并将 `nus_data.data` 中的数据作为该通知的 payload，通过 NUS 的 **TX 特征 (Characteristic)** (UUID: `...0003...`) 发送给由 `current_conn` 指定的已连接客户端。
            * **前提条件**: 为了让中心设备能够接收到这些通知，中心设备必须在连接后，通过 GATT 操作向该 NUS TX 特征的 **CCCD (Client Characteristic Configuration Descriptor)** 写入值 `0x0001` (使能通知)。如果客户端没有订阅通知，`bt_nus_send` 调用可能仍然会成功返回 (表示数据已提交给协议栈底层)，但数据不会实际传输到客户端。
            * **第一个参数 `current_conn`**: 在 `main.c` 的原始版本中，这里传入的是 `NULL`。`bt_nus_send` 的实现通常是如果第一个参数为 `NULL`，它会尝试发送给第一个已连接且已订阅该特征的客户端。传入明确的 `current_conn` (如果只有一个连接，这通常是正确的) 可以更精确地指定目标。对于多连接场景，这里需要更复杂的逻辑来确定要发送给哪个连接。
        * **MTU (Maximum Transmission Unit - 最大传输单元) 和数据分片**:
            * 一个 GATT 通知所能携带的实际用户数据 (payload) 的最大长度受限于当前连接的 **ATT MTU**。ATT MTU 减去 ATT 操作码 (Opcode, 1字节) 和属性句柄 (Attribute Handle, 2字节) 后，才是GATT特征值所能占用的空间。例如，如果 ATT MTU 是 23 字节 (默认值)，那么一个通知最多能携带 23 - 3 = 20 字节的用户数据。
            * `bt_nus_send()` 函数内部会处理数据分片：如果 `nus_data.len` 大于 `(ATT_MTU - 3)`，`bt_nus_send` 会自动将数据分割成多个适合 MTU 大小的GATT 通知包，并依次发送。这意味着应用程序层面通常不需要关心 MTU 分片细节，只需调用 `bt_nus_send` 即可。
            * **配置与应用 (MTU 优化)**:
                * 在 `prj.conf` 中设置 `CONFIG_BT_L2CAP_TX_MTU` (例如 `CONFIG_BT_L2CAP_TX_MTU=247`) 可以让本设备在 MTU 交换时提议一个较大的 L2CAP MTU，这通常会间接导致协商出较大的 ATT MTU (最大可达 `CONFIG_BT_L2CAP_TX_MTU` 或 `CONFIG_BT_L2CAP_RX_MTU` 中较小者，并受对端设备能力限制)。
                * 中心设备也可以主动发起 MTU 交换请求。当 MTU 交换完成后，双方会使用新的、通常更大的 MTU 进行通信。
                * **应用**: 对于需要传输大量数据的应用 (如固件更新 Over-the-Air DFU, 日志传输，传感器波形数据)，配置并成功协商一个较大的 MTU (例如 247 字节) 可以显著减少 GATT 操作的次数和协议开销，从而提高有效数据吞吐量。
        * **数据聚合逻辑 (`nus_data` 在 `ble_write_thread`)**:
            * 代码中的 `nus_data` 结构体用于累积从 `fifo_uart_rx_data` 队列中取出的数据，直到 `nus_data.data` 缓冲区即将填满 (`nus_data.len >= sizeof(nus_data.data)`)，或者在累积的数据末尾检测到换行符 (`\n` 或 `\r`) 时，才会调用 `bt_nus_send()` 将聚合的数据一次性发送出去。`sizeof(nus_data.data)` 的大小与 `UART_BUF_SIZE` (即 `CONFIG_BT_NUS_UART_BUFFER_SIZE`) 相同。
            * **应用 (自定义数据包化策略)**:
                * **实时性优先**: 如果应用场景要求 UART 上接收到的每个字符或非常短的数据片段都尽快通过 BLE 发送出去 (例如，模拟一个非常低延迟的按键输入)，可以修改此逻辑，在 `k_fifo_get` 取到任何非空数据后，不进行聚合，直接调用 `bt_nus_send`。但这会增加 BLE 的包开销和功耗。
                * **固定长度数据包**: 如果应用层协议要求数据以固定长度的包进行传输，可以在这里实现逻辑，累积到指定长度后再发送。
                * **特定结束符**: 如果数据流使用特定的结束符序列 (除了 `\n` 或 `\r`) 来标记一个完整消息的结束，需要修改判断条件。
                * **基于消息头的长度字段**: 如果每个消息的头部包含一个长度字段，可以先读取头部，解析出消息总长度，然后累积到该长度后再发送。
        * `k_free(buf);`: 在 `ble_write_thread` 中，当从 `fifo_uart_rx_data` 获取的 `buf` (它指向一个由 `uart_cb` 中 `k_malloc` 分配的 `uart_data_t` 结构体) 中的数据被完全处理并拷贝到 `nus_data` 后，必须调用 `k_free(buf)` 来释放这个结构体占用的内存，防止内存泄漏。

## 3. 整体工作流程梳理 (深度回顾)

1.  **初始化 (Initialization)**:
    * 系统启动，`main()` 函数按顺序执行硬件外设 (GPIO for LEDs, UART for serial communication) 的初始化。
    * 注册蓝牙连接事件回调 (`conn_callbacks`)。
    * 使能蓝牙协议栈 (`bt_enable`)。
    * 加载持久化设置 (`settings_load`)，主要是为了恢复已绑定的设备信息。
    * 初始化 Nordic UART Service (`bt_nus_init`)，注册 NUS 事件回调 (特别是 `bt_receive_cb`)。
    * 初始化并启动用于 BLE 数据发送的独立线程 `ble_write_thread` (它会等待蓝牙初始化完成的信号)。
    * 通过工作队列异步启动蓝牙广播 (`advertising_start`)。
    * `main` 线程进入一个低优先级循环，仅用于闪烁状态指示 LED。
    * **关键配置**: `prj.conf` 文件中的 `CONFIG_` 选项决定了哪些模块被编译到固件中，以及它们的默认参数 (如设备名称、广播间隔范围、堆栈大小、UART 缓冲区大小等)。设备树 (`.dts` 和 `.overlay` 文件) 定义了硬件资源 (如哪个 UART 控制器被 NUS 使用，其引脚和波特率等)。

2.  **广播阶段 (Advertising Phase)**:
    * 设备根据 `ad` (广播数据：Flags, 设备名) 和 `sd` (扫描响应数据：NUS 服务 UUID) 的内容，以及由 `BT_LE_ADV_CONN_FAST_2` (其参数源自 Kconfig) 决定的广播参数 (类型、间隔) 进行广播。
    * LED1 (RUN\_STATUS\_LED) 周期性闪烁，表示程序正在运行。
    * 设备处于可被发现和可连接状态，等待中心设备 (如手机上的 nRF Connect App) 的操作。

3.  **连接阶段 (Connection Phase)**:
    * 当中心设备扫描到此外设并发起连接请求时，如果连接成功：
        * 蓝牙协议栈调用 `connected` 回调函数。
        * `current_conn` 全局变量被设置并增加引用计数，指向代表此连接的 `bt_conn` 对象。
        * LED2 (CON\_STATUS\_LED) 被点亮，表示蓝牙已连接。
        * 外设通常会自动停止广播 (这是默认行为)。

4.  **数据交互阶段 (Data Exchange Phase - 双向)**:
    * **中心设备 -> 外设 (例如，手机App发送数据到nRF52832)**:
        1.  手机 App (GATT 客户端) 向 nRF52832 (GATT 服务器) 上的 NUS 服务的 **RX 特征**执行 GATT **写操作 (Write Operation)**。
        2.  nRF52832 的蓝牙协议栈接收到这个写请求，验证通过后，调用已注册的 `bt_receive_cb` 函数，并将接收到的数据 (`data` 和 `len`) 作为参数传递。
        3.  在 `bt_receive_cb` 中，数据被拷贝到动态分配的 `uart_data_t` 缓冲区中。
        4.  该缓冲区的数据通过 `uart_tx()` 尝试直接发送到物理 UART 端口。如果 UART 忙，则该缓冲区指针被放入 `fifo_uart_tx_data` 队列。
        5.  当 UART 发送空闲时，`uart_cb` 的 `UART_TX_DONE` 事件会从 `fifo_uart_tx_data` 队列中取出数据继续发送。
        6.  数据最终从 nRF52832 的 UART TX 引脚输出。
    * **外设 -> 中心设备 (例如，nRF52832通过物理UART接收数据并发送到手机App)**:
        1.  外部设备 (如 PC 串口终端) 通过物理 UART 发送数据到 nRF52832 的 UART RX 引脚。
        2.  nRF52832 的 UART 硬件驱动接收数据，当一个数据块接收完成时 (例如，收到换行符，或内部缓冲区满，或接收超时)，会触发 `uart_cb` 回调 (特别是 `UART_RX_BUF_RELEASED` 事件，在旧缓冲区被释放给应用层时)。
        3.  在 `uart_cb` 的 `UART_RX_BUF_RELEASED` 处理逻辑中，包含接收数据的 `uart_data_t` 缓冲区指针被放入 `fifo_uart_rx_data` 队列。
        4.  `ble_write_thread` 线程在其主循环中通过 `k_fifo_get()` (阻塞方式) 从 `fifo_uart_rx_data` 队列中取出数据。
        5.  线程对取出的数据进行聚合 (填充 `nus_data` 直至满或遇到换行符)。
        6.  调用 `bt_nus_send(current_conn, nus_data.data, nus_data.len)` 将聚合后的数据通过 NUS 服务的 **TX 特征**以 **GATT 通知 (Notification)** 的形式发送给已连接且已订阅此特征通知的中心设备 (手机 App)。
        7.  手机 App 接收到通知并显示数据。

5.  **断开连接阶段 (Disconnection Phase)**:
    * 当连接由于任何原因 (中心设备主动断开、外设主动断开 - 本例未实现、链路丢失导致超时等) 断开时：
        * 蓝牙协议栈调用 `disconnected` 回调函数。
        * `current_conn` 的引用被释放，并设为 `NULL`。
        * LED2 (CON\_STATUS\_LED) 被熄灭。
        * 随后，当连接对象资源被完全回收后，`recycled_cb` 回调被调用。
        * 在 `recycled_cb` 中，再次调用 `advertising_start()`，使设备重新进入广播状态，等待新的连接。

## 4. nRF52832 资源占用初探 (深度考量)

* **Flash 存储 (Code & Read-Only Data)** (nRF52832 通常具有 512KB Flash):
    * **主要占用**:
        * Zephyr RTOS 内核和配置的驱动程序 (UART, GPIO, Flash 等)。
        * 蓝牙协议栈 (Bluetooth Controller + Host stack)。Controller 部分通常是预编译的二进制库 (SoftDevice 或 Zephyr LL Controller)，Host 部分则包含 L2CAP, SM, ATT, GATT, GAP 以及本例中的 NUS 服务模块。
        * 应用程序逻辑 (`main.c` 中的代码) 和其他引用的库函数 (如 C 标准库 `stdio`, `string`)。
        * 日志系统 (`CONFIG_LOG`) 相关的代码和格式化字符串。
        * 设置系统 (`CONFIG_BT_SETTINGS`) 和 Flash 存储驱动。
    * **影响因素与优化应用**:
        * **Kconfig 选项**: 启用越多的内核特性 (如 `CONFIG_POSIX_API`, `CONFIG_FILE_SYSTEM`)、协议栈功能 (如 `CONFIG_BT_MESH`, `CONFIG_BT_AUDIO` - 虽然 nRF52832 支持有限) 或调试功能 (`CONFIG_DEBUG`, `CONFIG_STACK_SENTINEL`)，Flash 占用会越大。
        * **代码大小**: 应用程序代码的复杂度和体积直接影响 Flash 占用。
        * **日志级别与内容**: 大量的日志字符串 (尤其是在高日志级别下) 会显著增加 Flash 占用。`CONFIG_LOG_MINIMAL` 或 `CONFIG_LOG_PROCESS_THREAD_SLEEP_MS` 等可以优化日志。
        * **链接器优化**: 构建系统通常会进行链接时优化 (LTO) 和死代码消除，但仍需关注。
        * **应用**: 在功能开发过程中，应定期检查编译输出的 Flash 占用大小 (`ninja -C build map` 或查看 `.map` 文件)。如果接近 nRF52832 的上限，就需要考虑：
            * 通过 Kconfig 裁剪不必要的内核或协议栈功能。
            * 优化应用程序代码，减少冗余。
            * 降低默认日志级别，或使用更紧凑的日志格式。
            * 考虑使用代码覆盖率工具移除未使用的函数。

* **RAM (Runtime Data)** (nRF52832 通常具有 64KB RAM):
    * **主要占用**:
        * **线程堆栈 (Thread Stacks)**: 每个内核线程都需要自己的堆栈空间。
            * `main` 线程 (通常有默认堆栈大小，如 `CONFIG_MAIN_STACK_SIZE`)。
            * `ble_write_thread` (`CONFIG_BT_NUS_THREAD_STACK_SIZE`)。
            * 系统工作队列线程 (`CONFIG_SYSTEM_WORKQUEUE_STACK_SIZE=2048`)。
            * 蓝牙协议栈内部线程，如 RX 线程 (`CONFIG_BT_RX_STACK_SIZE`) 和 TX 线程 (如果由应用创建或协议栈内部有)。
        * **系统堆 (System Heap)**: `CONFIG_HEAP_MEM_POOL_SIZE=2048`。用于 `k_malloc` 动态分配 `uart_data_t` 结构体。
        * **蓝牙协议栈**:
            * 连接上下文 (`struct bt_conn`)，每个连接都需要。`CONFIG_BT_MAX_CONN=1` 限制为一个。
            * GATT 属性表 (Services, Characteristics, Descriptors)。
            * L2CAP 缓冲区 (用于组包和拆包)，其大小与 `CONFIG_BT_L2CAP_TX_MTU` 和 `CONFIG_BT_L2CAP_RX_MTU` 相关。
            * 发送和接收 ACL 数据包的缓冲区 (`CONFIG_BT_BUF_ACL_TX_SIZE`/`COUNT`, `CONFIG_BT_BUF_ACL_RX_SIZE`/`COUNT`)。
        * **全局/静态变量**: 如 `current_conn`, `auth_conn`, `ad[]`, `sd[]`, FIFO 队列本身占用的控制块内存。
        * **内核对象**: 信号量、工作队列项等自身也占用少量 RAM。
    * **影响因素与优化应用**:
        * **线程堆栈大小**: 如果线程函数调用层级深或使用大量局部变量，需要足够的堆栈，否则可能发生堆栈溢出。可以使用 `CONFIG_THREAD_STACK_INFO` 和 `CONFIG_THREAD_ANALYZER` 来分析实际堆栈使用情况，并精确调整，避免浪费。
        * **堆使用**: 频繁或大量的动态内存分配是 RAM 碎片化和耗尽的常见原因。对于 `uart_data_t`，如果 `UART_BUF_SIZE` 很大，且 FIFO 中可能累积多个，则堆消耗可观。
        * **蓝牙配置**:
            * 增加 `CONFIG_BT_MAX_CONN` 会显著增加 RAM 需求。
            * 启用更多蓝牙功能 (如 L2CAP CoC, ATT Caching, EATT) 也会增加 RAM。
            * 配置更大的 MTU 或 ACL 缓冲区会直接增加 RAM 占用。
        * **应用**:
            * 对于 RAM 极其敏感的 nRF52832，必须仔细审视和配置上述所有相关 Kconfig 选项。
            * **内存池 (Memory Pool / Memory Slab)**: 对于固定大小且频繁分配/释放的对象 (如 `uart_data_t`)，使用内核内存池 (`k_mem_pool` / `k_mem_slab`) 通常比使用系统堆 (`k_malloc`) 更高效且能避免碎片化。本示例未使用，但可以作为优化方向。
            * 在运行时通过调试器或 Zephyr 的内存报告工具监控 RAM 的高水位线 (High Watermark) 和堆的使用情况，特别是在高负载或长时间运行测试中。

* **功耗 (Power Consumption)**:
    * **主要影响因素**:
        * **无线电活动 (Radio Activity)**: 广播和连接事件是主要的功耗来源。
            * **广播间隔和时长**: 间隔越短、广播时间越长，功耗越高。
            * **连接间隔**: 间隔越短 (如 7.5ms)，无线电收发越频繁，功耗越高。间隔越长 (如 1s)，功耗越低。
            * **从设备延迟 (Slave Latency / Peripheral Latency)**: 允许外设在没有数据要发送时跳过一些连接事件中的实际收听，从而显著降低在长连接间隔下的功耗，同时保持对中心设备数据的响应能力。
        * **TX 发射功率 (Transmit Power)**:
            * **配置**: 通常通过 Kconfig (如 `CONFIG_BT_CTLR_TX_PWR_PLUS_4` 表示 +4dBm, `CONFIG_BT_CTLR_TX_PWR_0` 表示 0dBm, `CONFIG_BT_CTLR_TX_PWR_MINUS_20` 表示 -20dBm) 设置，或者在运行时通过特定 API 修改 (如果支持)。
            * **影响**: 发射功率越高，通信距离可能越远，但功耗也越大。应根据实际应用所需的通信距离选择尽可能低的发射功率。
        * **CPU 活动状态与时钟频率**:
            * CPU 频繁从睡眠模式唤醒来处理任务或中断会增加功耗。应尽量让 CPU 在空闲时进入尽可能深的**低功耗睡眠模式 (Low Power Sleep Mode)**。Zephyr RTOS 会在没有活动线程时自动尝试让 CPU 进入睡眠。
            * nRF52832 支持多种时钟源和频率。在对性能要求不高时，使用较低的 HFCLK (高频时钟) 频率可以降低动态功耗。
        * **外设使用 (Peripheral Usage)**: 活动的 UART、SPI、I2C、ADC、定时器等外设都会消耗电流。在不使用时应尽可能关闭它们或使其进入低功耗状态。
    * **应用 (电池供电设备优化)**:
        * **精心设计 GAP/GATT 参数**: 在满足应用需求的前提下，尽可能使用较长的广播间隔、连接间隔，并启用合适的从设备延迟。
        * **按需广播**: 仅在需要被发现或有数据要广播时才进行广播。
        * **按需连接**: 仅在需要交换数据时才建立和维持连接，数据交换完毕后可考虑主动断开连接。
        * **事件驱动编程**: 尽量避免轮询 (Polling)，使用中断和回调来响应事件，使 CPU 能更多时间处于睡眠状态。
        * **功耗分析工具**: 使用 Nordic 的 Power Profiler Kit II (PPK2) 或类似工具实际测量设备在不同工作状态下的电流消耗，以识别功耗瓶颈并进行针对性优化。
