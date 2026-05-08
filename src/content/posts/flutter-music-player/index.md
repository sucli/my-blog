---
title: 用 Flutter 从零搭建跨端音乐播放器
published: 2026-05-08
description: "从架构设计到代码实现，完整记录如何用 Flutter 搭建一个支持插件化音源、歌词同步、均衡器的跨端音乐播放器。"
tags: ["Flutter", "音乐播放器", "跨端开发", "Dart"]
category: 技术
draft: false
---

## 前言

一直想做一个自己的音乐播放器，不为别的，就是想要一个干净、无广告、可自定义的播放器。正好最近在玩 Flutter，就决定用 Flutter 来实现，一套代码覆盖 iOS、Android、Web 和桌面端。

本文完整记录了从架构设计到代码实现的全过程，包括：

- 插件化音源架构设计
- 音频播放引擎集成
- 歌词同步显示
- 均衡器音效
- 跨平台权限配置

## 技术选型

### 为什么选 Flutter？

| 方案 | 优点 | 缺点 |
|------|------|------|
| **Flutter** | 一套代码跨端、UI一致性好、性能强 | 包体略大 |
| React Native | JS生态丰富、社区活跃 | 后台播放支持较弱 |
| Electron | 桌面端体验好 | 包体巨大（100MB+） |
| Tauri | 极小包体 | 移动端生态不成熟 |

最终选择 Flutter，核心原因：

1. **just_audio + audio_service** 组合成熟，后台播放支持好
2. 跨端 UI 一致性最高
3. Dart 语言上手快，类型安全

### 核心依赖

```yaml
dependencies:
  just_audio: ^0.9.42          # 音频播放引擎
  audio_session: ^0.1.21       # 音频焦点管理
  provider: ^6.1.2             # 状态管理
  sqflite: ^2.4.2              # SQLite 本地存储
  dio: ^5.7.0                  # 网络请求
  cached_network_image: ^3.4.1 # 图片缓存
  palette_generator: ^0.3.3+4  # 封面取色
```

## 架构设计

### 插件化音源架构

这是整个项目最核心的设计。我采用了插件化架构，将音源与播放器核心解耦：

```
播放器核心（App）
      ↓ 注册/加载
┌─────┴─────┐
│  音源插件   │  ← 每个源是一个独立插件
│  接口层    │     实现统一的 search/getUrl/list 接口
└─────┬─────┘
      ↓
统一输出: { title, artist, url, cover, lyric }
```

**好处：**

- 平台和音源解耦，降低法律风险
- 用户自行导入音源，播放器只是壳子
- 新增音源只需写插件，不改核心代码

### 插件接口定义

```dart
/// 音源插件抽象接口
abstract class MusicSourcePlugin {
  Future<void> initialize(SourceConfig config);
  String get name;
  String get id;
  
  /// 搜索歌曲
  Future<SearchResult> search(String keyword, {int page = 1});
  
  /// 获取播放地址
  Future<String> getPlayUrl(String songId);
  
  /// 获取歌词
  Future<List<LyricLine>> getLyrics(String songId);
  
  /// 获取推荐歌单
  Future<List<Playlist>> getRecommendPlaylists();
}
```

### 插件管理器

```dart
class PluginManager {
  final Map<String, MusicSourcePlugin> _plugins = {};
  
  /// 跨音源搜索，合并所有源的结果
  Future<List<Song>> searchAll(String keyword) async {
    final futures = _plugins.values
        .where((p) => p.isInitialized)
        .map((p) => p.search(keyword));
    
    final results = await Future.wait(futures, 
        eagerError: true);
    
    return results.expand((r) => r.songs).toList();
  }
}
```

## 项目结构

```
lib/
├── main.dart                    # 应用入口
├── models/                      # 数据模型
│   ├── song.dart                # 歌曲
│   ├── playlist.dart            # 歌单
│   ├── lyric_line.dart          # 歌词行
│   └── source_config.dart       # 音源配置
├── plugins/                     # 插件化音源
│   ├── music_source_plugin.dart # 接口
│   ├── plugin_manager.dart      # 管理器
│   ├── netease_plugin.dart      # 网易云（学习用）
│   └── jamendo_plugin.dart      # Jamendo（CC免费）
├── services/                    # 服务层
│   ├── audio_service.dart       # 播放引擎
│   ├── storage_service.dart     # SQLite 存储
│   └── equalizer_service.dart   # 均衡器
├── providers/                   # 状态管理
│   ├── player_provider.dart     # 播放器状态
│   ├── search_provider.dart     # 搜索状态
│   ├── playlist_provider.dart   # 歌单状态
│   └── theme_provider.dart      # 主题状态
├── screens/                     # 页面
│   ├── home/                    # 首页（底部导航）
│   ├── player/                  # 全屏播放页
│   ├── search/                  # 搜索页
│   └── settings/                # 设置页
├── widgets/                     # 通用组件
│   ├── song_list_tile.dart      # 歌曲列表项
│   ├── mini_player.dart         # 迷你播放器
│   ├── lyric_view.dart          # 歌词同步
│   ├── equalizer_dialog.dart    # 均衡器面板
│   └── progress_bar.dart        # 进度条
└── utils/
    ├── constants.dart           # 常量
    └── helpers.dart             # 工具函数
```

## 核心功能实现

### 1. 音频播放引擎

使用 just_audio 作为底层引擎，封装为 MusicAudioService：

```dart
class MusicAudioService {
  final AudioPlayer _player = AudioPlayer();
  List<Song> _playlist = [];
  PlayMode _playMode = PlayMode.listLoop;
  
  /// 播放指定歌曲
  Future<void> play(Song song) async {
    if (song.audioUrl == null || song.audioUrl!.isEmpty) {
      // 从插件获取播放地址
      final plugin = PluginManager().getPlugin(song.sourceId);
      final url = await plugin?.getPlayUrl(song.id);
      if (url != null) {
        song = song.copyWith(audioUrl: url);
      }
    }
    await _player.setUrl(song.audioUrl!);
    await _player.play();
  }
  
  /// 播放模式切换
  PlayMode togglePlayMode() {
    const modes = PlayMode.values;
    final nextIndex = (modes.indexOf(_playMode) + 1) % modes.length;
    _playMode = modes[nextIndex];
    return _playMode;
  }
}
```

**播放模式实现：**

- **列表循环**：播放到末尾自动回到开头
- **单曲循环**：重复播放当前歌曲
- **随机播放**：随机选择下一首
- **顺序播放**：播放完停止

### 2. 歌词同步显示

歌词采用标准 LRC 格式解析，支持自动滚动和手动浏览：

```dart
class LyricView extends StatefulWidget {
  final String? lrcContent;
  
  // LRC 解析
  static List<LyricLine> parseLrc(String content) {
    final lines = <LyricLine>[];
    final regExp = RegExp(r'\[(\d{2}):(\d{2})\.(\d{2,3})\](.*)');
    
    for (var line in content.split('\n')) {
      final match = regExp.firstMatch(line);
      if (match != null) {
        final min = int.parse(match.group(1)!);
        final sec = int.parse(match.group(2)!);
        final ms = int.parse(match.group(3)!.padRight(3, '0'));
        final text = match.group(4)!.trim();
        
        if (text.isNotEmpty) {
          lines.add(LyricLine(
            timestamp: Duration(minutes: min, seconds: sec, milliseconds: ms),
            text: text,
          ));
        }
      }
    }
    return lines..sort((a, b) => a.timestamp.compareTo(b.timestamp));
  }
}
```

**同步逻辑：**

```dart
// 根据播放位置高亮当前行
int _getCurrentLineIndex(Duration position) {
  for (int i = _lyricLines.length - 1; i >= 0; i--) {
    if (position >= _lyricLines[i].timestamp) {
      return i;
    }
  }
  return 0;
}

// 自动滚动到当前行
void _scrollToLine(int index) {
  if (_isUserScrolling) return; // 用户手动滚动时不自动跳转
  
  _scrollController.animateTo(
    index * _lineHeight,
    duration: Duration(milliseconds: 300),
    curve: Curves.easeInOut,
  );
}
```

### 3. 均衡器

基于 just_audio 的 AndroidEqualizer 实现 5 频段均衡器：

```dart
class EqualizerService {
  AndroidEqualizer? _equalizer;
  
  /// 预设音效
  static const Map<String, List<double>> presets = {
    '流行': [-1, 2, 4, 2, -1],
    '摇滚': [4, 2, -1, -2, 4],
    '古典': [3, 1, -1, 0, 3],
    '爵士': [2, 4, 0, 2, -2],
    '人声': [-2, -1, 3, 4, 2],
  };
  
  /// 设置频段增益
  Future<void> setBandGain(int bandIndex, double gain) async {
    if (_equalizer == null) return;
    final params = await _equalizer!.parameters;
    final bands = params.bands;
    if (bandIndex < bands.length) {
      await bands[bandIndex].setGain(gain);
    }
  }
}
```

**均衡器 UI：**

- 5 个垂直滑块，对应 60Hz、230Hz、910Hz、3.6kHz、14kHz
- 增益范围 -12dB 到 +12dB
- 6 个预设按钮：自定义、流行、摇滚、古典、爵士、人声

### 4. 播放页 UI

全屏播放页是最复杂的 UI，包含：

- 封面旋转动画（播放时旋转，暂停时停止）
- 渐变背景（从封面提取主色调）
- 进度条（可拖拽）
- 播放控制按钮
- 歌词/封面切换

```dart
class PlayerPage extends StatefulWidget {
  @override
  _PlayerPageState createState() => _PlayerPageState();
}

class _PlayerPageState extends State<PlayerPage> 
    with SingleTickerProviderStateMixin {
  late AnimationController _rotateController;
  bool _showLyrics = false;
  
  @override
  void initState() {
    super.initState();
    _rotateController = AnimationController(
      vsync: this,
      duration: Duration(seconds: 20),
    );
  }
  
  @override
  Widget build(BuildContext context) {
    return Consumer<PlayerProvider>(
      builder: (context, player, child) {
        // 根据播放状态控制旋转
        if (player.isPlaying) {
          _rotateController.repeat();
        } else {
          _rotateController.stop();
        }
        
        return Container(
          decoration: BoxDecoration(
            gradient: LinearGradient(
              begin: Alignment.topCenter,
              end: Alignment.bottomCenter,
              colors: [_extractedColor, Colors.black],
            ),
          ),
          child: Column(
            children: [
              // 封面/歌词切换
              Expanded(
                child: _showLyrics 
                    ? LyricView(lrcContent: player.currentSong?.lyric)
                    : _buildRotatingCover(),
              ),
              // 进度条
              MusicProgressBar(),
              // 控制按钮
              _buildControls(player),
            ],
          ),
        );
      },
    );
  }
}
```

### 5. 平台权限配置

#### iOS (Info.plist)

```xml
<key>NSMicrophoneUsageDescription</key>
<string>需要麦克风权限用于音频会话管理</string>

<key>UIBackgroundModes</key>
<array>
  <string>audio</string>
</array>
```

#### Android (AndroidManifest.xml)

```xml
<!-- 存储权限 -->
<uses-permission android:name="android.permission.READ_EXTERNAL_STORAGE"/>
<uses-permission android:name="android.permission.READ_MEDIA_AUDIO"/>

<!-- 前台服务（后台播放） -->
<uses-permission android:name="android.permission.FOREGROUND_SERVICE"/>
<uses-permission android:name="android.permission.FOREGROUND_SERVICE_MEDIA_PLAYBACK"/>

<!-- 唤醒锁 -->
<uses-permission android:name="android.permission.WAKE_LOCK"/>
```

## 免费音乐源

### 合法免费源

1. **Jamendo** - CC 协议独立音乐，约 50 万首
2. **Free Music Archive** - CC 音乐合集
3. **Internet Archive** - 公共领域音乐

### 自建服务器

- **Navidrome** - 轻量级音乐服务器，兼容 Subsonic API
- **Jellyfin** - 全媒体服务器

### 学习研究源

- **NeteaseCloudMusicApi** - 网易云音乐非官方 API（仅供学习）

## 使用方式

### 1. 创建 Flutter 项目

```bash
flutter create music_player
cd music_player
```

### 2. 复制源码

```bash
cp -r /path/to/music_player/lib/* music_player/lib/
cp /path/to/music_player/pubspec.yaml music_player/
```

### 3. 安装依赖

```bash
flutter pub get
```

### 4. 运行

```bash
# macOS
flutter run -d macos

# iOS
flutter run -d ios

# Android
flutter run -d android

# Web
flutter run -d chrome
```

## 项目统计

| 指标 | 数值 |
|------|------|
| Dart 文件数 | 34 个 |
| 代码行数 | 8,400+ 行 |
| 支持平台 | iOS / Android / Web / macOS / Windows / Linux |
| 核心功能 | 播放 / 搜索 / 歌单 / 歌词 / 均衡器 / 主题 |

## 踩过的坑

### 1. 后台播放

iOS 和 Android 的后台播放实现完全不同：

- **iOS**: 需要在 Info.plist 配置 `UIBackgroundModes: audio`，并正确设置 `AVAudioSession`
- **Android**: 需要前台服务（Foreground Service）+ 通知栏

直接用 `audio_service` 库可以省很多事。

### 2. 音频焦点

多个音频源同时播放时需要处理音频焦点。`audio_session` 库可以帮助管理：

```dart
final session = await AudioSession.instance;
await session.configure(const AudioSessionConfiguration.music());
```

### 3. 封面图提取

从音频文件提取封面需要读取 ID3 标签，在原生层处理性能最好：

```dart
// 使用 flutter_media_metadata
final metadata = await FlutterMediaMetadata.getMetadataFromFilePath(filePath);
final cover = metadata.coverData;
```

### 4. SQLite 类型映射

Flutter 的 sqflite 不支持布尔类型，需要用 INTEGER 存储：

```dart
// 存储
'song': {'isLiked': song.isLiked ? 1 : 0}

// 读取
song.isLiked = (map['isLiked'] as int) == 1;
```

## 后续计划

- [ ] 歌词搜索与匹配
- [ ] 均衡器预设自定义
- [ ] 跨设备同步（WebDAV）
- [ ] 插件市场（用户上传音源插件）
- [ ] CarPlay / Android Auto 支持

## 总结

这个项目从零到完成大约花了一周时间，主要难点在于：

1. **插件化架构设计** - 需要定义清晰的接口，同时保持灵活性
2. **跨平台兼容** - iOS 和 Android 的音频处理差异很大
3. **UI 细节** - 封面旋转、歌词同步、渐变背景等动效需要仔细调试

Flutter 做跨端音乐播放器是个不错的选择，生态成熟，性能也好。如果你也想做一个，可以参考这个项目的架构。

源码地址：[GitHub](https://github.com/your-repo/music-player)

---

*本文使用 Hermes Agent 辅助生成*
