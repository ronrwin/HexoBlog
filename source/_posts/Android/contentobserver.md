---
title: ContentObserver监听媒体库变化
date: 2016-06-21 14:26:48
tags:
- Android
---
在Android中，我们可以使用ContentObserver监听ContentProvider数据的变化。

## 实例
```java
mPhotoObserver = new PhotoObserver(new Handler());
// 注册观察者
getContentResolver().registerContentObserver(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, true, mPhotoObserver);
  
  
// 监听图片变化
class PhotoObserver extends ContentObserver {
        public PhotoObserver(Handler handler) {
            super(handler);
        }
 
        @Override
        public void onChange(boolean selfChange) {
 
        }
 
        @Override
        public void onChange(boolean selfChange, Uri uri) {
            Log.d("image onChange", selfChange + ", " + uri);
 
            //查询出第一条记录
            Cursor c = getContentResolver().query(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, null, null, null, "_id DESC LIMIT 1");
//                Cursor c = getContentResolver().query(uri, null, null, null, "_id DESC LIMIT 1");
            if (c.moveToNext()) {
                int imageId = c.getInt((c.getColumnIndex(MediaStore.Images.Media._ID)));
				String title = c.getString(c.getColumnIndex(MediaStore.Files.FileColumns.DATA));
                Bitmap bm = MediaStore.Images.Thumbnails.getThumbnail(getContentResolver(),
                        imageId, MediaStore.Images.Thumbnails.MINI_KIND, null);
 
                Toast toast = Toast.makeText(MainActivity.this, "刚才的IMG :" + title, Toast.LENGTH_SHORT);
                toast.setGravity(Gravity.CENTER, 20, 69);
                LinearLayout linearLayout = (LinearLayout) toast.getView();
                ImageView imageView = new ImageView(MainActivity.this);
                imageView.setImageBitmap(bm);
                linearLayout.addView(imageView, 1);
                toast.setView(linearLayout);
                toast.show();
            }
            c.close();
        }
    }
```
<!--more-->
```java
mVideoObserver = new VideoObserver(new Handler());
getContentResolver().registerContentObserver(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, true, mVideoObserver);
  
// 监听视频变化
class VideoObserver extends ContentObserver {
    public VideoObserver(Handler handler) {
        super(handler);
    }
 
    @Override
    public void onChange(boolean selfChange) {
    }
 
    @Override
    public void onChange(boolean selfChange, Uri uri) {
        Log.d("video onChange", selfChange + ", " + uri);
        //查询出第一条记录
        Cursor c = getContentResolver().query(MediaStore.Video.Media.EXTERNAL_CONTENT_URI, null, null, null, "_id DESC LIMIT 1");
        if (c.moveToNext()) {
            int imageId = c.getInt((c.getColumnIndex(MediaStore.Video.Media._ID)));
            Bitmap bm = MediaStore.Video.Thumbnails.getThumbnail(getContentResolver(),
                    imageId, MediaStore.Video.Thumbnails.MINI_KIND, null);
 
            Toast toast = Toast.makeText(MainActivity.this, "刚才的Video", Toast.LENGTH_LONG);
            toast.setGravity(Gravity.CENTER, 20, 69);
            LinearLayout linearLayout = (LinearLayout) toast.getView();
            ImageView imageView = new ImageView(MainActivity.this);
            imageView.setImageBitmap(bm);
            linearLayout.addView(imageView, 1);
            toast.setView(linearLayout);
            toast.show();
        }
        c.close();
    }
}
```
我们试下拍照的事件响应：
```java
D/image onChange: false, content://media/external/images/media
D/image onChange: false, content://media/external/images/media/46827?blocking=1&orig_id=46827&group_id=0
```
当数据发生变化的时候，会通知到onChange方法。在API 16以上，可支持返回变化的Uri对象，在此之前，不会返回具体变化的对象，需要自己去找出被改变的对象。
在上面的例子中，Handler其实是基本没起到作用的，直接传null也不会影响使用。


> 在使用ContentObserver监听时，会接收到多次的onChange事件，事件传回来的Uri的值也有所不同，因此我们还需要对Uri做过滤的工作，以确保Uri是正确的。
当我们执行删除操作时，接收到的事件日志如下：
```
D/image onChange: false, content://media/external/images/media
```
所以在删除时，没有返回具体的Uri对象。
