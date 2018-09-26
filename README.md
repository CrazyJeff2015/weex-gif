# weex-gif
weex解决安卓[android]下gif图不动的问题 


> 写在前面，作为一个传统网页前端，分享开发weex三端应用中遇到问题的解决方案，可能对安卓开发者，经验老到的weex开发者本文毫无作用，无需细看。

weex打包出来的安卓包图片的实现需要自己用ImageAdapter去呈现，对于我这样一个写了N年网页前端的人来说，java还有安卓原生实在是看不懂。查询了N多地方、百度、翻墙、github等各式论坛，关于weex 解决gif图的方案少之又少，能找到的文章要么就是2年前，1年前的，很多包资源都找不到，有些是一句话带过。

踩坑踩了7-8天之后，抛弃了Picasso(动图不动)、Fresco、GifImage(自己重写的Adapter)，决定使用简单好用的glide，关于glide，可上官网查看介绍。

一、引入android glide
-----------------
在项目的Gradle Scripts - build.gradle(Module:app) 这个文件下，加上glide的依赖；

      compile 'com.github.bumptech.glide:glide:3.7.0'



二、我们先建一个自己的class文件 我的项目是放到extend -> adapter 下，先建了一个CommentSDKWeexImageAdapter.java;

    package com.weex.demo.extend.adapter;//更换为你自己的路径
    
    import android.graphics.Bitmap;
    import android.graphics.drawable.Drawable;
    import android.net.Uri;
    import android.text.TextUtils;
    import android.widget.ImageView;
    
    import com.bumptech.glide.Glide;
    import com.bumptech.glide.request.animation.GlideAnimation;
    import com.bumptech.glide.request.target.SimpleTarget;
    import com.taobao.weex.WXEnvironment;
    import com.taobao.weex.WXSDKManager;
    import com.taobao.weex.adapter.IWXImgLoaderAdapter;
    import com.taobao.weex.common.WXImageStrategy;
    import com.taobao.weex.dom.WXImageQuality;
    
    public class CommentSDKWeeXImageAdapter implements IWXImgLoaderAdapter {
    
        public CommentSDKWeeXImageAdapter() {}
    
        @Override
        public void setImage(final String url, final ImageView view,
                             WXImageQuality quality, final WXImageStrategy strategy) {
    
            WXSDKManager.getInstance().postOnUiThread(new Runnable() {
    
                @Override
                public void run() {
                    if (view == null || view.getLayoutParams() == null) {
                        return;
                    }
                    if (TextUtils.isEmpty(url)) {
                        view.setImageBitmap(null);
                        return;
                    }
                    if (url.startsWith("file://")) {
                        Glide.with(WXEnvironment.getApplication()).load(url).into(view);
                        return;
                    }
                    String temp = url;
                    if (url.startsWith("//")) {
                        temp = "http:" + url;
                    }
                    Glide.with(WXEnvironment.getApplication()).load(temp).asBitmap().into(new WeeXImageTarget(strategy, url, view));
                }
            }, 0);
        }
    
        private class WeeXImageTarget extends SimpleTarget<Bitmap> {
    
            private WXImageStrategy mWXImageStrategy;
            private String mUrl;
            private ImageView mImageView;
    
            WeeXImageTarget(WXImageStrategy strategy, String url, ImageView imageView) {
                mWXImageStrategy = strategy;
                mUrl = url;
                mImageView = imageView;
            }
    
            @Override
            public void onResourceReady(Bitmap resource, GlideAnimation<? super Bitmap> glideAnimation) {
                mImageView.setImageBitmap(resource);
            }
    
            @Override
            public void onLoadFailed(Exception e, Drawable errorDrawable) {}
        }
    }

三、APP启动的时候 注册这个class
--------------------

我们在JAVA这个目录下找到WXApplication.java这个文件，
先import这个class

    import com.weex.demo.extend.adapter.CommentSDKWeeXImageAdapter; //注意路径要写对
    
然后再onCreate的函数里面初始化

    public class WXApplication extends Application {
    
      @Override
      public void onCreate() {
        super.onCreate();
        WXSDKEngine.addCustomOptions("appName", "WXSample");
        WXSDKEngine.addCustomOptions("appGroup", "WXApp");
        WXSDKEngine.initialize(this,
            new InitConfig.Builder().setImgAdapter(new CommentSDKWeeXImageAdapter()).build()//就是这个位置
        );
        try {
          WXSDKEngine.registerModule("event", WXEventModule.class);
          WXSDKEngine.registerComponent("gsyWeb", GSYWeb.class);
        } catch (WXException e) {
          e.printStackTrace();
        }
        AppConfig.init(this);
        WeexPluginContainer.loadAll(this);
    
        StrictMode.VmPolicy.Builder builder = new StrictMode.VmPolicy.Builder();
        StrictMode.setVmPolicy(builder.build());
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.JELLY_BEAN_MR2) {
          builder.detectFileUriExposure();
        }
      }
    }

然后构建一下，你就发现你的gif图动了起来啦。自此，困扰我一个多星期的问题终于得到解决，希望这个文章能帮到和我一样遇到问题的人。

