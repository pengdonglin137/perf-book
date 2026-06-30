\phantomsection
# 附录 B. 启用大页 {.unnumbered}

\markboth{附录 B}{附录 B}

## Windows {.unnumbered}

要在 Windows 上使用大页，你需要启用 `SeLockMemoryPrivilege` [安全策略](https://docs.microsoft.com/en-us/windows/security/threat-protection/security-policy-settings/lock-pages-in-memory)。这可以通过 Windows API 以编程方式完成，或者通过安全策略 GUI 完成。

1. 单击开始 → 搜索"secpol.msc"，然后启动它。
2. 在左侧选择"本地策略" → "用户权限分配"，然后双击"锁定内存中的页"。

![Windows 安全：锁定内存中的页](../../img/appendix-C/WinLockPages.png){width=100%}

3. 添加你的用户并重新启动机器。

4. 使用 [RAMMap](https://docs.microsoft.com/en-us/sysinternals/downloads/rammap) 工具检查运行时是否使用了大页。

在代码中使用大页：

```cpp
void* p = VirtualAlloc(NULL, size, MEM_RESERVE | 
                                   MEM_COMMIT | 
                                   MEM_LARGE_PAGES,
                       PAGE_READWRITE);
...
VirtualFree(ptr, 0, MEM_RELEASE);
```

## Linux {.unnumbered}

在 Linux 操作系统上，有两种方式在应用程序中使用大页：显式大页和透明大页。

### 显式大页 {.unlisted .unnumbered}

显式大页可以在系统启动时或在应用程序启动之前保留。要永久更改以强制 Linux 内核在启动时分配 128 个大页，请运行以下命令：

```bash
$ echo "vm.nr_hugepages = 128" >> /etc/sysctl.conf
```
