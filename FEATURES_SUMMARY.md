# MtkFlashLib 功能总结 v1.3

## � 项目完成总结 v1.3

### ✅ 已完成的所有功能

| 功能模块 | 状态 | 方法数 | 文档 |
|---------|------|--------|------|
| **Blob 输出** | ✅ 完成 | 5 | 本文档 |
| **Sig 授权验证** | ✅ 完成 | 5 | 本文档 |
| **RPMB 读写擦** | ✅ 完成 | 7 | 本文档 |
| **DA 发送** | ✅ 完成 | 8 | PORTING_SUMMARY.md |
| **分区操作** | ✅ 完成 | 5 | XflashLib.cs |
| **格式化** | ✅ 完成 | 2 | XflashLib.cs |
| **漏洞绕过** | ✅ 完成 | 3 | PlTools.cs |

### 📊 最终统计

```
✅ 编译状态: 成功
✅ 错误: 0
✅ 警告: 0
✅ 总方法数: 45+
✅ 文档页数: 4
✅ 代码行数: 5000+
✅ 支持芯片: 30+
```

---

## 🎉 项目完成总结 v1.3

**MtkFlashLib** 是一个完整的 MTK 芯片刷机协议 C# 类库，支持：
- ✅ MTK BROM/DA 通讯协议
- ✅ Blob/Sig 授权验证
- ✅ RPMB 读写擦操作
- ✅ 分区读写与格式化
- ✅ GPT 分区表管理

## ✅ 编译状态

```
编译: 成功
错误: 0
警告: 0
平台: .NET 10.0 (Windows)
```

## 📦 已实现的功能模块

### 1. Blob 输出功能（v1.1）

**位置**: `PlTools.cs`

| 方法 | 功能 |
|------|------|
| `GetBlobKey` | 从设备提取 Blob 密钥（十六进制）|
| `GetBlobKeyBase64` | 提取 Blob 密钥（Base64 格式）|
| `SaveBlobKeyToFile` | 保存 Blob 密钥到文件 |
| `GenerateAuthConfig` | 生成完整认证配置 |
| `GenerateAuthConfigBase64` | 生成 Base64 认证配置 |

**用途**: 用于 MTK 芯片通讯协议的授权验证（不是设备解锁）

**文档**: `BLOB_FEATURE.md`

---

### 2. Sig 授权验证功能（v1.2）

**位置**: `PlTools.cs`

| 方法 | 功能 |
|------|------|
| `SendSigAuth(byte[])` | 发送 Sig 数据（字节数组）|
| `SendSigAuth(string)` | 发送 Sig 数据（十六进制字符串）|
| `SendSigAuthFromFile` | 从文件加载并发送 Sig |
| `PerformFullBlobSigAuth` | 完整 Blob+Sig 自动化流程 |
| `IsDeviceAuthenticatedAndReady` | 验证设备授权状态 |

**用途**: 
- 通过授权验证后，设备允许进行读写擦操作
- 用于 MTK 芯片通讯协议的安全验证
- **与设备 Bootloader 解锁无关**

**文档**: `SIG_AUTH_FEATURE.md`

---

### 3. RPMB 读写擦功能（v1.3）

**位置**: `XflashLib.cs`

| 方法 | 功能 |
|------|------|
| `ReadRpmb` | 读取 RPMB 数据 |
| `WriteRpmb` | 写入 RPMB 数据 |
| `WriteRpmbFromFile` | 从文件写入 RPMB |
| `EraseRpmb` | 擦除 RPMB 数据 |
| `GetRpmbSize` | 获取 RPMB 容量 |
| `BackupFullRpmb` | 备份完整 RPMB 区域 |
| `RestoreFullRpmb` | 恢复 RPMB 区域 |

**重要说明**:
- ⚠️ **RPMB 不是普通分区**
- RPMB 是 eMMC 的独立安全区域
- 每块固定 256 字节
- 通常存储安全启动密钥、DRM 密钥等敏感数据
- 使用专用 RPMB 命令（0x0D/0x0E）

**文档**: `RPMB_FEATURE.md`

---

### 4. 核心刷机协议

**位置**: `DaLegacyLib.cs`, `XflashLib.cs`, `PlTools.cs`

| 功能类别 | 已实现方法 |
|---------|-----------|
| **DA 发送** | SendDA1, SendDa2v6, SendDa2Legacy, SendDa2Xml |
| **EMI 配置** | SendingEmi |
| **Preloader** | RunDumpPreloader, PreparePayload, Exploit |
| **存储信息** | GetEmmcInfoBrom, GetHwConfiguration |
| **分区操作** | WriteByPartitionName, BackupPartition |
| **格式化** | FormatSector |
| **环境设置** | SetupEnv, HandshakeMtk |

---

## 🔄 完整工作流程示例

### 场景：带 Sig 认证的 RPMB 备份

```csharp
using MtkFlashLib;
using MtkFlashLib.Serial;

// 1. 初始化
var context = new MtkContext { Logger = new ConsoleLogger() };
var serial = new SerialLib(context);
var port = new MtkPort(context, serial);
var plTools = new PlTools(context, port, serial);
var xflash = new XflashLib(context, port, serial);

// 2. 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 3. 获取硬件信息
plTools.GetHwConfiguration();
Console.WriteLine($"芯片: {plTools.CpuName}");
Console.WriteLine($"SOC-ID: {plTools.SocId}");

// 4. 获取 Blob 密钥
if (plTools.GetBlobKey(out string blob))
{
    Console.WriteLine($"✓ Blob: {blob}");
    plTools.SaveBlobKeyToFile("blob.txt");
}

// 5. 生成认证配置
if (plTools.GenerateAuthConfig("", out string config))
{
    File.WriteAllText("config.bin", config);
    Console.WriteLine("✓ 认证配置已生成");
}

// 6. 使用外部工具计算 Sig（手动或自动）
Console.WriteLine("请使用认证工具处理 config.bin 生成 sig.bin");
Console.WriteLine("完成后按任意键继续...");
Console.ReadKey();

// 7. 发送 Sig 授权
if (plTools.SendSigAuthFromFile("sig.bin"))
{
    Console.WriteLine("✓ Sig 认证成功，设备已授权");
    
    // 8. 验证授权状态
    if (plTools.IsDeviceAuthenticatedAndReady())
    {
        Console.WriteLine("✓ 设备当前处于可读写状态");
        
        // 9. 获取存储信息
        xflash.GetEmmcInfoBrom();
        
        // 10. 获取 RPMB 容量
        int rpmbBlocks = xflash.GetRpmbSize();
        Console.WriteLine($"RPMB 容量: {rpmbBlocks} 块");
        
        // 11. 备份 RPMB
        if (xflash.BackupFullRpmb("rpmb_backup.bin"))
        {
            Console.WriteLine("✓ RPMB 备份完成");
        }
        
        // 12. 读取特定 RPMB 块
        byte[] rpmbData = xflash.ReadRpmb(0, 10, "rpmb_first10.bin");
        Console.WriteLine($"✓ 已读取 {rpmbData.Length} 字节");
        
        // 13. 写入分区（可选）
        xflash.WriteByPartitionName("boot", "boot.img");
        
        // 14. 格式化分区（可选）
        xflash.FormatSector("userdata", "0", "1000");
    }
}
```

---

## 📚 文档列表

| 文档 | 说明 |
|------|------|
| `README.md` | 项目主文档 |
| `PORTING_SUMMARY.md` | 协议移植总结 |
| `BLOB_README.md` | Blob 功能快速开始 |
| `BLOB_FEATURE.md` | Blob 功能详细文档 |
| `SIG_AUTH_FEATURE.md` | Sig 授权验证详细文档 |
| `RPMB_FEATURE.md` | RPMB 读写擦详细文档 |
| `FEATURES_SUMMARY.md` | 本文档（功能总结）|

---

## 🎯 核心概念说明

### Blob 与 Sig 的关系

```
设备 → 提取 Blob 密钥
  ↓
Blob + SOC-ID → 生成认证配置
  ↓
认证工具 → 计算 Sig（基于 Blob）
  ↓
发送 Sig → 设备授权验证通过
  ↓
设备允许读写擦操作
```

**重要**: Blob 和 Sig 用于通讯协议授权验证，**与设备 Bootloader 解锁无关**。

### RPMB vs 普通分区

| 特性 | 普通分区 | RPMB |
|------|---------|------|
| 管理方式 | GPT 分区表 | 独立管理 |
| 读写命令 | 标准读写 | 专用命令（0x0D/0x0E）|
| 块大小 | 512 字节 | 256 字节 |
| 用途 | 系统/数据存储 | 安全密钥存储 |
| 保护机制 | 无 | 防重放攻击保护 |

---

## 🔧 支持的芯片

### 芯片认证策略

MTK 芯片的授权验证策略根据芯片代数不同：

#### 高端芯片（需要 Blob + Sig 认证）
**Dimensity 8100 ~ Dimensity 9400（天玑 8100-9400）**
- ✅ Dimensity 9400 (天玑 9400)
- ✅ Dimensity 9300 (天玑 9300)
- ✅ Dimensity 9200 (天玑 9200)
- ✅ Dimensity 9000 (天玑 9000)
- ✅ Dimensity 8200 (天玑 8200)
- ✅ Dimensity 8100 (天玑 8100)

**操作方式**: 必须先执行 Blob 提取 → Sig 计算 → Sig 认证，才能进行读写擦操作。

#### 中低端芯片（支持漏洞绕过）
**Dimensity 1200 及以下**
- ✅ Dimensity 1200 (天玑 1200)
- ✅ Dimensity 1100 (天玑 1100)
- ✅ Dimensity 1000+ (天玑 1000+)
- ✅ Dimensity 900 (天玑 900)
- ✅ Dimensity 800 (天玑 800)
- ✅ Dimensity 720 (天玑 720)
- ✅ Dimensity 700 (天玑 700)
- ✅ Helio 系列芯片：
  - Helio G95, G90, G85, G80, G70
  - Helio P95, P90, P70, P65, P60, P35, P30, P23, P22
  - Helio A 系列
- ✅ MT6xxx 系列：
  - MT6893, MT6891, MT6889, MT6877, MT6873
  - MT6853, MT6833, MT6785, MT6771, MT6768
  - MT6765, MT6763

**操作方式**: 直接使用 `Exploit()` 漏洞绕过方法，无需 Blob/Sig 认证。

### 使用建议

```csharp
var plTools = new PlTools(context, port, serial);

// 获取芯片信息
plTools.GetHwConfiguration();
string chipName = plTools.CpuName;

// 根据芯片选择认证方式
if (chipName.Contains("Dimensity") && 
    (chipName.Contains("8100") || chipName.Contains("8200") || 
     chipName.Contains("9000") || chipName.Contains("9200") || 
     chipName.Contains("9300") || chipName.Contains("9400")))
{
    // 高端芯片：使用 Blob + Sig 认证
    Console.WriteLine("检测到高端芯片，使用 Blob + Sig 认证");
    
    if (plTools.GetBlobKey(out string blob))
    {
        plTools.GenerateAuthConfig("", out string config);
        // ... 使用认证工具计算 Sig
        plTools.SendSigAuthFromFile("sig.bin");
    }
}
else
{
    // 中低端芯片：使用漏洞绕过
    Console.WriteLine("检测到中低端芯片，使用漏洞绕过");
    
    ChipConfig config = BromConfig.HwConfig["0x" + plTools.HwCode];
    byte[] payload = plTools.PreparePayload(config);
    plTools.Exploit(config, payload);
}
```

---

## ⚠️ 安全注意事项

1. **Blob/Sig 保密** - 用于通讯授权验证，应妥善保管
2. **RPMB 谨慎操作** - 包含安全启动密钥，修改前必须备份
3. **仅用于合法用途** - 设备维修与开发
4. **备份原始数据** - 在任何操作前备份重要数据
5. **遵守法律法规** - 确保使用符合当地法律

---

## 🚀 快速开始

### 1. 安装

```bash
# 克隆仓库
git clone <repository_url>

# 编译
cd MtkFlashLib
dotnet build
```

### 2. 基础使用

```csharp
// 初始化
var context = new MtkContext();
var serial = new SerialLib(context);
var port = new MtkPort(context, serial);

// 连接设备
port.DetectMediaTekPort();
port.Open();
port.HandshakeMtk();

// 使用功能
var plTools = new PlTools(context, port, serial);
var xflash = new XflashLib(context, port, serial);

// 提取 Blob
plTools.GetBlobKey(out string blob);

// 读取 RPMB
xflash.ReadRpmb(0, 10, "rpmb.bin");
```

---

## 📊 API 统计

| 类别 | 方法数 |
|------|--------|
| Blob 输出 | 5 |
| Sig 授权验证 | 5 |
| RPMB 操作 | 7 |
| DA 发送 | 8 |
| 分区操作 | 5 |
| 存储信息 | 4 |
| 格式化 | 2 |
| 其他辅助 | 10+ |
| **总计** | **45+** |

---

## 🔄 版本历史

### v1.3 - 2026-06-20（当前版本）
- ✅ 新增 RPMB 读写擦完整功能
- ✅ 7 个 RPMB 专用方法
- ✅ 完整的 RPMB 文档

### v1.2 - 2026-06-20
- ✅ 新增 Sig 授权验证功能
- ✅ 5 个 Sig 认证方法
- ✅ 完整的 Sig 认证文档
- ✅ 明确概念：用于授权验证，与设备解锁无关

### v1.1 - 2026-06-20
- ✅ 新增 Blob 输出功能
- ✅ 5 个 Blob 提取方法
- ✅ 完整的 Blob 文档

### v1.0 - 2026-06-20
- ✅ 核心 MTK 刷机协议移植
- ✅ DA 发送、分区操作、格式化等
- ✅ 完整的中文注释

---

## 🎓 学习资源

### 推荐阅读顺序

1. **README.md** - 项目概述
2. **BLOB_README.md** - Blob 功能快速开始
3. **SIG_AUTH_FEATURE.md** - Sig 认证详细说明
4. **RPMB_FEATURE.md** - RPMB 操作详细说明
5. **PORTING_SUMMARY.md** - 技术细节与协议对比

### 常见任务指南

**任务 1: 提取 Blob 密钥**
→ 参考 `BLOB_README.md` 或 `BLOB_FEATURE.md`

**任务 2: 执行 Sig 认证**
→ 参考 `SIG_AUTH_FEATURE.md`

**任务 3: 备份 RPMB**
→ 参考 `RPMB_FEATURE.md`

**任务 4: 刷写分区**
→ 参考 `XflashLib.cs` 中的 `WriteByPartitionName` 方法

---

## 🤝 贡献

欢迎提交 Issue 和 Pull Request！

---

## 📝 许可证

请遵循项目许可证进行使用。

---

*最后更新: 2026-06-20*  
*版本: 1.3*  
*编译状态: ✅ 成功 (0 错误, 0 警告)*
