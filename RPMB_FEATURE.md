# MTK RPMB 读写擦功能使用指南

## 重要说明

⚠️ **RPMB 不是普通分区！**

**RPMB（Replay Protected Memory Block）** 是 eMMC 存储的特殊安全区域：
- ✅ 独立的安全存储区域，与普通分区完全分离
- ✅ 具有防重放攻击保护机制
- ✅ 通常用于存储安全启动密钥、DRM 密钥等敏感数据
- ✅ 每块固定 256 字节
- ✅ 需要专门的 MMC 命令进行操作

**与普通分区的区别：**
- 普通分区：通过 GPT 管理，使用标准读写命令
- RPMB：独立管理，使用专用 RPMB 命令（0x0D/0x0E）

## 功能概述

已实现完整的 RPMB 操作功能：
- ✅ **读取 RPMB** - 读取指定地址和块数的 RPMB 数据
- ✅ **写入 RPMB** - 写入数据到 RPMB 区域
- ✅ **擦除 RPMB** - 清空 RPMB 数据（填充 0xFF 或 0x00）
- ✅ **获取 RPMB 容量** - 查询 RPMB 区域大小
- ✅ **完整备份/恢复** - 备份和恢复整个 RPMB 区域

## 已实现的功能

### 1. 读取 RPMB 数据

#### `ReadRpmb(int address, int blockCount, string savePath = "")`
从 RPMB 区域读取数据。

**参数:**
- `address` - RPMB 块地址（从 0 开始）
- `blockCount` - 要读取的块数量（每块 256 字节）
- `savePath` - 保存路径（可选，留空则返回数据）

**返回值:**
- `byte[]` - 读取的 RPMB 数据

**使用示例:**
```csharp
var xflash = new XflashLib(context, port, serial);

// 读取 RPMB 前 10 块数据
byte[] rpmbData = xflash.ReadRpmb(0, 10);
Console.WriteLine($"已读取 {rpmbData.Length} 字节");

// 读取并保存到文件
xflash.ReadRpmb(0, 10, "rpmb_dump.bin");
```

---

### 2. 写入 RPMB 数据

#### `WriteRpmb(int address, byte[] data, byte[] authKey = null)`
将数据写入 RPMB 区域。

**参数:**
- `address` - RPMB 块地址（从 0 开始）
- `data` - 要写入的数据（必须是 256 字节的倍数）
- `authKey` - RPMB 认证密钥（32 字节，可选）

**返回值:**
- `bool` - 是否成功写入

**重要提示:**
- 数据长度必须是 256 字节的倍数
- RPMB 写入可能需要认证密钥（取决于设备配置）

**使用示例:**
```csharp
// 准备数据（必须是 256 字节的倍数）
byte[] data = new byte[512]; // 2 块
// ... 填充数据 ...

// 写入 RPMB
if (xflash.WriteRpmb(0, data))
{
    Console.WriteLine("✓ RPMB 写入成功");
}
```

---

### 3. 从文件写入 RPMB

#### `WriteRpmbFromFile(int address, string filePath, byte[] authKey = null)`
从文件加载数据并写入 RPMB。

**参数:**
- `address` - RPMB 块地址
- `filePath` - 源文件路径
- `authKey` - RPMB 认证密钥（可选）

**使用示例:**
```csharp
// 从文件写入 RPMB
if (xflash.WriteRpmbFromFile(0, "rpmb_data.bin"))
{
    Console.WriteLine("✓ 已从文件写入 RPMB");
}
```

---

### 4. 擦除 RPMB 数据

#### `EraseRpmb(int address, int blockCount, byte fillByte = 0xFF)`
擦除 RPMB 数据（填充指定字节）。

**参数:**
- `address` - RPMB 块地址
- `blockCount` - 要擦除的块数量
- `fillByte` - 填充字节（默认 0xFF）

**返回值:**
- `bool` - 是否成功擦除

**使用示例:**
```csharp
// 擦除前 10 块（填充 0xFF）
if (xflash.EraseRpmb(0, 10))
{
    Console.WriteLine("✓ RPMB 擦除成功");
}

// 擦除并填充 0x00
xflash.EraseRpmb(0, 10, fillByte: 0x00);
```

---

### 5. 获取 RPMB 容量

#### `GetRpmbSize()`
获取 RPMB 区域的容量（块数）。

**返回值:**
- `int` - RPMB 大小（块数，每块 256 字节）

**使用示例:**
```csharp
// 先获取存储信息
xflash.GetEmmcInfoBrom();

// 获取 RPMB 容量
int rpmbBlocks = xflash.GetRpmbSize();
int rpmbBytes = rpmbBlocks * 256;

Console.WriteLine($"RPMB 容量: {rpmbBlocks} 块 ({rpmbBytes} 字节)");
```

---

### 6. 备份整个 RPMB 区域

#### `BackupFullRpmb(string savePath)`
备份完整的 RPMB 区域到文件。

**参数:**
- `savePath` - 备份文件保存路径

**返回值:**
- `bool` - 是否成功备份

**使用示例:**
```csharp
// 获取存储信息（包含 RPMB 容量）
xflash.GetEmmcInfoBrom();

// 备份完整 RPMB
if (xflash.BackupFullRpmb("rpmb_backup.bin"))
{
    Console.WriteLine("✓ RPMB 完整备份已保存");
}
```

---

### 7. 恢复 RPMB 区域

#### `RestoreFullRpmb(string backupPath)`
从备份文件恢复 RPMB 区域。

**参数:**
- `backupPath` - 备份文件路径

**返回值:**
- `bool` - 是否成功恢复

**使用示例:**
```csharp
// 从备份恢复 RPMB
if (xflash.RestoreFullRpmb("rpmb_backup.bin"))
{
    Console.WriteLine("✓ RPMB 恢复成功");
}
```

---

## 完整工作流程示例

### 场景 1: RPMB 完整备份与恢复

```csharp
using MtkFlashLib;
using MtkFlashLib.Serial;

// 1. 初始化连接
var context = new MtkContext();
var serial = new SerialLib(context);
var port = new MtkPort(context, serial);
var xflash = new XflashLib(context, port, serial);

// 2. 连接并握手
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 3. 获取存储信息（包含 RPMB 容量）
xflash.GetEmmcInfoBrom();

// 4. 备份 RPMB
if (xflash.BackupFullRpmb("rpmb_backup.bin"))
{
    Console.WriteLine("✓ RPMB 备份成功");
    
    // 5. 恢复 RPMB（如果需要）
    if (xflash.RestoreFullRpmb("rpmb_backup.bin"))
    {
        Console.WriteLine("✓ RPMB 恢复成功");
    }
}
```

---

### 场景 2: 读取和修改 RPMB 特定块

```csharp
var xflash = new XflashLib(context, port, serial);

// 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 读取 RPMB 前 5 块
byte[] rpmbData = xflash.ReadRpmb(0, 5);

if (rpmbData.Length > 0)
{
    Console.WriteLine($"已读取 {rpmbData.Length} 字节");
    
    // 修改数据
    rpmbData[0] = 0xAA;
    rpmbData[1] = 0xBB;
    
    // 写回 RPMB
    if (xflash.WriteRpmb(0, rpmbData))
    {
        Console.WriteLine("✓ RPMB 数据已更新");
    }
}
```

---

### 场景 3: 清空 RPMB 区域

```csharp
var xflash = new XflashLib(context, port, serial);

// 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();
xflash.GetEmmcInfoBrom();

// 获取 RPMB 容量
int rpmbBlocks = xflash.GetRpmbSize();
Console.WriteLine($"RPMB 容量: {rpmbBlocks} 块");

// 清空整个 RPMB（填充 0xFF）
if (xflash.EraseRpmb(0, rpmbBlocks))
{
    Console.WriteLine("✓ RPMB 已完全清空");
}
```

---

### 场景 4: 部分 RPMB 操作

```csharp
var xflash = new XflashLib(context, port, serial);

// 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 读取 RPMB 块 10-20（共 10 块）
byte[] partialData = xflash.ReadRpmb(10, 10, "rpmb_partial.bin");

// 擦除 RPMB 块 50-60
xflash.EraseRpmb(50, 10);

// 写入自定义数据到块 100
byte[] customData = new byte[256]; // 1 块
// ... 填充数据 ...
xflash.WriteRpmb(100, customData);
```

---

## 技术细节

### RPMB 数据结构

- **块大小**: 256 字节（固定）
- **地址单位**: 块地址（不是字节地址）
- **数据对齐**: 所有读写操作的数据长度必须是 256 字节的倍数

### 协议命令序列

#### 读取 RPMB
```
1. 发送: 09 00 01 00 (初始化)
2. 发送: 0D 00 0C 00 (RPMB 读取命令)
3. 发送: EF EE EE FE 01 00 00 00 0C 00 00 00 (参数头)
4. 发送: [地址(4字节)] [块数(4字节)] [保留(4字节)]
5. 接收: 16 字节状态
6. 接收: 12 字节头 + RPMB 数据
```

#### 写入 RPMB
```
1. 发送: 09 00 01 00 (初始化)
2. 发送: 0E 00 0C 00 (RPMB 写入命令)
3. 发送: EF EE EE FE 01 00 00 00 0C 00 00 00 (参数头)
4. 发送: [地址(4字节)] [块数(4字节)] [保留(4字节)]
5. 接收: 16 字节确认
6. 发送: EF EE EE FE 01 00 00 00 [长度(4字节)]
7. 发送: RPMB 数据
8. 接收: 16 字节写入确认
```

### RPMB 容量说明

| 设备类型 | 典型 RPMB 容量 |
|---------|---------------|
| 4GB eMMC | 128 KB (512 块) |
| 8GB eMMC | 256 KB (1024 块) |
| 16GB eMMC | 512 KB (2048 块) |
| 32GB+ eMMC | 4 MB (16384 块) |

**注意**: 实际容量取决于 eMMC 芯片制造商的配置。

---

## 支持的芯片

### 通用支持

RPMB 功能适用于所有配备 eMMC 存储的 MTK 芯片：

#### 高端芯片（需要先完成 Sig 认证）
- **Dimensity 8100 ~ 9400** 系列
- 必须先执行 Sig 认证，然后才能访问 RPMB

#### 中低端芯片（需要漏洞绕过）
- **Dimensity 1200 及以下**
- Helio G/P/A 系列
- MT6xxx 系列
- 需要先执行漏洞绕过，然后才能访问 RPMB

### 使用流程

```csharp
var plTools = new PlTools(context, port, serial);
var xflash = new XflashLib(context, port, serial);

// 1. 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();
plTools.GetHwConfiguration();

// 2. 根据芯片选择认证方式
if (IsHighEndChip(plTools.CpuName))
{
    // 高端芯片：Sig 认证
    plTools.GetBlobKey(out string blob);
    // ... 计算并发送 Sig
    plTools.SendSigAuthFromFile("sig.bin");
}
else
{
    // 中低端芯片：漏洞绕过
    ChipConfig cfg = BromConfig.HwConfig["0x" + plTools.HwCode];
    plTools.Exploit(cfg, plTools.PreparePayload(cfg));
}

// 3. 认证/绕过成功后，即可操作 RPMB
xflash.GetEmmcInfoBrom();
xflash.BackupFullRpmb("rpmb_backup.bin");
```

**重要**: 
- 所有 RPMB 操作前必须先完成设备授权（Sig 认证或漏洞绕过）
- RPMB 是 eMMC 的特殊区域，不是 UFS 存储的一部分
- UFS 设备使用不同的安全区域（RPMB 功能可能不适用）

---

## 与其他操作的集成

### Sig 认证后进行 RPMB 操作

RPMB 操作通常需要先完成 Sig 认证：

```csharp
var plTools = new PlTools(context, port, serial);
var xflash = new XflashLib(context, port, serial);

// 1. 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 2. 获取硬件信息
plTools.GetHwConfiguration();

// 3. 执行 Sig 认证
if (plTools.SendSigAuthFromFile("auth_sig.bin"))
{
    Console.WriteLine("✓ Sig 认证成功");
    
    // 4. 现在可以进行 RPMB 操作
    xflash.GetEmmcInfoBrom();
    
    // 备份 RPMB
    xflash.BackupFullRpmb("rpmb_backup.bin");
    
    // 读取 RPMB
    byte[] rpmb = xflash.ReadRpmb(0, 10);
    
    // 写入 RPMB
    xflash.WriteRpmb(0, rpmb);
}
```

---

## 故障排除

### 问题 1: RPMB 读取失败
**错误信息**: "RPMB 读取请求失败"

**可能原因:**
- 设备未完成 Sig 认证
- RPMB 地址超出范围
- 通信错误

**解决方法:**
```csharp
// 1. 确认已完成 Sig 认证
plTools.IsDeviceAuthenticatedAndReady();

// 2. 检查 RPMB 容量
xflash.GetEmmcInfoBrom();
int rpmbSize = xflash.GetRpmbSize();
Console.WriteLine($"RPMB 容量: {rpmbSize} 块");

// 3. 确保地址在范围内
if (address + blockCount <= rpmbSize)
{
    xflash.ReadRpmb(address, blockCount);
}
```

---

### 问题 2: RPMB 写入失败
**错误信息**: "RPMB 数据长度必须是 256 字节的倍数"

**解决方法:**
```csharp
// 确保数据对齐到 256 字节
byte[] data = File.ReadAllBytes("data.bin");

if (data.Length % 256 != 0)
{
    // 填充到 256 字节的倍数
    int paddedLength = ((data.Length / 256) + 1) * 256;
    Array.Resize(ref data, paddedLength);
}

xflash.WriteRpmb(0, data);
```

---

### 问题 3: RPMB 容量为 0
**错误信息**: "RPMB 容量未知，请先调用 GetEmmcInfoBrom()"

**解决方法:**
```csharp
// 必须先获取存储信息
xflash.GetEmmcInfoBrom();

// 然后才能获取 RPMB 容量
int rpmbSize = xflash.GetRpmbSize();

if (rpmbSize > 0)
{
    Console.WriteLine($"RPMB 容量: {rpmbSize} 块");
}
else
{
    Console.WriteLine("设备可能不支持 RPMB 或 RPMB 未配置");
}
```

---

## 安全注意事项

⚠️ **重要警告:**

1. **RPMB 存储敏感数据** - 包含安全启动密钥、DRM 密钥等
2. **修改 RPMB 可能导致设备无法启动** - 请务必先备份
3. **RPMB 写入不可逆** - 某些 RPMB 区域写入后无法修改
4. **需要认证** - RPMB 写入通常需要认证密钥
5. **仅用于合法目的** - 修改 RPMB 仅用于设备维修与开发

### 操作前必须备份

```csharp
// 在修改 RPMB 前必须先备份
Console.WriteLine("正在备份 RPMB...");
if (xflash.BackupFullRpmb("rpmb_backup_" + DateTime.Now.ToString("yyyyMMdd_HHmmss") + ".bin"))
{
    Console.WriteLine("✓ 备份完成，现在可以安全操作");
    // 执行 RPMB 操作
}
else
{
    Console.WriteLine("备份失败，操作中止！");
    return;
}
```

---

## API 参考

### XflashLib 类 - RPMB 方法

| 方法 | 说明 | 返回值 |
|------|------|--------|
| `ReadRpmb(int addr, int count, string path)` | 读取 RPMB 数据 | byte[] |
| `WriteRpmb(int addr, byte[] data, byte[] key)` | 写入 RPMB 数据 | bool |
| `WriteRpmbFromFile(int addr, string path, byte[] key)` | 从文件写入 RPMB | bool |
| `EraseRpmb(int addr, int count, byte fill)` | 擦除 RPMB | bool |
| `GetRpmbSize()` | 获取 RPMB 容量 | int |
| `BackupFullRpmb(string path)` | 备份完整 RPMB | bool |
| `RestoreFullRpmb(string path)` | 恢复 RPMB | bool |

### XflashLib 属性

| 属性 | 类型 | 说明 |
|------|------|------|
| `EmmcRpmb` | string | RPMB 大小（十六进制字符串）|
| `RpmbSizeBlocks` | int | RPMB 大小（块数）|

---

## 更新日志

### v1.3 - 2026-06-20
- ✅ 新增 `ReadRpmb` - RPMB 数据读取
- ✅ 新增 `WriteRpmb` - RPMB 数据写入
- ✅ 新增 `WriteRpmbFromFile` - 从文件写入 RPMB
- ✅ 新增 `EraseRpmb` - RPMB 数据擦除
- ✅ 新增 `GetRpmbSize` - RPMB 容量查询
- ✅ 新增 `BackupFullRpmb` - 完整 RPMB 备份
- ✅ 新增 `RestoreFullRpmb` - RPMB 恢复
- ✅ 完整的中文文档与注释
- ✅ 明确说明：RPMB 是独立的安全区域，不是普通分区

---

*最后更新: 2026-06-20*  
*版本: 1.3*
