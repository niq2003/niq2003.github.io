---
layout: post
title:  "善用轮子之ViewPager继续滑动查看XXX"
date:   2015-12-18 12:50:25 +0800
categories: Android
excerpt: ViewPager 继续滑动 pull to next。
---

* content
{:toc}
最近继续滑动查看图文详情几乎成了电商应用的商品详情页面的必备效果，一番google，结合网友提供的思路和部分代码，基本实现了需求效果。

![ReboundViewPager](http://7xooqg.com1.z0.glb.clouddn.com/android/image/overscrolled-viewpager.gif)

直接上代码

    public class ReboundViewPager extends ViewPager {
        /**
         * maximum z distance to translate child view
         */
        final static int DEFAULT_OVERSCROLL_TRANSLATION = 150;
        /**
         * duration of overscroll animation in ms
         */
        final private static int DEFAULT_OVERSCROLL_ANIMATION_DURATION = 400;
        private final static int INVALID_POINTER_ID = -1;
        private OnReboundListener mOnReboundListener;
        /**
         * @author renard, extended by Piotr Zawadzki
         */
        private class OverscrollEffect {
            private float mOverscroll;
            private Animator mAnimator;

            /**
             * @param deltaDistance [0..1] 0->no overscroll, 1>full overscroll
             */
            public void setPull(final float deltaDistance) {
                mOverscroll = deltaDistance;
                invalidateVisibleChilds(mLastPosition);
                if (mOnReboundListener != null && isOverscrolling()) {
                    mOnReboundListener.onPull(deltaDistance);
                }
            }

            /**
             * called when finger is released. starts to animate back to default position
             */
            private void onRelease() {
                if (mAnimator != null && mAnimator.isRunning()) {
                    mAnimator.addListener(new Animator.AnimatorListener() {

                        @Override
                        public void onAnimationStart(Animator animation) {
                        }

                        @Override
                        public void onAnimationRepeat(Animator animation) {
                        }

                        @Override
                        public void onAnimationEnd(Animator animation) {
                            startAnimation(0);
                        }

                        @Override
                        public void onAnimationCancel(Animator animation) {
                        }
                    });
                    mAnimator.cancel();
                } else {
                    startAnimation(0);
                }
                if (mOnReboundListener != null && isOverscrolling()) {
                    mOnReboundListener.onRelease();
                }
            }

            private void startAnimation(final float target) {
                mAnimator = ObjectAnimator.ofFloat(this, "pull", mOverscroll, target);
                mAnimator.setInterpolator(new DecelerateInterpolator());
                final float scale = Math.abs(target - mOverscroll);
                mAnimator.setDuration((long) (mOverscrollAnimationDuration * scale));
                mAnimator.start();
            }

            private boolean isOverscrolling() {
                if (mScrollPosition == 0 && mOverscroll < 0) {
                    return true;
                }
                final boolean isLast = (getAdapter().getCount() - 1) == mScrollPosition;
                if (isLast && mOverscroll > 0) {
                    return true;
                }
                return false;
            }

        }

        final private OverscrollEffect mOverscrollEffect = new OverscrollEffect();
        final private Camera mCamera = new Camera();

        private OnPageChangeListener mScrollListener;
        private float mLastMotionX;
        private int mActivePointerId;
        private int mScrollPosition;
        private float mScrollPositionOffset;
        final private int mTouchSlop;

        private float mOverscrollTranslation;
        private int mOverscrollAnimationDuration;

        public ReboundViewPager(Context context, AttributeSet attrs) {
            super(context, attrs);
            setStaticTransformationsEnabled(true);
            final ViewConfiguration configuration = ViewConfiguration.get(context);
            mTouchSlop = ViewConfigurationCompat.getScaledPagingTouchSlop(configuration);
            super.setOnPageChangeListener(new MyOnPageChangeListener());
            init(attrs);
        }

        private void init(AttributeSet attrs) {
            TypedArray a = getContext().obtainStyledAttributes(attrs, R.styleable.ReboundViewPager);
            mOverscrollTranslation = a.getDimension(R.styleable.ReboundViewPager_overscroll_translation, DEFAULT_OVERSCROLL_TRANSLATION);
            mOverscrollAnimationDuration = a.getInt(R.styleable.ReboundViewPager_overscroll_animation_duration, DEFAULT_OVERSCROLL_ANIMATION_DURATION);
            a.recycle();
        }

        public int getOverscrollAnimationDuration() {
            return mOverscrollAnimationDuration;
        }

        public void setOverscrollAnimationDuration(int mOverscrollAnimationDuration) {
            this.mOverscrollAnimationDuration = mOverscrollAnimationDuration;
        }

        public float getOverscrollTranslation() {
            return mOverscrollTranslation;
        }

        public void setOverscrollTranslation(int mOverscrollTranslation) {
            this.mOverscrollTranslation = mOverscrollTranslation;
        }

        @Override
        public void setOnPageChangeListener(OnPageChangeListener listener) {
            mScrollListener = listener;
        };

        private void invalidateVisibleChilds(final int position) {
            for (int i = 0; i < getChildCount(); i++) {
                getChildAt(i).invalidate();

            }
            //this.invalidate();
            // final View child = getChildAt(position);
            // final View previous = getChildAt(position - 1);
            // final View next = getChildAt(position + 1);
            // if (child != null) {
            // child.invalidate();
            // }
            // if (previous != null) {
            // previous.invalidate();
            // }
            // if (next != null) {
            // next.invalidate();
            // }
        }

        private int mLastPosition = 0;

        private class MyOnPageChangeListener implements OnPageChangeListener {

            @Override
            public void onPageScrolled(int position, float positionOffset, int positionOffsetPixels) {
                if (mScrollListener != null) {
                    mScrollListener.onPageScrolled(position, positionOffset, positionOffsetPixels);
                }
                mScrollPosition = position;
                mScrollPositionOffset = positionOffset;
                mLastPosition = position;
                invalidateVisibleChilds(position);
            }

            @Override
            public void onPageSelected(int position) {

                if (mScrollListener != null) {
                    mScrollListener.onPageSelected(position);
                }
            }

            @Override
            public void onPageScrollStateChanged(final int state) {

                if (mScrollListener != null) {
                    mScrollListener.onPageScrollStateChanged(state);
                }
                if (state == SCROLL_STATE_IDLE) {
                    mScrollPositionOffset = 0;
                }
            }
        }

        @Override
        public boolean onInterceptTouchEvent(MotionEvent ev) {
            try {
                final int action = ev.getAction() & MotionEventCompat.ACTION_MASK;
                switch (action) {
                    case MotionEvent.ACTION_DOWN: {
                        mLastMotionX = ev.getX();
                        mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
                        break;
                    }
                    case MotionEventCompat.ACTION_POINTER_DOWN: {
                        final int index = MotionEventCompat.getActionIndex(ev);
                        final float x = MotionEventCompat.getX(ev, index);
                        mLastMotionX = x;
                        mActivePointerId = MotionEventCompat.getPointerId(ev, index);
                        break;
                    }
                }
                return super.onInterceptTouchEvent(ev);
            } catch (IllegalArgumentException e) {
                e.printStackTrace();
                return false;
            } catch (ArrayIndexOutOfBoundsException e) {
                e.printStackTrace();
                return false;
            }
        }

        @Override
        public boolean dispatchTouchEvent(MotionEvent ev) {
            try {
                return super.dispatchTouchEvent(ev);
            } catch (IllegalArgumentException e) {
                e.printStackTrace();
                return false;
            } catch (ArrayIndexOutOfBoundsException e) {
                e.printStackTrace();
                return false;
            }
        }

        @Override
        public boolean onTouchEvent(MotionEvent ev) {
            boolean callSuper = false;

            final int action = ev.getAction();
            switch (action) {
                case MotionEvent.ACTION_DOWN: {
                    callSuper = true;
                    mLastMotionX = ev.getX();
                    mActivePointerId = MotionEventCompat.getPointerId(ev, 0);
                    break;
                }
                case MotionEventCompat.ACTION_POINTER_DOWN: {
                    callSuper = true;
                    final int index = MotionEventCompat.getActionIndex(ev);
                    final float x = MotionEventCompat.getX(ev, index);
                    mLastMotionX = x;
                    mActivePointerId = MotionEventCompat.getPointerId(ev, index);
                    break;
                }
                case MotionEvent.ACTION_MOVE: {
                    if (mActivePointerId != INVALID_POINTER_ID) {
                        // Scroll to follow the motion event
                        final int activePointerIndex = MotionEventCompat.findPointerIndex(ev, mActivePointerId);
                        final float x = MotionEventCompat.getX(ev, activePointerIndex);
                        final float deltaX = mLastMotionX - x;
                        final float oldScrollX = getScrollX();
                        final int width = getWidth();
                        final int widthWithMargin = width + getPageMargin();
                        final int lastItemIndex = getAdapter().getCount() - 1;
                        final int currentItemIndex = getCurrentItem();
                        final float leftBound = Math.max(0, (currentItemIndex - 1) * widthWithMargin);
                        final float rightBound = Math.min(currentItemIndex + 1, lastItemIndex) * widthWithMargin;
                        final float scrollX = oldScrollX + deltaX;
                        if (mScrollPositionOffset == 0) {
                            if (scrollX < leftBound) {
                                if (leftBound == 0) {
                                    final float over = deltaX + mTouchSlop;
                                    mOverscrollEffect.setPull(over / width);
                                }
                            } else if (scrollX > rightBound) {
                                if (rightBound == lastItemIndex * widthWithMargin) {
                                    final float over = scrollX - rightBound - mTouchSlop;
                                    mOverscrollEffect.setPull(over / width);
                                }
                            }
                        } else {
                            mLastMotionX = x;
                        }
                    } else {
                        mOverscrollEffect.onRelease();
                    }
                    break;
                }
                case MotionEvent.ACTION_UP:
                case MotionEvent.ACTION_CANCEL: {
                    callSuper = true;
                    mActivePointerId = INVALID_POINTER_ID;
                    mOverscrollEffect.onRelease();
                    break;
                }
                case MotionEvent.ACTION_POINTER_UP: {
                    final int pointerIndex = (ev.getAction() & MotionEvent.ACTION_POINTER_INDEX_MASK) >> MotionEvent.ACTION_POINTER_INDEX_SHIFT;
                    final int pointerId = MotionEventCompat.getPointerId(ev, pointerIndex);
                    if (pointerId == mActivePointerId) {
                        // This was our active pointer going up. Choose a new
                        // active pointer and adjust accordingly.
                        final int newPointerIndex = pointerIndex == 0 ? 1 : 0;
                        mLastMotionX = ev.getX(newPointerIndex);
                        mActivePointerId = MotionEventCompat.getPointerId(ev, newPointerIndex);
                        callSuper = true;
                    }
                    break;
                }
            }

            if (mOverscrollEffect.isOverscrolling() && !callSuper) {
                return true;
            } else {
                return super.onTouchEvent(ev);
            }
        }

        @Override
        protected boolean getChildStaticTransformation(View child, Transformation t) {
            if (child.getWidth() == 0) {
                return false;
            }
            final int position = child.getLeft() / child.getWidth();
            final boolean isFirstOrLast = position == 0 || (position == getAdapter().getCount() - 1);
            if (mOverscrollEffect.isOverscrolling() && isFirstOrLast) {
                final float dx = getWidth() / 2;
                final int dy = getHeight() / 2;
                t.getMatrix().reset();
                final float translateX =(float) mOverscrollTranslation * (mOverscrollEffect.mOverscroll > 0 ? Math.min(mOverscrollEffect.mOverscroll, 1) : Math.max(mOverscrollEffect.mOverscroll, -1));
                mCamera.save();
                mCamera.translate(-translateX, 0, 0);
                mCamera.getMatrix(t.getMatrix());
                mCamera.restore();
                t.getMatrix().preTranslate(-dx, -dy);
                t.getMatrix().postTranslate(dx, dy);

                if (getChildCount() == 1) {
                    this.invalidate();
                } else {
                    child.invalidate();
                }
                return true;
            }
            return false;
        }

        public void setOnReboundListener(OnReboundListener onReboundListener) {
            this.mOnReboundListener = onReboundListener;
        }

        public interface OnReboundListener {
            public void onPull(float deltaDistance);
            public void onRelease();
        }
    }

attrs.xml中加入

    <declare-styleable name="ReboundViewPager">
        <!--determines the maximum amount of translation along the z-axis during the overscroll. Default is 150.-->
        <attr name="overscroll_translation" format="dimension" />
        <!-- Duration of animation when user releases the over scroll. Default is 400 ms. -->
        <attr name="overscroll_animation_duration" format="integer" />
    </declare-styleable>

控件的使用示例可参考另一篇陋文[善用轮子之图片预览](../../17/viewpager-preview)。
