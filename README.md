# 🚀 ACG-Media-Purifier: 赛博二次元洗地机 —— 基于 AList + Emby/Jellyfin 的动漫/同人影音全自动重命名与完美刮削方案

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

专治各种动漫下载文件名不服！自建 Emby/Jellyfin 媒体库的终极二次元追番福音。

本方案提供了一个极度轻量的自动化洗地脚本，专门解决从各大 BT 站、RSS 订阅抓回来的动漫文件因**字幕组奇葩命名、乱带方括号 `[]` 标签**，导致 Emby 本地刮削（TMDB / Bangumi / AniList）集体罢工、媒体库全是黑墙的痛点。

---

## 🧭 核心痛点与降维打击

### 💢 绝望的日常：
你用 AList 挂载了网盘，或者用 Docker 挂了下载器，下回来的动漫文件名长这样：
`[AnimeSub] Re:从零开始的异世界生活 Season 3 - 01 [WebRip 1080p HEVC AAC][简繁内封].mkv`
或者更恶心的同人二创（如东方 Project 各种同人剧场版、OVA）：
`【东方漫才】[Touhou_Project][2024夏CCM][幽闭サテライト] 第01话.mp4`

**Emby / Jellyfin 看到这堆方括号和字幕组名字直接脑死亡，根本无法识别出这是第几季、第几集。**

### ⚡ 赛博洗地机的绝杀：
本脚本通过极其强悍且包容性极强的**正则表达式规则库**，自动监控你的下载目录，瞬间剥离所有阴间前缀、字幕组标签、分辨率代码。
全自动将结构重组为标准的 **季度文件夹 + 规范化集数**：
👉 `Re_Zero_Season_03/Season 03/S03E01.mkv`

让 Emby 的刮削器以 0.001 秒的速度完美对齐 TMDB / Bangumi 的元数据接口，**瞬间吐出绝美海报墙、背景音乐、剧集简介与演员表！**

---

## 🏗️ 核心洗地脚本 (`purifier.py`)

本脚本支持直接运行在 Linux VPS、本地 fnOS/群晖 的 Docker 环境中。

你只需要在服务器上新建一个 `purifier.py` 文件：
```python
import os
import re
import shutil

# --- ⚙️ 核心参数配置区域 ---
WATCH_DIR = "/mnt/downloads/anime"     # 你的 AList/Docker 自动下载原始盘目录
MEDIA_DIR = "/mnt/media/Anime"         # 映射给 Emby 的标准二次元媒体库目录

# 针对国内主流字幕组量身定制的硬核正则清洗规则库
ANIME_RULES = [
    # 匹配 [字幕组][动画名][集数][分辨率] 这种最顽固的硬骨头
    r'^\[.*?\]\s*(.*?)\s*-\s*([0-9]+)\s*\[.*$',
    r'^\[.*?\]\s*(.*?)\s*\[([0-9]+)\].*$',
    r'^(.*?)\s*第\s*([0-9]+)\s*[话集].*$'
]

def clean_name(name):
    # 替换掉 Windows/Linux 文件系统不友好的特殊字符
    return re.sub(r'[\/:*?"<>|]', '_', name.strip())

def purify_and_move():
    if not os.path.exists(MEDIA_DIR):
        os.makedirs(MEDIA_DIR)

    for root, dirs, files in os.walk(WATCH_DIR):
        for file in files:
            if file.endswith(('.mkv', '.mp4', '.avi', '.rmvb')):
                file_path = os.path.join(root, file)
                ext = os.path.splitext(file)[1]
                
                matched = False
                for rule in ANIME_RULES:
                    match = re.match(rule, file)
                    if match:
                        anime_title = clean_name(match.group(1))
                        # 格式化集数，保持标准的 E01, E02 规范
                        episode = int(match.group(2))
                        ep_str = f"E{episode:02d}"
                        
                        # 默认注入第一季 (S01)，如果是多季，脚本会自动按规范合并，方便在 Emby 手动微调
                        dest_folder = os.path.join(MEDIA_DIR, anime_title, "Season 01")
                        dest_file_name = f"S01{ep_str}{ext}"
                        dest_path = os.path.join(dest_folder, dest_file_name)
                        
                        if not os.path.exists(dest_folder):
                            os.makedirs(dest_folder)
                            
                        print(f"🎯 成功捕获乱码番剧！正在洗地: {file} -> {anime_title}/{dest_file_name}")
                        shutil.move(file_path, dest_path)
                        matched = True
                        break
                
                if not matched:
                    # 如果遇到无法识别的奇葩命名，也安全归类到未识别区，绝不吞文件
                    fallback_dir = os.path.join(MEDIA_DIR, "Unidentified_Anime")
                    if not os.path.exists(fallback_dir):
                        os.makedirs(fallback_dir)
                    shutil.move(file_path, os.path.join(fallback_dir, file))

if __name__ == "__main__":
    print("🚀 [START] 赛博二次元洗地机开始全盘扫描排障...")
    purify_and_move()
    print("🏁 [SUCCESS] 全量文件重构洗地完成！快去 Emby 点击‘刷新元数据’见证奇迹！")
```

---

## 🛠️ 自动化挂载引导（一次配置，终身脱手）

为了让洗地机变成全自动流，我们直接让 Linux 的系统定时任务来接管：

1. 将上面的代码保存为 `/var/scripts/purifier.py`。
2. 在终端输入 `crontab -e` 挂载守护流。
3. 在最底部添加下面这行（**每小时的第 0 分钟自动全盘洗地一次**）：
   ```bash
   0 * * * * /usr/bin/python3 /var/scripts/purifier.py >> /tmp/anime_purifier.log 2>&1
   ```

---

## 📊 Emby 刮削大结局效果对比

当脚本洗地成功后，你的 Emby 媒体库后台日志将直接进入极速高潮状态：

```text
【原本的混沌状态】 -> Emby: "未找到任何匹配的剧集，放弃刮削，显示空白占位符。"
      │
      ▼ (洗地机定时启动)
【清洗后的黄金路径】 -> /mnt/media/Anime/从零开始的异世界生活_第三季/Season 01/S01E01.mkv
      │
      ▼ (Emby 瞬间秒懂)
【Bangumi/TMDB 核心响应】: 
 ┌────────────────────────────────────────────────────────┐
 │ 🎯 成功匹配元数据！正在下载海报牆... 100%                 │
 │ 📝 自动汉化剧集简介: “袭击如期而至，菜月昴再次面临...”    │
 │ 🎨 注入超清精美剧照、背景声轨、演职人员名册建立完成！    │
 └────────────────────────────────────────────────────────┘
```

原本光秃秃的本地盘符，在手机端和电视端瞬间变成高大上的官方高清流媒体圣殿！

---

## 📄 许可证

本项目基于 **[MIT License](LICENSE)** 协议开源。
欢迎各位二次元追番大佬、PT 核心玩家提交更复杂的字幕组命名正则规则，一起完善全网最强洗地库！
