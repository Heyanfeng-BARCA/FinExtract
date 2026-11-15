# FinExtract 快速上手

适用于 64 位 Windows 10 / 11。完整版本包含本地服务和浏览器界面。

## 1. 下载并解压

1. 打开 [v1.0 发布页](https://github.com/Heyanfeng-BARCA/FinExtract/releases/tag/v1.0)，在 **Assets** 下选择 [FinExtract_Windows_Portable_v1.0.zip](https://github.com/Heyanfeng-BARCA/FinExtract/releases/download/v1.0/FinExtract_Windows_Portable_v1.0.zip)。GitHub 自动生成的 **Source code (zip)** 只包含仓库文件。
2. 右键点击 ZIP，选择「全部解压缩」。保留 `FinExtract.exe`、`_internal` 文件夹及两个 `.bat` 文件的原有目录结构。
3. 在解压目录双击 `启动 FinExtract.bat`。如果浏览器没有自动打开，请访问启动窗口显示的地址，默认是 `http://127.0.0.1:8000/`。

## 2. 配置模型密钥

1. 在文件资源管理器的地址栏输入 `%LOCALAPPDATA%\FinExtract\config`，按回车进入。首次启动会创建该目录。
2. 新建 `keys.txt`，写入下方内容。将占位文字换成你自己的密钥，等号两侧不留空格。使用迅捷方案时配置 DeepSeek 即可。启用智衡方案的独立复核时再配置 Qwen Plus。

   ```text
   DEEPSEEK_API_KEY=你的DeepSeek密钥
   QWEN_API_KEY=你的阿里云百炼密钥
   ```

3. 双击 `关闭 FinExtract.bat` 停止服务，再运行 `启动 FinExtract.bat`。在工作台确认所需方案可用。请把真实密钥留在本机，提交问题时也请移除密钥及敏感文档内容。

## 3. 完成首次提取

1. 打开「智能文档处理」，选择处理方案，上传一份公告 PDF 并开始提取。
2. 任务完成后，在「数据审查中心」检查提取结果。可在「质量评估与分析」查看质量指标。
3. 使用完毕，双击 `关闭 FinExtract.bat` 停止本地服务。

## 下载校验与常见问题

在下载目录打开 PowerShell，运行以下命令。

```powershell
Get-FileHash -Algorithm SHA256 .\FinExtract_Windows_Portable_v1.0.zip
```

预期 SHA-256 为 `2422ad541bed620233b4a7c1e912c274d7489bd74849d280c7660c31567ffb7f`，大小写均可。

- 页面打不开时，确认启动窗口仍在运行，并以窗口显示的地址为准。
- 处理方案不可用时，检查配置目录、密钥拼写和模型账户额度，保存后重启。
- [在线使用手册](https://heyanfeng-barca.github.io/FinExtract/manual.html)包含详细的配置和排查步骤。也可以打开解压目录内的 `使用手册.html`。
- [报告问题](https://github.com/Heyanfeng-BARCA/FinExtract/issues)时，请附上 Windows 版本、FinExtract 版本、复现步骤与脱敏后的错误信息。

[返回仓库首页](README.md) · [打开在线界面预览](https://heyanfeng-barca.github.io/FinExtract/)
