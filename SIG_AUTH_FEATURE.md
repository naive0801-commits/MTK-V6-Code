# MTK Sig 授权验证功能使用指南

## 重要说明

⚠️ **Blob 和 Sig 用于 MTK 芯片通讯协议的授权验证，与设备 Bootloader 解锁无关！**

- **Blob** - 用于通讯授权验证的密钥
- **Sig** - 基于 Blob 通过认证工具计算的签名数据
- **Sig 认证成功** = 设备通过授权验证，允许进行读写擦操作
- **与设备解锁无关** - 这是 MTK 芯片通讯协议的安全验证机制

## 功能概述

Sig 授权验证功能用于向 MTK 设备发送签名数据，通过后设备允许进行读写擦等操作。

### 完整流程

```
1. 提取 Blob 密钥（从设备）
   ↓
2. 生成认证配置（Blob + SOC-ID）
   ↓
3. 使用外部认证工具计算 Sig（基于 Blob）
   ↓
4. 发送 Sig 到设备进行授权验证
   ↓
5. 验证成功 → 设备允许读写擦操作
```

## 支持的芯片范围

### 需要 Sig 认证的芯片（高端）

此功能专为 **Dimensity 8100 ~ Dimensity 9400（天玑 8100-9400）** 系列设计：

| 芯片系列 | 型号 | Sig 认证 |
|---------|------|---------|
| 天玑 9000 系列 | 9400, 9300, 9200+, 9200, 9000+, 9000 | ✅ 必需 |
| 天玑 8000 系列 | 8300, 8200, 8100 | ✅ 必需 |

**特点**: 这些高端芯片必须完成 Sig 授权验证后，才能进行读写擦等操作。

### 不需要 Sig 认证的芯片（中低端）

**Dimensity 1200 及以下** 芯片使用漏洞绕过，不需要 Sig 认证：
- Dimensity 1200, 1100, 1000+, 900, 800, 720, 700
- Helio G/P/A 系列
- MT6xxx 系列

**推荐方式**: 使用 `PlTools.Exploit()` 漏洞绕过方法。

### 自动识别示例

```csharp
var plTools = new PlTools(context, port, serial);
plTools.GetHwConfiguration();

// 自动识别芯片并选择认证方式
if (plTools.CpuName.Contains("Dimensity"))
{
    int seriesNum = ExtractDimensityNumber(plTools.CpuName);
    
    if (seriesNum >= 8100 && seriesNum <= 9400)
    {
        Console.WriteLine("高端芯片，需要 Sig 认证");
        
        // Blob + Sig 认证流程
        plTools.GetBlobKey(out string blob);
        plTools.GenerateAuthConfig("", out string config);
        // ... 计算 Sig
        plTools.SendSigAuthFromFile("sig.bin");
    }
    else if (seriesNum <= 1200)
    {
        Console.WriteLine("中低端芯片，使用漏洞绕过");
        
        // 漏洞绕过流程
        ChipConfig cfg = BromConfig.HwConfig["0x" + plTools.HwCode];
        plTools.Exploit(cfg, plTools.PreparePayload(cfg));
    }
}
```

---

## 已实现的功能

### 1. 发送 Sig 授权数据（字节数组）

#### `SendSigAuth(byte[] sigData)`
将签名数据发送到设备进行授权验证。

**参数:**
- `sigData` - 签名数据字节数组（最多 256 字节）

**返回值:**
- `true` - 授权验证成功
- `false` - 验证失败

**协议流程:**
```
1. 发送: 09 00 01 00 (初始化命令)
2. 发送: 0B 00 02 00 (准备命令)
3. 发送: EF EE EE FE 01 00 00 00 00 01 00 00 (长度头)
4. 发送: Sig 数据（256 字节，不足则填充）
5. 接收: 16 字节响应
   - 最后一个字节为 00 = 成功
   - 其他值 = 失败
```

**使用示例:**
```csharp
// 从认证工具获取的 Sig 数据
byte[] sigData = GetSigFromAuthTool();

if (plTools.SendSigAuth(sigData))
{
    Console.WriteLine("✓ 授权验证成功，可进行读写擦操作");
    // 现在可以调用读写擦等方法
}
```

---

### 2. 发送 Sig 授权数据（十六进制字符串）

#### `SendSigAuth(string sigHexString)`
将十六进制格式的签名数据发送到设备。

**参数:**
- `sigHexString` - 十六进制格式的签名字符串

**使用示例:**
```csharp
// 十六进制格式的 Sig
string sigHex = "A1B2C3D4E5F6...";

if (plTools.SendSigAuth(sigHex))
{
    Console.WriteLine("授权验证成功");
}
```

---

### 3. 从文件加载并发送 Sig

#### `SendSigAuthFromFile(string sigFilePath, bool isBinary = false)`
从文件加载 Sig 数据并发送到设备。

**参数:**
- `sigFilePath` - Sig 文件路径
- `isBinary` - 是否为二进制文件（false = 十六进制文本）

**使用示例:**
```csharp
// 从十六进制文本文件加载
plTools.SendSigAuthFromFile("auth.bin", isBinary: false);

// 从二进制文件加载
plTools.SendSigAuthFromFile("auth_sig.bin", isBinary: true);
```

---

### 4. 完整的 Blob + Sig 自动化流程

#### `PerformFullBlobSigAuth(string authToolPath, string sigOutputPath)`
自动化执行完整的 Blob 提取和 Sig 认证流程。

**参数:**
- `authToolPath` - 外部认证工具路径（可选）
- `sigOutputPath` - Sig 输出文件路径（默认 "auth.bin"）

**流程:**
1. 提取 Blob 密钥
2. 生成认证配置
3. 调用外部认证工具计算 Sig
4. 发送 Sig 到设备

**使用示例:**
```csharp
// 完整自动化流程
if (plTools.PerformFullBlobSigAuth("AuthTool.exe", "sig_output.bin"))
{
    Console.WriteLine("完整授权验证流程完成");
}

// 仅提取 Blob 和配置（手动处理 Sig）
if (plTools.PerformFullBlobSigAuth())
{
    Console.WriteLine("Blob 和配置已生成，请使用认证工具处理");
    // config.bin 已生成，使用外部工具处理后调用 SendSigAuth
}
```

---

### 5. 验证设备授权状态

#### `IsDeviceAuthenticatedAndReady()`
检查设备是否已通过 Sig 授权验证。

**返回值:**
- `true` - 设备已授权，可读写
- `false` - 设备未授权或通信失败

**使用示例:**
```csharp
if (plTools.IsDeviceAuthenticatedAndReady())
{
    Console.WriteLine("设备已授权，可以开始操作");
    // 执行读写擦等操作
}
else
{
    Console.WriteLine("设备未授权，请先执行 Sig 认证");
}
```

---

## 完整工作流程示例

### 场景 1: 手动 Sig 认证

```csharp
using MtkFlashLib;
using MtkFlashLib.Serial;

// 1. 初始化连接
var context = new MtkContext();
var serial = new SerialLib(context);
var port = new MtkPort(context, serial);
var plTools = new PlTools(context, port, serial);

// 2. 连接并握手
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 3. 获取 Blob 密钥
if (plTools.GetBlobKey(out string blob))
{
    Console.WriteLine($"Blob: {blob}");
    
    // 4. 生成认证配置
    if (plTools.GenerateAuthConfig("", out string config))
    {
        File.WriteAllText("config.bin", config);
        Console.WriteLine("认证配置已保存: config.bin");
        
        // 5. 使用外部工具计算 Sig
        Console.WriteLine("请使用认证工具处理 config.bin 生成 Sig");
        Console.WriteLine("生成后按任意键继续...");
        Console.ReadKey();
        
        // 6. 发送 Sig 到设备
        if (plTools.SendSigAuthFromFile("auth_sig.bin"))
        {
            Console.WriteLine("✓ 授权验证成功！");
            
            // 7. 验证状态
            if (plTools.IsDeviceAuthenticatedAndReady())
            {
                Console.WriteLine("✓ 设备当前处于可读写状态");
                // 现在可以进行读写擦操作
            }
        }
    }
}
```

---

### 场景 2: 自动化 Sig 认证

```csharp
// 全自动流程（如果有认证工具）
var plTools = new PlTools(context, port, serial);

// 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();
plTools.GetHwConfiguration();

// 执行完整认证流程
if (plTools.PerformFullBlobSigAuth("C:\\Tools\\AuthTool.exe", "sig.bin"))
{
    Console.WriteLine("=== 授权验证完成 ===");
    
    // 直接开始读写操作
    var xflash = new XflashLib(context, port, serial);
    xflash.GetEmmcInfo();
    // ... 其他操作
}
```

---

### 场景 3: 批量设备授权

```csharp
// 为多个设备执行 Sig 授权
var devices = DetectAllMtkDevices();

foreach (var device in devices)
{
    var plTools = new PlTools(context, port, serial);
    
    if (port.Open(device.PortNumber))
    {
        if (port.HandshakeMtk())
        {
            // 提取 Blob
            if (plTools.GetBlobKey(out string blob))
            {
                // 保存 Blob 用于后续处理
                string blobFile = $"blob_{device.SerialNumber}.txt";
                File.WriteAllText(blobFile, blob);
                
                // 如果已有对应的 Sig 文件
                string sigFile = $"sig_{device.SerialNumber}.bin";
                if (File.Exists(sigFile))
                {
                    if (plTools.SendSigAuthFromFile(sigFile))
                    {
                        Console.WriteLine($"✓ 设备 {device.SerialNumber} 已授权");
                    }
                }
            }
        }
    }
}
```

---

## 技术细节

### Sig 数据格式

- **长度**: 最多 256 字节
- **填充**: 不足 256 字节时会自动填充 0x00
- **格式**: 由外部认证工具基于 Blob 和 SOC-ID 计算

### 协议命令序列

```
步骤 1: 发送初始化
    发送: 09 00 01 00
    接收: 16 字节状态确认

步骤 2: 发送准备命令
    发送: 0B 00 02 00
    接收: 16 字节状态确认

步骤 3: 发送长度头
    发送: EF EE EE FE 01 00 00 00 00 01 00 00
    说明: 0x0100 = 256 字节

步骤 4: 发送 Sig 数据
    发送: 256 字节 Sig 数据（不足则填充）

步骤 5: 接收验证结果
    接收: 16 字节响应
    验证: response[15] == 0x00 表示成功
```

### 响应码说明

| 最后字节 | 说明 |
|---------|------|
| 0x00 | 授权验证成功 |
| 0x01 | 签名无效 |
| 0x02 | 数据格式错误 |
| 其他 | 未知错误 |

---

## 与其他操作的集成

### Sig 认证后可执行的操作

授权验证成功后，可以执行以下操作：

```csharp
// 1. 读取存储信息
var xflash = new XflashLib(context, port, serial);
xflash.GetEmmcInfo();

// 2. 读取分区
xflash.BackupPartition("system");

// 3. 写入分区
xflash.WriteByPartitionName("boot", bootData);

// 4. 格式化扇区
xflash.FormatSector("userdata", startSector, length);

// 5. 读取 GPT
var gpt = new Gpt(context, port, serial);
gpt.ReadGptTable();
```

---

## 故障排除

### 问题 1: Sig 认证失败
**错误信息**: "Sig 认证失败，响应: XXX"

**可能原因:**
- Sig 数据不正确
- Blob 密钥不匹配
- SOC-ID 错误
- 认证工具版本不匹配

**解决方法:**
```csharp
// 1. 重新提取 Blob
plTools.GetBlobKey(out string blob);
Console.WriteLine($"当前 Blob: {blob}");

// 2. 检查 SOC-ID
plTools.GetHwConfiguration();
Console.WriteLine($"SOC-ID: {plTools.SocId}");

// 3. 重新生成认证配置
plTools.GenerateAuthConfig("", out string config);

// 4. 使用正确的认证工具重新计算 Sig
```

---

### 问题 2: 设备未响应
**错误信息**: "Sig 认证初始化命令失败"

**解决方法:**
- 检查设备连接
- 重新握手
- 确认设备处于正确状态

```csharp
// 重新初始化
port.Close();
port.Open();
port.HandshakeMtk();
```

---

### 问题 3: Sig 文件格式错误
**错误信息**: "Sig 数据为空" 或加载失败

**解决方法:**
```csharp
// 检查文件内容
if (File.Exists("auth.bin"))
{
    string content = File.ReadAllText("auth.bin");
    Console.WriteLine($"Sig 内容: {content}");
    Console.WriteLine($"长度: {content.Length} 字符");
}

// 尝试不同的加载方式
// 十六进制文本
plTools.SendSigAuthFromFile("auth.bin", isBinary: false);

// 二进制文件
plTools.SendSigAuthFromFile("auth.bin", isBinary: true);
```

---

## 安全注意事项

⚠️ **重要提示:**

1. **Blob 和 Sig 保密** - 这些数据用于授权验证，应妥善保管
2. **仅用于授权目的** - 不能用于设备 Bootloader 解锁
3. **合法使用** - 仅用于合法的设备维修与开发
4. **备份数据** - 在进行任何操作前备份重要数据
5. **与解锁无关** - 这是 MTK 芯片通讯协议的授权验证，不涉及设备解锁

---

## API 参考

### PlTools 类 - Sig 授权方法

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `SendSigAuth(byte[] sigData)` | 发送 Sig 数据（字节数组）| bool |
| `SendSigAuth(string sigHexString)` | 发送 Sig 数据（十六进制）| bool |
| `SendSigAuthFromFile(string path, bool isBinary)` | 从文件加载并发送 Sig | bool |
| `PerformFullBlobSigAuth(string toolPath, string outPath)` | 完整自动化流程 | bool |
| `IsDeviceAuthenticatedAndReady()` | 验证设备授权状态 | bool |

---

## 更新日志

### v1.2 - 2026-06-20
- ✅ 新增 `SendSigAuth(byte[])` - 发送 Sig 授权数据
- ✅ 新增 `SendSigAuth(string)` - 十六进制字符串版本
- ✅ 新增 `SendSigAuthFromFile` - 从文件加载并发送
- ✅ 新增 `PerformFullBlobSigAuth` - 完整自动化流程
- ✅ 新增 `IsDeviceAuthenticatedAndReady` - 状态验证
- ✅ 完整的中文文档与注释
- ✅ 明确说明：用于授权验证，与设备解锁无关

---

*最后更新: 2026-06-20*  
*版本: 1.2*
