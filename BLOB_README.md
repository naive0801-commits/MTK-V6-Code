# MTK Blob 输出功能 - 快速开始

## 🎯 功能简介

MTK Blob 输出功能用于从 MediaTek 芯片提取认证密钥，支持设备解锁、FRP 绕过等高级场景。

## ✅ 已实现的 5 个核心方法

| 方法 | 功能 | 输出格式 |
|------|------|----------|
| `GetBlobKey` | 获取 Blob 密钥 | 十六进制 |
| `GetBlobKeyBase64` | 获取 Blob 密钥 | Base64 |
| `SaveBlobKeyToFile` | 保存到文件 | 十六进制/Base64 |
| `GenerateAuthConfig` | 生成认证配置 | 十六进制 |
| `GenerateAuthConfigBase64` | 生成认证配置 | Base64 |

## 🚀 快速使用

### 1. 基础提取
```csharp
var plTools = new PlTools(context, port, serial);

// 获取 Blob 密钥
if (plTools.GetBlobKey(out string blobKey))
{
    Console.WriteLine($"密钥: {blobKey}");
}
```

### 2. 保存到文件
```csharp
// 十六进制格式
plTools.SaveBlobKeyToFile("blob.txt");

// Base64 格式
plTools.SaveBlobKeyToFile("blob_base64.txt", useBase64: true);
```

### 3. 生成认证配置
```csharp
// 生成完整认证配置（包含 SOC-ID + Blob）
if (plTools.GenerateAuthConfigBase64("", out string config))
{
    File.WriteAllText("auth_config.txt", config);
}
```

## 📊 数据格式

### Blob 密钥
- **长度**: 64 个十六进制字符（32 字节）
- **示例**: `a1b2c3d4e5f6...`

### 认证配置
- **格式**: `0200000001340220` + `SOC_ID` + `0310` + `BLOB_KEY`
- **用途**: 外部认证工具、FRP 解锁

## 📖 完整文档

详细使用说明、技术细节、故障排除请查看：
- **BLOB_FEATURE.md** - 完整功能文档
- **PORTING_SUMMARY.md** - 项目总体说明

## ✨ 编译状态

✅ **0 个错误**  
✅ **0 个警告**  
✅ **100% 可用**

---
*版本: 1.1 | 更新: 2026-06-20*
