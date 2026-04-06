# aroll-skill

一个 [Claude Code](https://docs.anthropic.com/en/docs/claude-code) 斜杠命令，自动完成口播/教程类视频的 **A-roll 粗剪**。

丢一个素材文件夹进去 → 自动转写、识别好坏 take、去掉静音 → 在达芬奇里生成可直接编辑的 **时间线 + SRT 字幕**。

## 它做了什么

```
原始素材文件夹
  ↓  FFmpeg 提取音频
  ↓  Whisper 语音转写（逐字时间戳）
  ↓  静音检测
  ↓  LLM 分析：保留/丢弃 take，去重复段落
  ↓  生成达芬奇 Resolve Python 脚本
  ↓  生成 SRT 字幕（对齐剪辑后时间线）
  ↓  执行 → 达芬奇里直接出现时间线
```

核心逻辑：很多创作者录制时会在满意的 take 后说 **"OK"**，不满意时说 **"pass"/"重来"**。这个工具能识别这些口头标记，自动只保留最佳 take。如果你录制时不说标记词也没关系——它会保留所有语音段落，只切掉静音。

## 环境要求

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code)（CLI 或 IDE 插件）
- [FFmpeg](https://ffmpeg.org/) — 音频提取和静音检测
- [OpenAI Whisper](https://github.com/openai/whisper) — 语音转写（`pip install openai-whisper`）
- [DaVinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve)（免费版或 Studio 版）— 执行脚本时需要打开

## 安装

把 `aroll.md` 复制到你项目的 `.claude/commands/` 目录：

```bash
mkdir -p .claude/commands
cp aroll.md .claude/commands/
```

搞定，`/aroll` 命令就可以用了。

## 使用方法

在 Claude Code 中输入：

```
/aroll ~/Videos/产品评测第12期 landscape
```

或者直接输入 `/aroll`，助手会主动问你：
- **素材文件夹路径**
- **横屏还是竖屏**
- **项目名称**（可选）
- **帧率**（可选，默认 23.976）
- **语言**（可选，默认自动检测）

助手会依次：
1. 扫描并列出素材文件
2. 用 Whisper 转写全部音频
3. 分析剪辑点，展示决策列表
4. 等你确认后才执行
5. 生成达芬奇时间线 + SRT 字幕

## 保留/丢弃逻辑

| 录制时说的话 | 结果 |
|-------------|------|
| "OK" / "好" / "keep" / "good" | 前面那段 **保留** |
| "pass" / "不要" / "cut" / "again" / "redo" | 前面那段 **丢弃** |
| *（什么都没说，一直在讲）* | 保留所有语音段，只切静音 |

同一句话录了多遍？只保留**最后一个好 take**（离 keep 标记最近的那个）。

## 自定义

直接用自然语言告诉助手就行：

- *"用 29.97 帧率和 1080p"*
- *"我的保留标记是 'nice'，丢弃标记是 'nope'"*
- *"静音阈值 -40dB，我这环境比较吵"*
- *"用 whisper large 模型，精度高一些"*

完整配置表见 [aroll.md](aroll.md#configuration-reference)。

## 平台说明

- **macOS**：开箱即用，默认路径适配标准达芬奇安装
- **Windows / Linux**：生成的脚本中有注释，标明了需要修改路径的位置

---

# aroll-skill (English)

A [Claude Code](https://docs.anthropic.com/en/docs/claude-code) slash command that automates **A-roll rough cuts** for talking-head and voiceover videos.

Give it a folder of clips → it transcribes, detects good/bad takes, removes silence, and builds a ready-to-edit **DaVinci Resolve timeline** with synced **SRT subtitles**.

## What it does

```
Raw footage folder
  ↓  FFmpeg extract audio
  ↓  Whisper transcribe (word-level timestamps)
  ↓  Silence detection
  ↓  LLM analyzes: keep/drop takes, remove repeats
  ↓  Generates DaVinci Resolve Python script
  ↓  Generates SRT subtitles (aligned to edited timeline)
  ↓  Executes → timeline ready in DaVinci Resolve
```

The key insight: many creators say **"OK"** after a good take and **"pass" / "again"** after a bad one. This skill understands those spoken markers and automatically keeps only the best takes. If you don't use markers, it still works — it'll keep all speech segments and just cut the silence.

## Requirements

- [Claude Code](https://docs.anthropic.com/en/docs/claude-code) (CLI or IDE extension)
- [FFmpeg](https://ffmpeg.org/) — for audio extraction and silence detection
- [OpenAI Whisper](https://github.com/openai/whisper) — for transcription (`pip install openai-whisper`)
- [DaVinci Resolve](https://www.blackmagicdesign.com/products/davinciresolve) (Free or Studio) — must be running when the script executes

## Installation

Copy `aroll.md` into your project's `.claude/commands/` directory:

```bash
mkdir -p .claude/commands
cp aroll.md .claude/commands/
```

That's it. The `/aroll` command is now available in Claude Code.

## Usage

In Claude Code, type:

```
/aroll ~/Videos/product-review-ep12 landscape
```

Or just type `/aroll` and the assistant will ask you for:
- **Footage folder path**
- **Landscape or portrait**
- **Project name** (optional)
- **Frame rate** (optional, default 23.976)
- **Language** (optional, default auto-detect)

The assistant will:
1. Scan and list your footage files
2. Transcribe everything with Whisper
3. Analyze edit points and show you the decision list
4. Wait for your confirmation
5. Generate and run the DaVinci Resolve script
6. Output an SRT subtitle file

## How the keep/drop logic works

| You say during recording | What happens |
|--------------------------|-------------|
| "OK" / "好" / "keep" / "good" | Previous segment is **kept** |
| "pass" / "不要" / "cut" / "again" / "redo" | Previous segment is **dropped** |
| *(nothing — just keep talking)* | All speech segments are kept, silence is cut |

When you record multiple takes of the same line, the skill keeps the **last good take** (closest to a keep marker).

## Customization

You can adjust defaults by telling the assistant in natural language:

- *"Use 29.97 fps and 1080p"*
- *"My keep marker is 'nice', drop marker is 'nope'"*
- *"Silence threshold -40dB, I'm in a noisy environment"*
- *"Use whisper large model for better accuracy"*

See the full configuration table in [aroll.md](aroll.md#configuration-reference).

## Platform notes

- **macOS**: Works out of the box with standard DaVinci Resolve installation paths
- **Windows/Linux**: The generated script includes comments showing where to adjust DaVinci Resolve paths

## License

MIT
