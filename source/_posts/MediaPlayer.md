---
title: 使用MediaPlayer开发简单的在线音乐播放器
date: 2016-11-17 15:11:44
tags: [Android, 干货]
category: Android
---

由于工作内容的局限性，Service组件在平时并不经常用得上，本篇以一个简单的音乐播放器为例，记录开发过程，以熟悉Service组件的使用，以免生疏。
本篇中所用到的音乐数据及音乐播放地址皆以爬虫实时获取，只用于学习交流所用。

<!-- more -->
本篇核心在于温习Service组件的使用，爬取音乐数据及列表展示不在此赘述，详情可查看篇尾项目地址

#### 定义Service用于编写所有的音乐播放控制及监听逻辑
首先我们定义一个MusicPlayService
```java
public class MusicPlayerService extends Service {
    private MusicInfo currentMusic = null;
    private int playState = 0; //0stop 1playing 2paused -1加载中

    @Override
    public IBinder onBind(Intent intent) {
        return new MusicBinder();
    }

    public class MusicBinder extends Binder {
        public void start(MusicInfo musicInfo) {
            currentMusic = musicInfo;
            Toast.makeText(MusicPlayerService.this, "play", Toast.LENGTH_SHORT).show();
        }
        public void pauseOrPlay(){
            Toast.makeText(MusicPlayerService.this, "pauseOrPlay", Toast.LENGTH_SHORT).show();
        }
        public void stop(){
            Toast.makeText(MusicPlayerService.this, "play", Toast.LENGTH_SHORT).show();
        }
    }
}
```
在Service中定义一个内部类继承自Binder用于onBind方法返回，因为是内部类，所以该类的方法可以与Service交互操作。
onBind方法在多次调用bindService时只会调用一次，除第一次外，再次调用bindService时会从缓存中获取第一次获取到的binder，也就是说我们不用担心在onBind方法中直接new对象会造成多次bind操作new多个binder的情况。

#### 在Activity中开启Service并BindService
我们定义一个Activity来模拟音乐播放界面或其他可操作的音乐播放界面
```java
public class MusicPlayActivity extends AppCompatActivity {
    private ServiceConnection serviceConnection;
    private MusicPlayerService.MusicBinder musicBinder;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_music_play);
        startService(new Intent(this, MusicPlayerService.class));
        serviceConnection = new ServiceConnection() {
            @Override
            public void onServiceConnected(ComponentName name, IBinder service) {
                musicBinder = (MusicPlayerService.MusicBinder) service;
            }
            @Override
            public void onServiceDisconnected(ComponentName name) { }
        };
        bindService(new Intent(this, MusicPlayerService.class), serviceConnection, AppCompatActivity.BIND_AUTO_CREATE);
    }

    public void start(View view) {
        musicBinder.start(null);
    }
    public void pauseOrPlay(View view) {
        musicBinder.pauseOrPlay();
    }
    public void stop(View view) {
       musicBinder.stop();
    }
    @Override
    protected void onDestroy() {
        super.onDestroy();
        unbindService(serviceConnection);
    }
}
```
这里我们首先调用了startService来激活Service，接下来使用bindService和unbindService来进行Service的绑定和解绑
如果在不使用startService的情况下只是用bind和unbind来操作Service的话，Service会在所有bind都被unbind的情况下自动销毁的，所以在这里先使用了startService。
使用了StartService激活的Service只有在调用stopService或者service自身stopself的情况下才会被正常结束，这样就能实现结束Activity之后仍能在后台播放音乐了。

通过bindservice我们在onServiceConnected回调方法中能够拿到在service的onbind方法中放回的Ibinder，由于我们返回的是MusicBinder内部类，所以可以直接强转得到musicbinder对象，
通过此对象我们就可以和musicService进行通信了，也就是说我们可以通过调用musicBinder的播放暂停停止方法来通知Service进行相应的音乐控制操作。

#### 在Service中通过MediaPlayer实现音乐的播放控制操作
我们需要在Service初始化的时候同时创建一个Mediaplayer对象让Service在运行期间播放音乐时使用，所以在oncreate中加入以下代码
```java
    @Override
    public void onCreate() {
        super.onCreate();
        createMediaPlayer();
    }

    private void createMediaPlayer() {
        if (mediaPlayer == null) {
            mediaPlayer = new MediaPlayer();
            mediaPlayer.setAudioStreamType(AudioManager.STREAM_MUSIC);
            mediaPlayer.setOnPreparedListener(new MediaPlayer.OnPreparedListener() {
                @Override
                public void onPrepared(MediaPlayer mp) {
                    mp.start();
                }
            });
        }
    }
```
接下来在MusicBinder中用MediaPlayer实现对音乐播放控制
```java 
    public class MusicBinder extends Binder {
        public void start(MusicInfo musicInfo) {
            currentMusic = musicInfo;
            if (mediaPlayer == null) {
                createMediaPlayer();
            }
            mediaPlayer.reset();
            try {
                mediaPlayer.setDataSource(musicInfo.url);
                mediaPlayer.prepareAsync();
            } catch (Exception e) {
                e.printStackTrace();
            }
        }

        public void pauseOrPlay() {
            Toast.makeText(MusicPlayerService.this, "pauseOrPlay", Toast.LENGTH_SHORT).show();
            if (mediaPlayer==null){
                createMediaPlayer();
            }
            mediaPlayer.pause();
//       or     mediaPlayer.start();
        }

        public void stop() {
            Toast.makeText(MusicPlayerService.this, "play", Toast.LENGTH_SHORT).show();
            if (mediaPlayer==null){
                createMediaPlayer();
            }
            mediaPlayer.stop();
        }
    }
```
上面代码只是大致示意了音乐播放暂停停止的操作，篇幅限制，这里不再赘述
到此为止对音乐的简单播放停止控制已经完成了。

#### 新增播放进度监听
MediaPlayer并没有为我们提供监听播放进度的api，所以我们要实现进度监听只能通过定时查询播放进度的方式来进行。
我们使用RXJava来定义一个定时器，当然你也可以使用Thread或者其他方式
```java
    override fun onCreate() {
        super.onCreate()
        Observable.interval(1, TimeUnit.SECONDS)
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe {
                    if (!(mediaPlayer?.isPlaying ?: false)) return@subscribe
                    val duration = mediaPlayer?.duration ?: 0
                    val currentPosition = mediaPlayer?.currentPosition ?: 0
                    if (binder.currentPosition != currentPosition || binder.duration != duration) {
                        binder.currentPosition = currentPosition
                        binder.duration = duration
                        if (binder.isBinderAlive) {
                            for (callBack in binder.callBackList) {
                                callBack.onProgress(currentPosition, duration)
                            }
                        }
                    }
                };
```
这个示例项目其实是用kotlin来写的，上面几段代码都是从kotlin又翻译回java的，这段实在不想用java来表示了，因为又长又丑
简单说一下吧，这里定义了一个1秒触发一次的定时器，并在主线程中回调，判断mediaplayer的播放状态，若是正在播放，则从mediaplayer中获取正在播放的音乐的最大长度和当前播放位置。
binder是维护的另外一个容器，用来存储播放器的进度以及注册的所有回调函数，然后遍历binder中注册的回调进行invoke。

完工。

完整项目代码地址：https://github.com/NightFarmer/MusicPlayer
此项目为kotlin项目，人生苦短，我用kotlin