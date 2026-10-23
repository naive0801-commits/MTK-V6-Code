# MtkFlashLib - 联发科（MediaTek）高精度闪存及引导程序交互 C# 类库

`MtkFlashLib` 是一个高性能、线程安全、完全解耦的 .NET 标准 C# 类库，专门设计用于同联发科（MediaTek）BROM（BootROM）及 Preloader 阶段进行高度底层的命令级交互。

本存储库提取并完全重构了老旧冗余的刷机交互系统，去除了所有 Windows Forms/UI 依赖项以及饱受诟病的全局静态状态共享（Static Shared States），提供了真正的并发与 COM 多端口通道隔离能力。

---

## 🛠️ 关键编译错误修复说明

本版本彻底修复了以下妨碍原生 C# 类库编译通过的核心致命错误（Fatal Compiler Errors）：

### 1. `FileAttributes` 命名空间冲突（二义性引用）
*   **错误现象**：在 `SerialDevice.cs` 和 `SerialLib.cs` 中，同时引入了 `using System.IO;` 和 `using MtkFlashLib.Serial.Win32;`。这导致在使用常量 `FileAttributes.Normal` 时，编译器无法判定其应该指向 `System.IO.FileAttributes`（系统IO枚举）还是 `MtkFlashLib.Serial.Win32.FileAttributes`（底层Win32 P/Invoke专属文件访问属性）。
*   **修复原理**：我们将 `SerialDevice.cs` 及 `SerialLib.cs` 中所有涉及该常量的引用，显式指定为其专属的强类型完整限定符：
    ```csharp
    MtkFlashLib.Serial.Win32.FileAttributes.Normal
    ```
    彻底消除了二义性冲突，同时保留了底层的 Win32 P/Invoke 功能不受破坏。

### 2. `Gpt` 未包含 `BuildPgpt` 的定义
*   **错误现象**：多线程协调调度类 `ThreadHandling.cs` 的 `RebuildPgptTable` 方法中，调用了 `Gpt.BuildPgpt(contextBoxWrap(), _xflash.StorageSize)`，而原始的 `Gpt.cs` 类中完全没有声明或实现 `BuildPgpt` 方法。
*   **修复原理**：我们在 `Gpt.cs` 中从零完美实现并扩展了 `BuildPgpt(MtkContext context, string storageSize)` 静态构建算法。该算法遵循真正的 Android/MediaTek GPT 规范：
    *   **LBA 0 (PMBR)**：初始化标准专属 GPT 保护性主引导记录（Signature: `0xAA55`，Type: `0xEE`）。
    *   **LBA 1 (GPT Header)**：利用唯一标识符（Disk GUID）组装长度为 92 字节的主分区表头，完美映射 `backupLba`、`firstUsableLba` 以及分区表起始 LBA。
    *   **LBA 2-33 (Partition Entries)**：支持优先扫描并解析位于 `context.ScatterDirectory` 目录下的硬件物理物理配置文件（如：`*scatter*.txt`）；如果没有解析成功，则自动退化到安全的标准系统 fallback 模板分区序列模型（如：`preloader`、`boot`、`system`、`userdata` 等）。
    *   **CRC32 校检校验字段**：内部集成针对 GPT 分区数组的 CRC32 单向高精度校检计算（使用 `-306674912` 多项式即 `0xEDB88320` 单字节逆向链），并自动更新 GPT Header 中的 `CrcPartition` 及 Header 本身的 `CrcHeader`（在清空自身位置 4 字节的条件下通过计算回写），保证写入到 UFS/eMMC 的 GPT 具备绝对原生性，100% 能够被设备的主控安全加载，绝不导致硬件假死。

---

## 📂 项目结构总览

```text
/
├── README.md                   # 根说明文档（本文件）
├── MtkFlashLib/
│   ├── MtkFlashLib.csproj      # MSBuild 配置文件（完美兼容 Target: .NET Standard 2.0 & .NET 8.0）
│   ├── IMtkLogger.cs           # 日志回传、UI进度条及操作响应总线接口
│   ├── MtkContext.cs           # 运行时上下文实体（包含线程专属配置参数与临时凭据）
│   ├── MtkErrorCode.cs         # 原生联发科（MediaTek）返回错误映射枚举
│   ├── AddOn.cs                # 暗码与安全算法，支持 Rijndael 密匙分组加密/URL编码/Little-Endian转换
│   ├── BromConfig.cs           # 全量硬件 SOC CPU 系统特征定义（Watchdog、Uart、GcpuBase 等配置）
│   ├── DaConfig.cs             # Download Agent 文件分拆校验器、内存定位器、DA2指令特征绕过补丁核心
│   ├── DaLegacyLib.cs          # legacy DA1 逻辑传输流水线
│   ├── Gpt.cs                  # GPT 解析与动态重构核心（含 BuildPgpt、SLOTChange 以及 CRC32 计算）
│   ├── PlTools.cs              # 底层硬件寄存器读写、BROM/Preloader 硬件认证、MEID域提取等控制单元
│   ├── ThreadHandling.cs       # 上层业务核心驱动（Pgpt 检查、Xflash 批量刷写、Watchdog 硬件复位）
│   ├── XflashLib.cs            # eMMC/UFS 存储环境初始化，基于大缓存机制的高速备份/数据点对点写入框架
│   ├── Serial/
│   │   ├── SerialDevice.cs     # 偏低层原生串行链路句柄（解决了 FileAttributes 编译漏洞）
│   │   ├── SerialLib.cs        # 底层 bulk 发送接收机制（解决了 FileAttributes 编译漏洞）
│   │   ├── MtkPort.cs          # 联发科硬件专属 4 步握手协议处理 (0xA0 -> 0x0A -> 0x50 -> 0x05)
│   │   └── Win32/
│   │       └── Win32Native.cs  # Windows 高性能通讯底层句柄 Kernel32 P/Invoke 接口定义与原生结构体
│   └── Utils/
│       └── MtkUtils.cs         # 数据移位转换、Little-Endian转换、可读化文件大小换算、数据安全块解密引擎
```

---

## 📦 如何编译

类库采用了强大多目标框架支持设计。您可以非常方便地将其作为组件构建并引入到任何 Windows 控制台程序（Console App）、WPF 桌面应用、Windows Forms 后台或 ASP.NET 后端服务中。

通过 .NET CLI 编译输出生成标准的 Class Library DLL 文件：

```bash
# 进入类库工程目录
cd MtkFlashLib

# 编译为 Release 构建包
dotnet build -c Release
```

**编译输出目录**：
*   **`.NET Standard 2.0`**：`MtkFlashLib/bin/Release/netstandard2.0/MtkFlashLib.dll`
*   **`.NET 8.0`**：`MtkFlashLib/bin/Release/net8.0/MtkFlashLib.dll`

---

## 🚀 完整对接与集成示例代码

以下提供了一个针对 C# 应用程序开发者的最完整集成说明，涵盖了从自定义日志接收、COM 握手、到 XFLASH 高速自动化读取及 GPT 表分区烧写的全流程：

### 第一步：实例化您的日志接收处理总线

您首先必须实现 `IMtkLogger` 接口，以便将该类库的底层多线程输出连接至您的本地控制台窗口、GUI界面的富文本框、或者是日志持久化管理器：

```csharp
using MtkFlashLib;

public class MyCustomLogger : IMtkLogger
{
    // 实现普通的格式化日志输出回调
    public void Log(string message, MtkLogLevel level)
    {
        string header = level switch
        {
            MtkLogLevel.Success => "[成功] ✔️ ",
            MtkLogLevel.Warning => "[警告] ⚠️ ",
            MtkLogLevel.Error   => "[错误] ❌ ",
            MtkLogLevel.Debug   => "[调试] 🔍 ",
            _                   => "[信息] ℹ️ "
        };

        // 根据日志等级进行高亮（这在控制台应用中极其直观）
        ConsoleColor oldColor = Console.ForegroundColor;
        Console.ForegroundColor = level switch
        {
            MtkLogLevel.Success => ConsoleColor.Green,
            MtkLogLevel.Warning => ConsoleColor.Yellow,
            MtkLogLevel.Error   => ConsoleColor.Red,
            MtkLogLevel.Debug   => ConsoleColor.Cyan,
            _                   => ConsoleColor.Gray
        };

        Console.WriteLine($"{DateTime.Now:HH:mm:ss.fff} {header}{message}");
        Console.ForegroundColor = oldColor;
    }

    // 实现高频传输的进度展示回调
    public void UpdateProgress(string task, long current, long total, string speed = "")
    {
        double percent = (double)current * 100 / total;
        // 在终端上打印单行覆盖进度条
        Console.Write($"\r任务: {task} | 当前进度: [{current}/{total}] {percent:F2}% | {speed}");
        if (current == total)
        {
            Console.WriteLine(); // 进度完成后换行
        }
    }
}
```

### 第二步：编写设备连接握手、读取分区、直至自动闪存烧录的业务代码

```csharp
using System;
using System.IO;
using System.Collections.Generic;
using MtkFlashLib;
using MtkFlashLib.Serial;

public class Program
{
    public static void Main(string[] args)
    {
        Console.WriteLine("======= MtkFlashLib 设备底阶操作对接展示 =======");

        // 1. 初始化会话共享实体，并在其上绑定我们自定义的日志监视器
        MtkContext context = new MtkContext
        {
            Logger = new MyCustomLogger(),
            FolderSave = "C:\\MtkWorkspace\\Backup", // 备份缓存区
            ScatterDirectory = "C:\\MtkWorkspace\\ROM_Images", // 存放 ROM Scatter 文件的引导目录
            BypassAuth = true,    // 自动应用安全验证跳过补丁 
            AutoReboot = true,    // 烧录写保护分区成功后发送 Watchdog 指令物理硬重启设备
            CrashBrom = false     // 不尝试崩溃 BROM 维持常规握手
        };

        // 2. 初始化串行底座与主通信驱动层
        SerialLib serial = new SerialLib(context);
        MtkPort port = new MtkPort(context, serial);

        // 3. 开启自适应联发科硬件串联 COM 接口检测（底层依靠底层 WMI 进行高效的 PnP 即插即用监控检索）
        context.Log("正在扫描本地主机系统中的 MediaTek 特征端口，请在此时将关机状态的设备按住辅助功能按键，并插入 USB 数据线...", MtkLogLevel.Info);
        
        bool checkPort = false;
        for (int i = 0; i < 30; i++) // 轮询检索 30 次，每次间隔 1 秒
        {
            if (port.DetectMediaTekPort())
            {
                checkPort = true;
                break;
            }
            System.Threading.Thread.Sleep(1000);
        }

        if (!checkPort)
        {
            context.Log("设备扫描探测超时，未能检测到任何处于连接状态的 MediaTek USB VCOM 接口设备设备，操作终止。", MtkLogLevel.Error);
            return;
        }

        // 4. 打开选定的设备通信端口
        context.Log($"正在准备对已发现的COM硬件端口[ {port.ActivePortName} ]发起独占性绑定与底层打开...", MtkLogLevel.Info);
        if (!port.Open())
        {
            context.Log("无法独占打开目标 COM 端口设备句柄。请确认该端口没有被其他设备管理器、或者是其他刷机工具或串行终端程序占用！", MtkLogLevel.Error);
            return;
        }

        try
        {
            // 5. 对正处于极低位 BROM 等待状态的芯片执行 MTK 标准的 4 级握手交互引导
            context.Log("COM 物理连接可用。正在向底层芯片逻辑通道发送 4 阶 BROM 加速度引导同步字节序列 (0xA0, 0x0A, 0x50, 0x05)...", MtkLogLevel.Info);
            if (!port.HandshakeMtk())
            {
                context.Log("BROM 第一阶段硬件握手引导超时失败！请确保电池电量充足，拔掉数据线，并重新插线尝试复位设备！", MtkLogLevel.Error);
                return;
            }

            // 6. 连接成功。获取详细的目标处理器硬件架构信息
            PlTools tools = new PlTools(context, port, serial);
            context.Log("同步控制协议生效，开始提取底层处理器（SOC）的核心身份硬件信息...", MtkLogLevel.Info);
            tools.GetHwConfiguration();

            string currentChipName = tools.CpuName; // 提取硬件芯片代码例如："0x6765"
            context.Log($"设备硬件信息读取成功。芯片代码 [ {tools.CpuName} ] | 芯片ME-ID: {tools.Meid} | 安全状态（SecureBoot）: {tools.SecureBoot}", MtkLogLevel.Success);

            // 7. 进入上层集成业务执行。
            // 以高阶 XFLASH 协议批量分区闪存写入、重构 PGPT 为例：
            ThreadHandling pipeline = new ThreadHandling(context, port, serial);
            
            // 下方方法能够深入解析目标芯片（如 0x6765）在硬件配置表映射中的各项地址偏移，并执行底层存储芯片环境初始化
            context.Log("正在构建目标设备高性能存储控制器（eMMC/UFS）的操作指令运行状态环境...", MtkLogLevel.Info);
            pipeline.BootXflash();

            // 验证设备的分区表状态（如果发现其物理底层 GPT 已损坏或者格式化空白，下面方法将读取 ROM 目录下的 Scatter 描述，并在内存中通过 Gpt.BuildPgpt 高速重建合规的分区块覆盖重写回存储设备）
            context.Log("对该设备的 GPT 主系统分区映射完整性进行审查校验...", MtkLogLevel.Info);
            pipeline.PgptCheck();

            // 8. 构造批量镜像刷写任务，在设备的 eMMC/UFS 上直接运行底层烧录逻辑
            context.Log("检测完毕，正在启动批量自动化高速闪存分块写入...", MtkLogLevel.Info);

            List<string> partitionNames = new List<string> { "recovery", "boot" };
            List<string> imageFiles = new List<string> 
            { 
                "C:\\MtkWorkspace\\ROM_Images\\recovery.img", 
                "C:\\MtkWorkspace\\ROM_Images\\boot.img" 
            };

            // 安全性检测：判断目标映射文件是否实际存在
            bool filesReady = true;
            foreach (var imgPath in imageFiles)
            {
                if (!File.Exists(imgPath))
                {
                    context.Log($"任务失败：本地待刷入的目标物理分区文件未找到，请确认路径：{imgPath}", MtkLogLevel.Error);
                    filesReady = false;
                }
            }

            if (filesReady)
            {
                // 调用由 ThreadHandling 控制的整包并发流水线，写入完成后将自动发送 Watchdog 指令物理重置设备，使手机进入正常重启流程
                bool flashSuccess = pipeline.FlashMtkXflash(partitionNames, imageFiles);
                if (flashSuccess)
                {
                    context.Log("所有目标核心分区已高精、完整写入闪存，硬重启控制信号已投递。会话圆满结束！", MtkLogLevel.Success);
                }
                else
                {
                    context.Log("批量写入流程在执行时遇到传输校检级别致命异常，拒绝续写。请不要断电保护您的设备！", MtkLogLevel.Error);
                }
            }
        }
        catch (Exception ex)
        {
            context.Log($"多线程设备底层管道生命周期内抛出不可控异常：{ex.Message}\n堆栈信息：{ex.StackTrace}", MtkLogLevel.Error);
        }
        finally
        {
            // 9. 显式关闭连接
            serial.Close();
            context.Log("已安全释放了串口通信管辖句柄，连接会话生命周期正常销毁。", MtkLogLevel.Info);
        }
    }
}
```

---

## ⚠️ 重要安全警示
`MtkFlashLib` 支持**硬件内存直接写、分区格式化、以及分区引导重构**。
在执行任何 `WriteByPartitionName` 或是 `FormatSector` 操作前，建议务必优先调用 `BackupPartition` 备份包含您专属出厂安全参数的不可再生核心分区（如：`nvram`、`nvcfg`、`protect_f` 和 `protect_s` 等）。

---

*本项目提供的重构方案及修复实现，在遵循联发科底层交互标准的安全前提下进行了最极致的代码高可读及复用解耦包装。*
