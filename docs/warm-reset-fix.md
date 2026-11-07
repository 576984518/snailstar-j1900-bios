# 热复位无限重启：根因与修复

## 1. 现象

| 操作 | 结果 |
|---|---|
| Windows 启动管理器（F8）按 `Ctrl+Alt+Del` | 黑屏，无限重启，只能拔电 |
| WinRE（Shift+重启）按 `Ctrl+Alt+Del` | 同上 |
| 用 PE 启动后热复位 | 同上 |
| 关机再冷启动 | 正常 |
| Windows 内点“重启” | 正常 |

由于 PE 也复现，排除 Windows / BCD / 引导项问题。

## 2. 定位方法

1. 从固件镜像中提取所有与复位相关的模块，按名称比对；
2. 发现只有 `PchReset`（DXE 驱动，GUID `F5883FC5-F8EE-4E44-B386-6021FB320C9B`）在两个固件之间差异明显：
   - 本机：2848 字节，`.text` 2024 字节
   - 七彩虹 C.Q1900M：6240 字节，`.text` 5328 字节
3. 其余运行时驱动（`RuntimeDxe`、`CapsuleRuntimeDxe`、`PchSpiRuntime`）大小完全一致。

## 3. 反汇编

`PchReset` 的复位入口（`sub_774`）反编译后大致为：

```c
void sub_774(int resetType)
{
    if (!resetType) {                  // 0：冷复位
        outb(0xCF9, 2);
        outb(0xCF9, 6);                // ← 本平台忽略 0x06
        while (1) ;                     // ← 死转
    }
    if (resetType - 1 == 0) {          // 1：热复位
        sub_694();                      // 胶囊更新路径（依赖不存在的变量）
        goto cold_path;                 // 同样写 0xCF9
    }
    ...
}
```

`0xCF9` 是 PCH 的复位控制寄存器：

- `0x02`：SYS_RST
- `0x06`：SYS_RST + CPU 复位 —— **在本平台被忽略**
- `0x0E`：全复位（含 ME）—— 有效

## 4. 补丁

| 偏移 | 原 | 新 |
|---|---|---|
| `0x7FF` | `06` | `0E` |
| `0x851` | `06` | `0E` |
| `0x7F1`~`0x7F5` | `E8 9E FE FF FF`（call sub_694） | `90 90 90 90 90` |

只改 7 个字节，模块大小不变（2848B）。

## 5. 验证

- WinRE 中 `Ctrl+Alt+Del` → 正常重启；
- 启动管理器 → 正常；
- 多次冷热交替启动无异常。

## 6. 复现补丁

```python
b = bytearray(open("PchReset.bin","rb").read())
assert b[0x7F1:0x7F6] == bytes.fromhex("e89efeffff")
b[0x7F1:0x7F6] = b"\x90"*5     # 跳过 sub_694
assert b[0x7FF] == 0x06 and b[0x851] == 0x06
b[0x7FF] = 0x0E                # 全复位
b[0x851] = 0x0E
open("PchReset_patched.bin","wb").write(b)
```

替换方式：UEFITool → `PchReset` → `PE32 image section` → `Replace body…`。
