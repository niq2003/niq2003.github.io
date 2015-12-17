---
layout: post
title:  "善用车轮之图片预览"
date:   2015-12-17 15:39:24
categories: Android
excerpt: ViewPager 图片预览。
---

* content
{:toc}

前段时间项目需求做一个图片预览的功能，包括图片点击放大全屏预览，可双击放大，手势缩放，适用于单张图片和ViewPager轮播图。一番Github后，找到了一个比较适用的轮子：[GestureImageView](https://github.com/alexvasilkov/GestureViews) 经过一番改造，先上效果：

![viewpager-preview](http://7xooqg.com1.z0.glb.clouddn.com/android/image/viewpager-preview.gif)

直接上代码

继承GestureImageView，增加显示，消失过渡效果
```javascript
import android.animation.Animator;
import android.animation.PropertyValuesHolder;
import android.animation.ValueAnimator;
import android.content.Context;
import android.graphics.Canvas;
import android.graphics.Paint;
import android.graphics.Paint.Style;
import android.util.AttributeSet;
import android.view.animation.AccelerateDecelerateInterpolator;

import com.alexvasilkov.gestures.views.GestureImageView;

public class FadingGestureImageView extends GestureImageView {

    private static final int STATE_NORMAL = 0;
    private static final int STATE_ANIMATOR_IN = 1;
    private static final int STATE_ANIMATOR_OUT = 2;
    private final int mBgColor = 0xFF000000;
    private int mState = STATE_NORMAL;
    private boolean mTransformStart = false;
    private int mBgAlpha = 0;
    private Paint mPaint;
    private AnimationListener mAnimationListener;

    public FadingGestureImageView(Context context) {
        super(context);
        init();
    }

    public FadingGestureImageView(Context context, AttributeSet attrs) {
        super(context, attrs);
        init();
    }

    public FadingGestureImageView(Context context, AttributeSet attrs, int defStyle) {
        super(context, attrs, defStyle);
        init();
    }

    private void init() {
        mPaint = new Paint();
        mPaint.setColor(mBgColor);
        mPaint.setStyle(Style.FILL);
    }

    public void animationIn() {
        mState = STATE_ANIMATOR_IN;
        mTransformStart = true;
        invalidate();
    }

    public void animationOut() {
        mState = STATE_ANIMATOR_OUT;
        mTransformStart = true;
        invalidate();
    }

    @Override
    protected void onDraw(Canvas canvas) {
        if (getDrawable() == null) {
            return;
        }

        if (mState == STATE_ANIMATOR_IN || mState == STATE_ANIMATOR_OUT) {

            mPaint.setAlpha(mBgAlpha);
            canvas.drawPaint(mPaint);
            canvas.save();
            if (mTransformStart) {
                mTransformStart = false;
                startAnimation(mState);
            }
        } else {
            //当Transform In变化完成后，把背景改为黑色
            mPaint.setAlpha(255);
            canvas.drawPaint(mPaint);
            super.onDraw(canvas);
        }
    }

    private void startAnimation(final int state) {
        ValueAnimator valueAnimator = new ValueAnimator();
        valueAnimator.setDuration(300);
        valueAnimator.setInterpolator(new AccelerateDecelerateInterpolator());
        if (state == STATE_ANIMATOR_IN) {
            PropertyValuesHolder alphaHolder = PropertyValuesHolder.ofInt("alpha", 0, 255);
            valueAnimator.setValues(alphaHolder);
        } else {
            PropertyValuesHolder alphaHolder = PropertyValuesHolder.ofInt("alpha", 255, 0);
            valueAnimator.setValues(alphaHolder);
        }

        valueAnimator.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {
            @Override
            public synchronized void onAnimationUpdate(ValueAnimator animation) {
                mBgAlpha = (Integer) animation.getAnimatedValue("alpha");
                invalidate();
            }
        });
        valueAnimator.addListener(new ValueAnimator.AnimatorListener() {
            @Override
            public void onAnimationStart(Animator animation) {
                if (mAnimationListener != null) {
                    mAnimationListener.onAnimationStart(state);
                }
            }

            @Override
            public void onAnimationRepeat(Animator animation) {

            }

            @Override
            public void onAnimationEnd(Animator animation) {
                if (state == STATE_ANIMATOR_IN) {
                    mState = STATE_NORMAL;
                }
                if (mAnimationListener != null) {
                    mAnimationListener.onAnimationComplete(state);
                }
            }

            @Override
            public void onAnimationCancel(Animator animation) {

            }
        });
        valueAnimator.start();
    }

    public void setOnAnimationListener(AnimationListener listener) {
        mAnimationListener = listener;
    }

    public interface AnimationListener {
        void onAnimationStart(int mode);

        void onAnimationComplete(int mode);
    }

}
```

弹出的预览框，用Activity实现,里面的[ReboundViewPager](http://niq2003.github.io/2015/11/26/viewpager-preview/)可换成ViewPager。
```javascript
/**
 * 图片预览
 */
public class ImagePreviewActivity extends Activity {

    public static final String KEY_IMAGE_URLS = "key_preview_image_urls";
    public static final String KEY_IMAGE_POSITION = "key_preview_image_position";
    public static final String KEY_SHOW_INDICATOR = "key_show_image_indicator";
    public static final String KEY_SHOW_DRAG_TIP = "key_show_drag_tip";
    public static final int REQUESTCODE_IMAGE_PREVIEW = 1;

    private static int mOverscrollTranslation;

    private ReboundViewPager mViewPager;
    private ImagePreviewAdapter mAdatpter;
    private String[] mImageUrls;
    private int mPosition;
    private boolean mShowIndicator, mShowDragTip;
    private TextView mIndicatorView;
    private View mTipView;
    private FadingGestureImageView mCurrentIem;
    private boolean mPullRelease;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.image_preview_layout);
        getWindow().setLayout(WindowManager.LayoutParams.MATCH_PARENT, WindowManager.LayoutParams.MATCH_PARENT);
        init();
    }

    private void init() {
        mImageUrls = getIntent().getStringArrayExtra(KEY_IMAGE_URLS);
        if (mImageUrls == null) {
            UUtil.showShortToast(this, getString(R.string.empty_image_tip));
            return;
        }
        mOverscrollTranslation = UUtil.dip2px(this, 200);
        mPosition = getIntent().getIntExtra(KEY_IMAGE_POSITION, 0);
        mShowIndicator = getIntent().getBooleanExtra(KEY_SHOW_INDICATOR, true) && mImageUrls.length > 1;
        mShowDragTip = getIntent().getBooleanExtra(KEY_SHOW_DRAG_TIP, false);
        mViewPager = (ReboundViewPager) findViewById(R.id.vp_images);
        mViewPager.setOverScrollMode(View.OVER_SCROLL_NEVER);
        mViewPager.setOverscrollTranslation(mOverscrollTranslation);
        mIndicatorView = (TextView) findViewById(R.id.tv_pager_footer);
        mTipView = findViewById(R.id.view_tip);
        mAdatpter = new ImagePreviewAdapter(mViewPager, mImageUrls, null);
        mViewPager.setAdapter(mAdatpter);
        mViewPager.getViewTreeObserver().addOnPreDrawListener(new ViewTreeObserver.OnPreDrawListener() {
            @Override
            public boolean onPreDraw() {
                mViewPager.getViewTreeObserver().removeOnPreDrawListener(this);
                mViewPager.setCurrentItem(mPosition, false);
                fixCurrentItem();
                mCurrentIem.animationIn();
                return false;
            }
        });
//        mViewPager.post(new Runnable() {
//            @Override
//            public void run() {
//                mViewPager.setCurrentItem(mPosition, false);
//                fixCurrentItem();
//                mCurrentIem.animationIn();
//            }
//        });
        mViewPager.setOnPageChangeListener(mOnPageChangeListener);
        if (mShowDragTip) {
            mViewPager.setOnReboundListener(mOnReboundListener);
        }
        if (mShowIndicator) {
            if (mImageUrls.length == 0) {
                mIndicatorView.setText("0/0");
            } else {
                mIndicatorView.setText((mPosition + 1) + "/" + (mImageUrls.length));
            }
            mIndicatorView.setVisibility(View.VISIBLE);
        } else {
            mIndicatorView.setVisibility(View.GONE);
        }
    }

    private ReboundViewPager.OnReboundListener mOnReboundListener = new ReboundViewPager.OnReboundListener() {
        private float mDistance;
        @Override
        public void onPull(float deltaDistance) {
            mDistance = deltaDistance;
            if (mDistance > 0 && mViewPager.getCurrentItem() == mImageUrls.length - 1) {
                float x = mViewPager.getX() + mViewPager.getWidth() - mOverscrollTranslation * deltaDistance;
                mTipView.setX(x);
                mTipView.setVisibility(View.VISIBLE);
            }
        }

        @Override
        public void onRelease() {
            boolean releaseTodo = Math.abs(mDistance) >= 0.2;
            if (mDistance > 0 && releaseTodo) {
                fixCurrentItem();
                new Handler().postDelayed(new Runnable() {
                    @Override
                    public void run() {
                        mPullRelease = true;
                        finish();
                    }
                }, mViewPager.getOverscrollAnimationDuration());
            }
        }
    };

    private ViewPager.OnPageChangeListener mOnPageChangeListener = new ViewPager.OnPageChangeListener () {
        @Override
        public void onPageScrolled (int position, float positionOffset, int positionOffsetPixels) {
        }

        @Override
        public void onPageSelected(int position) {
            int picCount = mImageUrls.length;
            mIndicatorView.setText((position + 1) + "/" + (picCount));
        }

        @Override
        public void onPageScrollStateChanged (int state) {
        }
    };

    private FadingGestureImageView.AnimationListener mAnimationListener = new FadingGestureImageView.AnimationListener() {

        @Override
        public void onAnimationStart(int mode) {
            if (mode == 2) {
                mViewPager.setBackgroundColor(getResources().getColor(android.R.color.transparent));
            }
        }

        @Override
        public void onAnimationComplete(int mode) {
            if (mode == 1) {
                mViewPager.setBackgroundColor(getResources().getColor(R.color.c_000000));
            } else if (mode == 2) {
                if (mPullRelease) {
                    setResult(RESULT_OK, new Intent());
                }
                ImagePreviewActivity.super.finish();
                overridePendingTransition(0, 0);
            }
        }
    };

    @Override
    public void finish() {
        fixCurrentItem();
        mCurrentIem.animationOut();
    }

    private void fixCurrentItem() {
        if (mCurrentIem != mAdatpter.getPrimaryItem()) {
            mCurrentIem = mAdatpter.getPrimaryItem();
            mCurrentIem.setOnAnimationListener(mAnimationListener);
        }
    }

    @Override
    public void onBackPressed() {
        finish();
    }
  }
```
布局文件：image_preview_layout
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="match_parent">

    <com.metersbonwe.app.view.uview.ReboundViewPager
        android:id="@+id/vp_images"
        android:layout_width="match_parent"
        android:layout_height="match_parent" />

    <LinearLayout
        android:id="@+id/view_tip"
        android:layout_width="wrap_content"
        android:layout_height="match_parent"
        android:gravity="center_vertical"
        android:orientation="horizontal"
        android:padding="@dimen/d_7"
        android:layout_alignParentRight="true"
        android:visibility="gone">

        <ImageView
            android:layout_width="wrap_content"
            android:layout_height="wrap_content"
            android:src="@drawable/icon_home_pull"
            android:rotation="-90"/>

        <TextView
            android:layout_width="@dimen/t7"
            android:layout_height="wrap_content"
            android:text="@string/release_tip"
            android:textSize="@dimen/t7"
            android:textColor="@color/c6"
            android:layout_marginLeft="@dimen/d_7"/>

    </LinearLayout>

    <TextView
        android:id="@+id/tv_pager_footer"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:layout_alignParentBottom="true"
        android:layout_alignParentRight="true"
        android:layout_marginRight="28dp"
        android:layout_marginBottom="24dp"
        android:textSize="@dimen/t1"
        android:textColor="@color/c3"/>

</RelativeLayout>
```
```javascript
public class ImagePreviewAdapter extends RecyclePagerAdapter<ImagePreviewAdapter.ViewHolder> implements GestureController.OnGestureListener {

    private static final String TAG = ImagePreviewAdapter.class.getSimpleName();

    private final ViewPager mViewPager;
    private final String[] mUrls;
    private final OnSetupGestureViewListener mSetupListener;
    private Context mContext;
    private FadingGestureImageView mImageView;

    public ImagePreviewAdapter(ViewPager pager, String[] urls,
                               OnSetupGestureViewListener listener) {
        mViewPager = pager;
        mUrls = urls;
        mSetupListener = listener;
    }

    @Override
    public int getCount() {
        return mUrls.length;
    }

    @Override
    public ViewHolder onCreateViewHolder(@NonNull ViewGroup container) {
        mContext = container.getContext();
        ViewHolder holder = new ViewHolder(container);
        GestureControllerForPager controller = holder.image.getController();
        controller.enableScrollInViewPager(mViewPager);
        controller.getSettings().setFillViewport(true);
        controller.getSettings().setMaxZoom(5);
        controller.setOnGesturesListener(this);
        return holder;
    }

    @Override
    public void onBindViewHolder(@NonNull ViewHolder holder, int position) {
        if (mSetupListener != null) mSetupListener.onSetupGestureView(holder.image);
        holder.image.getController().resetState();
        String uri = mUrls[position];
        ULog.logd(TAG," onBindViewHolder uri = ",uri);
        String url = uri;
        if(ImageUtil.checkIsNetworkUri(uri)) {
            url = UUtil.getSoaThumUrl(mUrls[position], UConfig.screenWidth, UConfig.screenWidth);
            ImageLoader.getInstance().displayImage(url, holder.image);
        }else{
            Bitmap bitmap = BitmapUtil.getBitmapinSampleSizeFromPath
                    (mContext, url, UConfig.screenWidth, UConfig.screenWidth);
            ULog.logd(TAG, " onBindViewHolder uri = ", uri, " size = ", String.valueOf(UConfig.screenWidth));
            if(bitmap != null){
                ULog.logd(TAG, " onBindViewHolder uri = ", uri
                        , " width = ", String.valueOf(bitmap.getWidth())
                        ," height =", String.valueOf(bitmap.getHeight()));
            }
            holder.image.setImageBitmap(bitmap);
        }

    }

    @Override
    public void onDown(MotionEvent e) {}

    @Override
    public boolean onSingleTapUp(MotionEvent e) {
        return false;
    }

    @Override
    public boolean onSingleTapConfirmed(MotionEvent e) {
        ((Activity)mContext).finish();
        return false;
    }

    @Override
    public void onLongPress(MotionEvent e) {}

    @Override
    public boolean onDoubleTap(MotionEvent e) {
        return false;
    }


    static class ViewHolder extends RecyclePagerAdapter.ViewHolder {
        public final FadingGestureImageView image;

        public ViewHolder(ViewGroup container) {
            super(new FadingGestureImageView(container.getContext()));
            image = (FadingGestureImageView) itemView;
        }
    }

    public interface OnSetupGestureViewListener {
        void onSetupGestureView(GestureView view);
    }

    @Override
    public void setPrimaryItem(ViewGroup container, int position, Object object) {
        super.setPrimaryItem(container, position, object);
        mImageView = ((ViewHolder) object).image;
    }

    public FadingGestureImageView getPrimaryItem()  {
        return mImageView;
    }
}
```
最后将ImagePreViewActivity的theme设置成dialog的样式
```xml
<activity
    android:name="ImagePreviewActivity"
    android:screenOrientation="portrait"
    android:theme="@style/dialogStyle" />
```
```xml
<style name="dialogStyle" parent="@android:style/Theme.Dialog">
    <item name="android:windowFrame">@android:color/transparent</item>
    <item name="android:windowIsFloating">true</item>
    <item name="android:windowIsTranslucent">true</item>
    <item name="android:windowNoTitle">true</item>
    <item name="android:windowBackground">@android:color/transparent</item>
    <item name="android:windowAnimationStyle">@null</item>
</style>
```
第一次贴代码，有些乱,凑合着看～
