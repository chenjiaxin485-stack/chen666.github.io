# 音频文件目录

把 `.mp3` 音乐文件上传到这个目录（GitHub 网页：进入本目录 → Add file → Upload files）。

上传后，编辑 `static/music.html` 里的歌曲列表，加一行：

```js
{ title: "歌名", artist: "歌手", src: "audio/你的文件名.mp3" },
```

然后到 Actions 里手动 Run 一次 "build Gmeek"，音乐就会出现在音乐页了。

注意：单个文件不要超过 100MB（GitHub 限制）；请上传你有权使用的音乐文件。
