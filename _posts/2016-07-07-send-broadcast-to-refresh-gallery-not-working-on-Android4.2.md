---
layout: post
title: "解决Android4.2版本广播刷新图库无效"
tags: Android
comments: true
---

## 问题描述

Android图库在拍完照之后并不会自动刷新，导致无法看到自己拍的照片，这时需要手动去刷新一下图库，在网上找到的诸多文章里提到最多的是发送一个广播通知，类似如下： 刷新整个sd卡
{% highlight java %}
sendBroadcast(new Intent(Intent.ACTION_MEDIA_MOUNTED, Uri.parse("file://"+ Environment.getExternalStorageDirectory())));
{% endhighlight java %}
或者刷新某个文件
{% highlight java %}
sendBroadcast(new Intent(Intent.ACTION_MEDIA_SCANNER_SCAN_FILE, Uri.fromFile(new File("/sdcard/DCIM/Camera/xxxx.jpg"))););
{% endhighlight java %}
以上方法在4.4上确实没有问题，但在我最近测试的4.2版本上却没有效果。

## 解决方法

在寻找解决办法的过程中，有很多博文都提到了MediaScannerConnection，下面是我用MediScannerConnection来解决此问题的办法。
在Activity的onResume方法中调用scanCameraFolder方法，然后通过判断相片是否存在来选择是否进行扫描，如果不做判断直接扫描的话，图库中会出现黑图现象。
{% highlight java %}
public void takePhoto(){
	String FOLDER_PATH = Environment.getExternalStorageDirectory()+"/DCIM/Camera/";
	String FILE_PATH = FOLDER_PATH+DateUtil.getCurrentDateText()+".jpg";
	File photoFile = new File(FILE_PATH);
							
	Intent intent = new Intent();
	Uri uri = Uri.fromFile(photoFile);
	intent.putExtra(MediaStore.EXTRA_OUTPUT, uri);
	// 指定开启系统相机的Action
	intent.setAction(MediaStore.ACTION_IMAGE_CAPTURE);
	startActivity(intent);
}


public void scanCameraFolder(){
	if(photoFile!=null&&photoFile.exists()){
	   //如果拍照文件存在才扫描，否则会出现黑图。
	   MediaScannerConnection.scanFile(this, new String[]{photoFile.getAbsolutePath()}, new String[]{"image/jpeg"}, new MediaScannerConnection.OnScanCompletedListener() {
		   @Override
		   public void onScanCompleted(String arg0, Uri arg1) {
			   Logger.d("onScanCompleted");
		   }
	   });
	}
}
{% endhighlight java %}