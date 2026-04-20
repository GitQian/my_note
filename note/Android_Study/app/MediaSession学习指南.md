# Android MediaSession 框架学习指南

## 1. 什么是 MediaSession？

`MediaSession` 是 Android 提供的一套标准媒体交互框架。它的核心目的是**解耦“播放器”与“控制器”**。

*   **播放端 (MediaSession)**：音乐 App、视频 App（如网易云音乐）。它负责告诉系统：“我正在播放什么，我的进度是多少，我支持什么操作（暂停/切歌）”。
*   **控制端 (MediaController)**：系统锁屏、蓝牙耳机、智能手表、或者你开发的“媒体监控 App”。它通过 `MediaController` 发送指令或监听状态。

---

## 2. 核心组件 (Core Components)

| 组件 | 作用 |
| :--- | :--- |
| **MediaSession** | 管理媒体状态。通常在 Service 中创建。 |
| **MediaController** | 连接到 Session 的工具，用于发送指令和接收回调。 |
| **PlaybackState** | 描述播放状态（播放中/暂停、当前进度、缓冲位置）。 |
| **MediaMetadata** | 描述媒体信息（歌名、歌手、封面、时长）。 |
| **MediaSession.Callback** | 播放端接收指令的回调（如 `onPlay()`, `onPause()`, `onSkipToNext()`）。 |

---

## 3. 实现流程：播放端 (Provider)

在实际开发中，你需要将 **播放引擎 (如 MediaPlayer)** 与 **MediaSession** 绑定在一起。

### 3.1 核心逻辑：联动原理
- **下行指令**：用户点击锁屏上的“播放”按钮 -> 系统触发 `MediaSession.Callback.onPlay()` -> 你在回调里调用 `mediaPlayer.start()`。
- **上行状态**：你的 `mediaPlayer` 准备好了 -> 你手动调用 `mediaSession.setPlaybackState()` 通知系统“现在变更为正在播放状态”。

### 3.2 联动代码示例

```java
public class MyPlayerService extends Service {
    private MediaPlayer mediaPlayer; // 真正的“发动机”
    private MediaSession mediaSession; // “远程控制接口”

    @Override
    public void onCreate() {
        super.onCreate();
        
        // 1. 初始化发动机
        mediaPlayer = new MediaPlayer();
        
        // 2. 初始化方向盘
        mediaSession = new MediaSession(this, "MusicService");
        
        // 3. 设置回调：当外界（锁屏、耳机）想控制你时，会触发这里
        mediaSession.setCallback(new MediaSession.Callback() {
            @Override
            public void onPlay() {
                // 外界让我播放，我命令发动机启动
                mediaPlayer.start();
                // 别忘了告诉外界，我真的动起来了
                updatePlaybackState(PlaybackState.STATE_PLAYING);
            }

            @Override
            public void onPause() {
                // 外界让我暂停，我命令发动机停止
                mediaPlayer.pause();
                updatePlaybackState(PlaybackState.STATE_PAUSED);
            }
        });
        
        mediaSession.setActive(true);
    }

    private void updatePlaybackState(int state) {
        PlaybackState playbackState = new PlaybackState.Builder()
            .setState(state, mediaPlayer.getCurrentPosition(), 1.0f)
            .setActions(PlaybackState.ACTION_PLAY | PlaybackState.ACTION_PAUSE)
            .build();
        mediaSession.setPlaybackState(playbackState);
    }
}
```

---

## 4. 实现流程：控制端/监控端 (Controller)

这是你最关心的部分：**如何获取当前正在播放的 App 信息**。

### 4.1 获取 MediaSessionManager
要获取系统中所有活跃的 Session，需要 `MediaSessionManager`。

```java
MediaSessionManager sessionManager = (MediaSessionManager) getSystemService(Context.MEDIA_SESSION_SERVICE);

// 需要一个 ComponentName，指向你的 NotificationListenerService (见下文)
ComponentName componentName = new ComponentName(this, MyNotificationListenerService.class);

// 获取当前所有活跃的控制器
List<MediaController> controllers = sessionManager.getActiveSessions(componentName);

for (MediaController controller : controllers) {
    // 获取包名（知道是哪个 App 在播放）
    String pkgName = controller.getPackageName(); 
    
    // 获取歌名、歌手
    MediaMetadata metadata = controller.getMetadata();
    if (metadata != null) {
        String title = metadata.getString(MediaMetadata.METADATA_KEY_TITLE);
    }
    
    // 获取播放状态
    PlaybackState state = controller.getPlaybackState();
    
    // 发送指令（比如暂停它）
    controller.getTransportControls().pause();
}
```

---

## 5. 关键权限：NotificationListenerService

在 Android 中，普通 App 是无法直接获取其他 App 的 `MediaSession` 的。你必须通过 **“通知监听服务”** 来获得授权。

1.  **在 AndroidManifest.xml 中声明**:
    ```xml
    <service android:name=".MyNotificationListenerService"
             android:permission="android.permission.BIND_NOTIFICATION_LISTENER_SERVICE">
        <intent-filter>
            <action android:name="android.service.notification.NotificationListenerService" />
        </intent-filter>
    </service>
    ```

2.  **引导用户开启权限**:
    你需要在 App 中引导用户跳转到设置页面手动开启“通知使用权”。
    ```java
    startActivity(new Intent("android.settings.ACTION_NOTIFICATION_LISTENER_SETTINGS"));
    ```

---

---

## 6. 现代化的实现：Jetpack Media3 (推荐)

在现代开发中，Google 将 `ExoPlayer` 和 `MediaSession` 统一到了 **Media3** 库中。

### 6.1 为什么用 Media3？
以往你需要手动写大量的代码来让 `MediaPlayer` 和 `MediaSession` 同步，而 Media3 实现了**自动化**：
- 当 ExoPlayer 开始播放，MediaSession 自动变为 `STATE_PLAYING`。
- 当 ExoPlayer 获取到视频封面，MediaSession 自动更新 `Metadata`。

### 6.2 Media3 核心代码示例

```java
// 1. 创建播放器 (ExoPlayer 是 Media3 的默认实现)
ExoPlayer player = new ExoPlayer.Builder(context).build();

// 2. 创建 MediaSession 并直接绑定播放器
// 这一行代码就完成了所有的联动逻辑！
MediaSession mediaSession = new MediaSession.Builder(context, player).build();

// 3. 准备播放
MediaItem mediaItem = MediaItem.fromUri(videoUri);
player.setMediaItem(mediaItem);
player.prepare();
player.play();
```

### 6.3 Media3 库引入 (build.gradle)
```gradle
dependencies {
    implementation "androidx.media3:media3-exoplayer:1.x.x"
    implementation "androidx.media3:media3-session:1.x.x"
    implementation "androidx.media3:media3-ui:1.x.x"
}
```

---

## 7. 学习建议

1.  **第一步**：先实现一个简单的 `NotificationListenerService`，并在 `onNotificationPosted` 中尝试获取 `MediaSessionManager.getActiveSessions()`。
2.  **第二步**：打印所有活跃 Session 的包名，观察当你打开网易云、抖音、YouTube 时，包名的变化。
3.  **第三步**：尝试通过 `controller.getTransportControls()` 实现一个悬浮窗媒体控制器。

---
*文档生成于: 2026-04-17*
