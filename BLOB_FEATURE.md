# MTK Blob 输出功能使用指南

## 功能概述

MTK Blob 输出功能用于从 MediaTek 芯片读取认证所需的 Blob 配置密钥。这个密钥通常用于：
- 设备解锁与认证
- FRP（出厂重置保护）绕过
- 安全启动验证
- 高级刷机场景

## 已实现的功能

### 1. 基础 Blob 密钥获取

#### `GetBlobKey(out string blobKey)`
从设备读取原始 Blob 密钥（十六进制格式）。

**返回值:**
- `true` - 成功获取
- `false` - 获取失败
- `blobKey` - 输出的十六进制 Blob 密钥（64 字符）

**工作流程:**
1. 发送初始化命令到设备
2. 请求 Blob 数据（68 字节）
3. 提取两个 Token（token2 和 token3）
4. 组合为完整的 Blob 密钥
5. 保存到 MtkContext.ConfigData

**使用示例:**
```csharp
var context = new MtkContext();
var port = new MtkPort(context, serial);
var plTools = new PlTools(context, port, serial);

if (plTools.GetBlobKey(out string blobKey))
{
    Console.WriteLine($"Blob 密钥: {blobKey}");
    // 输出示例: a1b2c3d4e5f6...（64个十六进制字符）
}
```

---

### 2. Base64 格式 Blob 密钥

#### `GetBlobKeyBase64(out string base64BlobKey)`
获取 Base64 编码的 Blob 密钥，便于与外部工具集成。

**返回值:**
- `true` - 成功获取并编码
- `false` - 失败
- `base64BlobKey` - Base64 编码的密钥

**使用示例:**
```csharp
if (plTools.GetBlobKeyBase64(out string base64Key))
{
    Console.WriteLine($"Base64 Blob 密钥: {base64Key}");
    // 可直接用于认证服务器或外部工具
}
```

---

### 3. 保存 Blob 密钥到文件

#### `SaveBlobKeyToFile(string filePath, bool useBase64 = false)`
将 Blob 密钥保存到文件。

**参数:**
- `filePath` - 保存路径（如 "blob_key.txt"）
- `useBase64` - 是否使用 Base64 格式（默认为十六进制）

**使用示例:**
```csharp
// 保存为十六进制格式
plTools.SaveBlobKeyToFile("blob_hex.txt");

// 保存为 Base64 格式
plTools.SaveBlobKeyToFile("blob_base64.txt", useBase64: true);
```

---

### 4. 生成完整认证配置

#### `GenerateAuthConfig(string socId, out string configData)`
生成包含 SOC-ID 和 Blob 密钥的完整认证配置数据。

**格式规范:**
```
认证配置 = "0200000001340220" + SOC_ID + "0310" + BLOB_KEY
```

**参数:**
- `socId` - SOC-ID（留空则使用当前读取的 SOC-ID）
- `configData` - 输出的完整配置数据

**使用示例:**
```csharp
// 使用当前 SOC-ID
if (plTools.GenerateAuthConfig("", out string config))
{
    Console.WriteLine($"认证配置: {config}");
}

// 指定 SOC-ID
if (plTools.GenerateAuthConfig("A1B2C3D4", out string config))
{
    File.WriteAllText("auth_config.txt", config);
}
```

---

### 5. Base64 认证配置

#### `GenerateAuthConfigBase64(string socId, out string base64ConfigData)`
生成 Base64 编码的认证配置数据。

**使用示例:**
```csharp
if (plTools.GenerateAuthConfigBase64("", out string base64Config))
{
    // 可直接传递给认证服务器
    SendToAuthServer(base64Config);
}
```

---

## 完整工作流程示例

### 场景 1: 基础 Blob 提取
```csharp
using MtkFlashLib;
using MtkFlashLib.Serial;

// 1. 初始化
var context = new MtkContext { Logger = new ConsoleLogger() };
var serial = new SerialLib(context);
var port = new MtkPort(context, serial);
var plTools = new PlTools(context, port, serial);

// 2. 连接设备
if (!port.DetectMediaTekPort())
{
    Console.WriteLine("未检测到 MTK 设备");
    return;
}

if (!port.Open())
{
    Console.WriteLine("打开端口失败");
    return;
}

// 3. 握手
if (!port.HandshakeMtk())
{
    Console.WriteLine("握手失败");
    return;
}

// 4. 获取硬件配置（包含 SOC-ID）
plTools.GetHwConfiguration();

// 5. 获取 Blob 密钥
if (plTools.GetBlobKey(out string blobKey))
{
    Console.WriteLine($"✓ Blob 密钥: {blobKey}");
    
    // 6. 保存到文件
    plTools.SaveBlobKeyToFile("blob_output.txt");
}
```

---

### 场景 2: 生成认证配置用于 FRP 绕过
```csharp
// 1. 初始化并连接（同上）
// ...

// 2. 获取 Blob 密钥并生成认证配置
if (plTools.GenerateAuthConfigBase64("", out string authConfig))
{
    // 3. 保存认证配置
    File.WriteAllText("config.bin", authConfig);
    Console.WriteLine("✓ 认证配置已生成");
    
    // 4. 可用于外部认证工具
    // 例如: FrpLoader.exe 或其他认证服务
}
```

---

### 场景 3: 批量设备 Blob 提取
```csharp
var devices = DetectAllMtkDevices();

foreach (var device in devices)
{
    var context = new MtkContext();
    var port = new MtkPort(context, serial);
    var plTools = new PlTools(context, port, serial);
    
    if (port.Open(device.PortNumber))
    {
        if (port.HandshakeMtk())
        {
            plTools.GetHwConfiguration();
            
            if (plTools.GetBlobKey(out string blob))
            {
                string fileName = $"blob_{plTools.SocId}_{plTools.Meid}.txt";
                plTools.SaveBlobKeyToFile(fileName);
                Console.WriteLine($"✓ 已保存: {fileName}");
            }
        }
    }
}
```

---

## 技术细节

### Blob 数据结构

从设备读取的 68 字节数据结构：
```
偏移    长度    说明
0-3     4字节   头部标识
4-19    16字节  Token2（第一个密钥片段）
20-35   16字节  Token3（第二个密钥片段）
36-67   32字节  保留数据
```

### Blob 密钥组合
```
最终 Blob 密钥 = Token3 + Token2
长度: 64 个十六进制字符（32 字节）
```

### 认证配置格式
```
完整配置 = "0200000001340220" + SOC_ID + "0310" + Blob密钥

示例:
0200000001340220A1B2C3D40310abcdef1234567890...
|              ||        ||  ||
|              ||        ||  |+-- Blob 密钥 (64字符)
|              ||        ||  +--- 中间标识 "0310"
|              ||        |+------ SOC-ID (8字符)
|              |+---------- 认证头部
```

---

## 协议命令序列

### 获取 Blob 密钥的完整协议流程:

```
1. 发送: 09 00 01 00 (初始化命令 1)
   接收: 状态确认

2. 发送: 0C 00 02 00 (初始化命令 2)
   接收: 状态确认

3. 发送: EF EE EE FE 01 00 00 00 00 00 00 00 (空数据包)
   接收: 16 字节响应

4. 发送: 09 00 01 00 (请求 Blob 数据 1)
   接收: 状态确认

5. 发送: 13 00 04 00 (请求 Blob 数据 2)
   接收: 状态确认

6. 接收: 12 字节头部 + 68 字节 Blob 数据

7. 发送: 09 00 01 00 (确认命令 1)
   接收: 状态确认

8. 发送: 16 00 04 00 (确认命令 2)
   接收: 16 字节最终确认
```

---

## 故障排除

### 问题 1: 获取 Blob 失败
**可能原因:**
- 设备未正确连接
- 未完成握手
- 设备不支持 Blob 读取（旧芯片）

**解决方法:**
```csharp
// 确保握手成功
if (!port.HandshakeMtk())
{
    Console.WriteLine("握手失败，请重试");
    return;
}

// 检查设备是否支持
plTools.GetHwConfiguration();
Console.WriteLine($"芯片: {plTools.CpuName}");
Console.WriteLine($"安全启动: {plTools.SecureBoot}");
```

---

### 问题 2: Blob 数据长度不足
**错误信息:** "Blob 数据长度不足: X 字节，期望 68 字节"

**解决方法:**
- 检查串口通信是否稳定
- 增加读取超时时间
- 重新连接设备

---

### 问题 3: SOC-ID 未知
**错误信息:** "SOC-ID 未知，请先调用 GetHwConfiguration"

**解决方法:**
```csharp
// 在生成认证配置前先获取硬件配置
plTools.GetHwConfiguration();

// 现在可以生成认证配置
plTools.GenerateAuthConfig("", out string config);
```

---

## 安全注意事项

⚠️ **重要提示:**

1. **Blob 密钥保密** - Blob 密钥是敏感数据，应妥善保管
2. **仅用于合法用途** - 此功能仅用于合法的设备维修与开发
3. **备份原始数据** - 在进行任何操作前备份设备数据
4. **遵守法律法规** - 确保使用符合当地法律法规

---

## 支持的芯片

### 芯片认证策略

此功能支持 **Dimensity 8100 ~ Dimensity 9400（天玑 8100-9400）** 系列高端芯片。

#### 需要 Blob + Sig 认证的芯片（高端）

| 芯片型号 | 代号 | Blob/Sig 支持 |
|---------|------|--------------|
| Dimensity 9400 | 天玑 9400 | ✅ 完全支持 |
| Dimensity 9300 | 天玑 9300 | ✅ 完全支持 |
| Dimensity 9200+ | 天玑 9200+ | ✅ 完全支持 |
| Dimensity 9200 | 天玑 9200 | ✅ 完全支持 |
| Dimensity 9000+ | 天玑 9000+ | ✅ 完全支持 |
| Dimensity 9000 | 天玑 9000 | ✅ 完全支持 |
| Dimensity 8300 | 天玑 8300 | ✅ 完全支持 |
| Dimensity 8200 | 天玑 8200 | ✅ 完全支持 |
| Dimensity 8100 | 天玑 8100 | ✅ 完全支持 |

**操作方式**: 必须先执行 Blob 提取 → 使用认证工具计算 Sig → 发送 Sig 到设备，才能进行读写擦操作。

#### 使用漏洞绕过的芯片（中低端）

**Dimensity 1200 及以下芯片** 不需要 Blob/Sig 认证，直接使用漏洞绕过：

- ✅ Dimensity 1200, 1100, 1000+, 900, 800, 720, 700
- ✅ Helio G 系列: G95, G90, G85, G80, G70
- ✅ Helio P 系列: P95, P90, P70, P65, P60, P35, P30, P23, P22
- ✅ Helio A 系列
- ✅ MT6xxx 系列: MT6893, MT6891, MT6889, MT6877, MT6873, MT6853, MT6833, MT6785, MT6771, MT6768, MT6765, MT6763

**操作方式**: 使用 `PlTools.Exploit()` 方法进行漏洞绕过，无需 Blob/Sig 认证。

### 识别与选择

```csharp
var plTools = new PlTools(context, port, serial);
plTools.GetHwConfiguration();

string chipName = plTools.CpuName;
Console.WriteLine($"检测到芯片: {chipName}");

// 判断是否需要 Blob/Sig
bool needsBlobSig = chipName.Contains("Dimensity") && 
                    (chipName.Contains("8100") || chipName.Contains("8200") || 
                     chipName.Contains("8300") || chipName.Contains("9000") || 
                     chipName.Contains("9200") || chipName.Contains("9300") || 
                     chipName.Contains("9400"));

if (needsBlobSig)
{
    Console.WriteLine("→ 使用 Blob + Sig 认证方式");
    // 执行 Blob 提取流程
    plTools.GetBlobKey(out string blob);
}
else
{
    Console.WriteLine("→ 使用漏洞绕过方式");
    // 执行漏洞绕过
    ChipConfig config = BromConfig.HwConfig["0x" + plTools.HwCode];
    byte[] payload = plTools.PreparePayload(config);
    plTools.Exploit(config, payload);
}
```

---

## API 参考

### PlTools 类新增方法

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `GetBlobKey(out string blobKey)` | 获取十六进制 Blob 密钥 | bool |
| `GetBlobKeyBase64(out string base64BlobKey)` | 获取 Base64 Blob 密钥 | bool |
| `SaveBlobKeyToFile(string filePath, bool useBase64)` | 保存 Blob 密钥到文件 | bool |
| `GenerateAuthConfig(string socId, out string configData)` | 生成认证配置（十六进制）| bool |
| `GenerateAuthConfigBase64(string socId, out string base64ConfigData)` | 生成认证配置（Base64）| bool |

### MtkContext 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `ConfigData` | string | 存储获取的 Blob 配置数据 |

---

## 更新日志

### v1.1 - 2026-06-20
- ✅ 新增 `GetBlobKey` - 基础 Blob 密钥获取
- ✅ 新增 `GetBlobKeyBase64` - Base64 格式支持
- ✅ 新增 `SaveBlobKeyToFile` - 文件保存功能
- ✅ 新增 `GenerateAuthConfig` - 认证配置生成
- ✅ 新增 `GenerateAuthConfigBase64` - Base64 认证配置
- ✅ 完整的中文日志与错误处理
- ✅ 详细的使用文档与示例

---

*最后更新: 2026-06-20*  
*版本: 1.1*
