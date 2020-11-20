---
title: Android 10 设置铃声失败
author: 小飞
date: 2020-03-21 22:49:58
tags:
---

## 前言
公司App有设置铃声的需求，通常在Android中设置铃声的方式如下：
```java
// 省略一些代码
Uri uri = MediaStore.Audio.Media.getContentUriForPath(ringtoneFile.getAbsolutePath());
        // 铃声通过contentvaues插入到数据库
final Uri newUri = getContentResolver().insert(uri, content);
RingtoneManager.setActualDefaultRingtoneUri(getApplicationContext(),
        RingtoneManager.TYPE_RINGTONE, newUri);
```

<!-- more -->

为了保证每次都能设置成功，需要在设置之前删除旧的铃声，但是在Android 10设备上调试的时候发现，调用`getContentResolver().delete()`的时候，铃声设置不成功，最后发现这行代码在android 10的设备上会删除媒体文件，而在之前的版本仅仅是删除数据库中的记录，以下是 Android 10 和之前的代码：     

Android 10:
```java
...
if (mediaType == FileColumns.MEDIA_TYPE_AUDIO) {
    if (!helper.mInternal) {
        // 删除磁盘中的源文件
        deleteIfAllowed(uri, data);
        MediaDocumentsProvider.onMediaStoreDelete(getContext(),
                volumeName, FileColumns.MEDIA_TYPE_AUDIO, id);

        idvalue[0] = String.valueOf(id);
        db.delete("audio_genres_map", "audio_id=?", idvalue);
        // for each playlist that the item appears in, move
        // all the items behind it forward by one
        Cursor cc = db.query("audio_playlists_map",
                    sPlaylistIdPlayOrder,
                    "audio_id=?", idvalue, null, null, null);
        try {
            while (cc.moveToNext()) {
                long playlistId = cc.getLong(0);
                playlistvalues[0] = String.valueOf(playlistId);
                playlistvalues[1] = String.valueOf(cc.getInt(1));
                int rowsChanged = db.executeSql("UPDATE audio_playlists_map" +
                        " SET play_order=play_order-1" +
                        " WHERE playlist_id=? AND play_order>?",
                        playlistvalues);

                if (rowsChanged > 0) {
                    updatePlaylistDateModifiedToNow(db, playlistId);
                }
            }
            db.delete("audio_playlists_map", "audio_id=?", idvalue);
        } finally {
            IoUtils.closeQuietly(cc);
        }
    }
}
...
```
上面的`deleteIfAllowed()`方法就是删除磁盘中的文件。     

而在Android 9 只删除数据库中的记录，并不删除媒体文件
```java
if (mediaType == FileColumns.MEDIA_TYPE_AUDIO) {
    if (!database.mInternal) {
        MediaDocumentsProvider.onMediaStoreDelete(getContext(),
                volumeName, FileColumns.MEDIA_TYPE_AUDIO, id);
        idvalue[0] = String.valueOf(id);
        database.mNumDeletes += 2; // also count the one below
        db.delete("audio_genres_map", "audio_id=?", idvalue);
        // for each playlist that the item appears in, move
        // all the items behind it forward by one
        Cursor cc = db.query("audio_playlists_map",
                    sPlaylistIdPlayOrder,
                    "audio_id=?", idvalue, null, null, null);
        try {
            while (cc.moveToNext()) {
                playlistvalues[0] = "" + cc.getLong(0);
                playlistvalues[1] = "" + cc.getInt(1);
                database.mNumUpdates++;
                db.execSQL("UPDATE audio_playlists_map" +
                        " SET play_order=play_order-1" +
                        " WHERE playlist_id=? AND play_order>?",
                        playlistvalues);
            }
            db.delete("audio_playlists_map", "audio_id=?", idvalue);
        } finally {
            IoUtils.closeQuietly(cc);
        }
    }
}
```