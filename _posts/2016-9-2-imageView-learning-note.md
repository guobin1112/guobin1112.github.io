---
title : ImageView学习笔记
---

最近在研究图片加载库，趁机找时间学习下ImageView的源码，探究下它内部机制，同时学习下View的封装思想。

###概述
>  Displays an arbitrary image, such as an icon.  The ImageView class can load images from various sources (such as resources or content providers), takes care of computing its measurement from the image so that it can be used in any layout manager, and provides various display options such as scaling and tinting.

ImageView可以显示各种图片，比如icon图。它可以显示各种来源的图片，包括Resource，Content Provider等，显示的过程中会计算图片的尺寸从而适应任何布局管理器，并且提供各种显示可选项，比如缩放、着色等。（来自笔者蹩脚的翻译）

###XML属性
我们平时一般在XML声明ImageView，除了继承自View的属性，官方文档列出了以下属性，我们也主要从这些属性了解ImageView的功能


|XML属性      |  说明           |
|------------|--------------|
| android:adjustViewBounds |设置为true，可以让ImageView根据图片的宽宽比调整大小，使用该属性时不能固定ImageView的宽高，或者将宽高设置为match_parent  |
|android:baseline|设置文字的baseLine到ImageView顶部的距离，如果baselineAlignBottom被设置为true，则该属性会被覆盖，即该属性无效 |
|android:baselineAlignBottom|设置为true，baseLine则为ImageView的底部 |
|android:cropToPadding|设置为true，图像会在Padding属性的基础上进行裁剪 |
|android:maxHeight|设置ImageView的最大高度|
|android:maxWidth|设置ImageView的最大宽度|
|android:scaleType|设置ImageView的缩放模式|
|android:src|设置ImageView显示图片资源|
|android:tint|设置ImageView显示图片的着色颜色|
|android:tintMode|设置ImageView显示图片的着色模式|

###源码学习

接下来通过查看ImageView源码，分析下ImageView上面这些属性是怎么起效果的，像`adjustViewBound` 、`cropToPadding`这些属性经常用不太顺，从源码的角度了解这些属性是怎么影响ImageView最终效果。
#####图片显示
ImageView最大的功能就是显示图片，首先研究下图片是怎么在ImageView中显示的。
在平时使用中，我们可以通过以下两种方式设置图片：
- XML 设置src属性，可以设置drawable资源，eg. 
`android:src="@drawable/test"`  


- Java代码设置，可以设置调用imageView的`setImageBitmap`,  `setImageDrawable`, `setImageResource` , `setImageURI` 设置

注意如果使用 `setImageResource` , `setImageURI`  官方提示这两个方法是需要在UI线程中解析图片资源的，建议不要使用这两个方法加载太大的图，或者在其他线程中解析成Bitmap或者Drawable，避免画面卡顿。
>This does Bitmap reading and decoding on the UI thread, which can cause a latency hiccup. If that's a concern, consider using

接下来看一下在ImageView内部如何处理。
在构造函数中，XML中的src属性会被读取并转化成Drawable，并通过`setDrawable`设置图片。

    Drawable d = a.getDrawable(com.android.internal.R.styleable.ImageView_src);
    if (d != null) {
   setImageDrawable(d);
}

接着看下`setDrawable`代码，判断当前的Drawable与设置的Drawable是否不同，不同的话，把ImageView的成员变量`mResource` 和`mUri`   值置空，避免其它方式的值影响内容显示。然后调用`updateDrawable`这个更新Drawable的方法。如果图片宽高不同，需要重新布局`requestLayout`，最后通过`invalidate`刷新页面，因为最后图片的显示是通过`onDraw`刷新。

    public void setImageDrawable(Drawable drawable) {
    if (mDrawable != drawable) {
        mResource = 0;
        mUri = null;

        final int oldWidth = mDrawableWidth;
        final int oldHeight = mDrawableHeight;

        updateDrawable(drawable);

        if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
            requestLayout();
        }
        invalidate();
      }
    }

`updateDrawable`主要功能是将`ImageView`中`mDrawable`中替换，同时修改`mDrawable`属性，这里大部分是`Drawable`的知识，先不细究了，留意下最后三个方法``applyImageTint applyColorMod configureBounds``会影响图片着色和缩放，后面会继续提到。

    private void updateDrawable(Drawable d) {
        if (mDrawable != null) {
            mDrawable.setCallback(null);
            unscheduleDrawable(mDrawable);
        }

        mDrawable = d; 

        if (d != null) {
            d.setCallback(this);
            d.setLayoutDirection(getLayoutDirection());
            if (d.isStateful()) {
                d.setState(getDrawableState());
            }
            d.setVisible(getVisibility() == VISIBLE, true);
            d.setLevel(mLevel);
            mDrawableWidth = d.getIntrinsicWidth();
            mDrawableHeight = d.getIntrinsicHeight();
            applyImageTint();
            applyColorMod();
            configureBounds();
        } else {
            mDrawableWidth = mDrawableHeight = -1;
        }
    }

到这里我们可以了解到通过XML设置属性就是最终是修改了ImageView中的`mDrawable`变量，同时`setDrawable`方法也是通过`updateDrawable`来更新`mDrawable`。

接下来看一下`ImageView`的几个设置显示图片的方法。
`setImageBitmap`也是调用了`setImageDrawable`，同时使用`BitmapDrawable`可以减少中间过程中产生的drawable对象，可以大胆推测其他的`Drawable`最终会转化成`BitmapDrawable`用于显示。

    public void setImageBitmap(Bitmap bm) {
        setImageDrawable(new BitmapDrawable(mContext.getResources(), bm));
    }

然后看下`setImageResource`和`setImageUri`两个方法的思路是是一样，首先判断`mResource`或者`mUri`有值，然后将原有的`mDrawable`置空，`resolverUri`重新解析资源，这个方法最终也是调用了`updateDrawable`，更新图片显示，同样也需要考虑到重新布局的问题。

    public void setImageResource(int resId) {
         if (mUri != null || mResource != resId) {
             final int oldWidth = mDrawableWidth;
             final int oldHeight = mDrawableHeight;

             updateDrawable(null);
             mResource = resId;
             mUri = null;

             resolveUri();

             if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
                 requestLayout();
             }
             invalidate();
         }
    }


    public void setImageURI(Uri uri) {
        if (mResource != 0 ||
                (mUri != uri &&
                 (uri == null || mUri == null || !uri.equals(mUri)))) {
            updateDrawable(null);
            mResource = 0;
            mUri = uri;

            final int oldWidth = mDrawableWidth;
            final int oldHeight = mDrawableHeight;

            resolveUri();

            if (oldWidth != mDrawableWidth || oldHeight != mDrawableHeight) {
                requestLayout();
            }
            invalidate();
        }
    }

####图片缩放 ScaleType
详细大家都了解过`ImageView`的图片缩放类型，现在看一下是怎么通过代码实现的。
首先看`setScaleType`发现好像没有什么特别，是重新设置`ScaleType`枚举，然后重新布局与绘制。

    public void setScaleType(ScaleType scaleType) {
        if (scaleType == null) {
            throw new NullPointerException();
        }

        if (mScaleType != scaleType) {
            mScaleType = scaleType;

            setWillNotCacheDrawing(mScaleType == ScaleType.CENTER);            

            requestLayout();
            invalidate();
        }
    }

`Ctrl+F`搜索  `ScaleType`，仔细一看，原来藏在`configureBounds`里，这里对`ScaleType`枚举进行处理，这个方法我们刚才已经在`updateDrawable`中看到过，
接下来看下代码。

    private void configureBounds() {
        if (mDrawable == null || !mHaveFrame) {
            return;
        }

        //期望显示的宽高，即要显示的Drawable的宽高
        int dwidth = mDrawableWidth;
        int dheight = mDrawableHeight;
        
        //ImageView中显示内容的宽与高（不包括内边距）
        int vwidth = getWidth() - mPaddingLeft - mPaddingRight;
        int vheight = getHeight() - mPaddingTop - mPaddingBottom;

        //判断ImageView的显示内容宽高是否与期望显示的宽高一致
        boolean fits = (dwidth < 0 || vwidth == dwidth) &&
                       (dheight < 0 || vheight == dheight);

        if (dwidth <= 0 || dheight <= 0 || ScaleType.FIT_XY == mScaleType) {
            /* 如果Drawable没有明确的尺寸或者设定为FIX_XY，那么图片内容会充满整个ImageView
            */
            mDrawable.setBounds(0, 0, vwidth, vheight);
            mDrawMatrix = null;
        } else {
            //否则我们需要自己处理缩放，让Drawable显示原有尺寸
            mDrawable.setBounds(0, 0, dwidth, dheight);

            if (ScaleType.MATRIX == mScaleType) {
                //判断mMatric是否为单位矩阵，如果不是的话将mMatric赋值给mDrawMatric,用于onDraw中对图片进行变换
                if (mMatrix.isIdentity()) {
                    mDrawMatrix = null;
                } else {
                    mDrawMatrix = mMatrix;
                }
            } else if (fits) {
                //如果刚好合适则不需要变换
                mDrawMatrix = null;
            } else if (ScaleType.CENTER == mScaleType) {
                //如果ScaleType为CENTER，对图片不进行缩放，将图片移动到ImageView中心
                mDrawMatrix = mMatrix;
                mDrawMatrix.setTranslate((int) ((vwidth - dwidth) * 0.5f + 0.5f),
                                         (int) ((vheight - dheight) * 0.5f + 0.5f));
            } else if (ScaleType.CENTER_CROP == mScaleType) {
               //如果ScaleType为CENTER_CROP ，对图片进行等比缩放至图片宽高大于等于ImageView，将图片中心移动到ImageView中心
                mDrawMatrix = mMatrix;

                float scale;
                float dx = 0, dy = 0;

                if (dwidth * vheight > vwidth * dheight) {
                    scale = (float) vheight / (float) dheight; 
                    dx = (vwidth - dwidth * scale) * 0.5f;
                } else {
                    scale = (float) vwidth / (float) dwidth;
                    dy = (vheight - dheight * scale) * 0.5f;
                }

                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate((int) (dx + 0.5f), (int) (dy + 0.5f));
            } else if (ScaleType.CENTER_INSIDE == mScaleType) {
             //如果ScaleType为CENTER_INSIDE ，对图片进行等比缩放至图片宽高小等于ImageView，将图片中心移动到ImageView中心
                mDrawMatrix = mMatrix;
                float scale;
                float dx;
                float dy;
                
                if (dwidth <= vwidth && dheight <= vheight) {
                    scale = 1.0f;
                } else {
                    scale = Math.min((float) vwidth / (float) dwidth,
                            (float) vheight / (float) dheight);
                }
                
                dx = (int) ((vwidth - dwidth * scale) * 0.5f + 0.5f);
                dy = (int) ((vheight - dheight * scale) * 0.5f + 0.5f);

                mDrawMatrix.setScale(scale, scale);
                mDrawMatrix.postTranslate(dx, dy);
            } else {
                //剩下三种情况生成相应的转化矩阵
                mTempSrc.set(0, 0, dwidth, dheight);
                mTempDst.set(0, 0, vwidth, vheight);
                
                mDrawMatrix = mMatrix;
                mDrawMatrix.setRectToRect(mTempSrc, mTempDst, scaleTypeToScaleToFit(mScaleType));
            }
        }
    }

通过`configureBounds`中可以知道`ScaleType`本质就是通过`Matrix`矩阵变换,主要通过缩放和位移实现具体效果。
ImageView构造函数中初始化时将`ScaleType`默认为`FIT_CENTER`。

    mScaleType  = ScaleType.FIT_CENTER;


####cropToPadding 属性
`CropToPadding`这个属性感觉平时不太用，尝试用了下又感觉没什么效果，所以这个属性究竟是怎么其效果的呢？
首先我们可以找到`mCropToPadding`这个变量，我们平时修改的就是这个值，进一步查找，可以发现在`onDraw`中有用到这个变量，所以这个变量是在图形绘制过程中生效的。我们来看下`onDraw`中的相关代码。

    @Override
    protected void onDraw(Canvas canvas) {
        super.onDraw(canvas);

        if (mDrawable == null) {
            return; 
        }

        if (mDrawableWidth == 0 || mDrawableHeight == 0) {
            return;     
        }

        //如果变换矩阵为空，并且没有上内边距和左内边距则只直接绘制
        if (mDrawMatrix == null && mPaddingTop == 0 && mPaddingLeft == 0) {
            mDrawable.draw(canvas);
        } else {
            int saveCount = canvas.getSaveCount();
            canvas.save();
            //如果mCropToPadding设置为ture，需要裁剪区域，裁剪区域要考虑到x和y方向的内容滚动位移
            if (mCropToPadding) {
                final int scrollX = mScrollX;
                final int scrollY = mScrollY;
                canvas.clipRect(scrollX + mPaddingLeft, scrollY + mPaddingTop,
                        scrollX + mRight - mLeft - mPaddingRight,
                        scrollY + mBottom - mTop - mPaddingBottom);
            }
            //根据上内边距和左内边距确定内容绘制起点
            canvas.translate(mPaddingLeft, mPaddingTop);
            //如果存在变换矩阵则进行变换
            if (mDrawMatrix != null) {
                canvas.concat(mDrawMatrix);
            }
            mDrawable.draw(canvas);
            canvas.restoreToCount(saveCount);
        }
    }


从上面代码中我们可以看到`mCropToPadding`如果为`true`，`canvas`会划出除去内边距的实际显示区域，这里我们还发现除了考虑到`padding`属性，还要考虑`mScrollX` `mScrollY`两个属性，为什么呢？
>The offset, in pixels, by which the content of this view is scrolled

官方的解释是内容区域在View中滚动偏移量，View容器实际是不会滚动的，真正滚动效果是通过内容的移动达到滚动（偏移量，也可以理解成位移），而`padding`是包括在内容区域里的，这两个偏移量会让`padding`区域滚动，从而导致原有`padding`效果失效。以下代码是`View`类中获取绘制区域的方法，绘制区域是考虑到`Scroll`偏移的。

    public void getDrawingRect(Rect outRect) {
        outRect.left = mScrollX;
        outRect.top = mScrollY;
        outRect.right = mScrollX + (mRight - mLeft);
        outRect.bottom = mScrollY + (mBottom - mTop);
    }

为了让`padding`区域是基于`View`，所以我们要事先绘制好期望的内容区域，避免`Scroll`带来的影响。画了张图。
![](http://upload-images.jianshu.io/upload_images/2401022-e2e1c0b5355a1294.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

除了`Scroll`偏移会影响边距，我们能从代码中发现`Matrix`矩阵缩放也会影响到边距，大家可以回到上面讲`ScaleType`的部分，我们可以看到``CENTER  ,  CENTER_CROP  ,  CENTER_INSIDE  ,  FIT_CENTER  ,  FIT_START  ,  FIT_END``的矩阵变换都是基于不包括内边距的区域的（·`vheight` ，`vwidth`），而`MATRIX`只是设置了下`mDrawMatix`，没有前几种缩放类型做的范围处理。`onDraw`代码中我们也能看到是先`translate`确定绘制起点，然后再`contact`矩阵变换，如果我们这样做。

    <ImageView
    android:id="@+id/imageView"
    android:layout_width="80dp"
    android:layout_height="80dp"
    android:background="@color/colorAccent"
    android:cropToPadding="false"
    android:padding="10dp"
    android:scaleType="matrix"
    android:src="@mipmap/ic_launcher" />


    ImageView imageView = (ImageView) findViewById(R.id.imageView);

    Matrix matrix = new Matrix();
    matrix.setTranslate(-40, -40);

    imageView.setImageMatrix(matrix);

最后我们看到的效果是这样的，同样我们理解中的`padding`失效了
![](http://upload-images.jianshu.io/upload_images/2401022-f7148a52c3dc338a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
如果我们把`cropToPadding`属性设置为`true`则得到下面的效果。
![](http://upload-images.jianshu.io/upload_images/2401022-1e2828dc662f2605.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
所以总结来说`cropToPadding`是`ImageView`为了补偿其他属性导致我们最终的“理解的内边距效果”实现而做出的调整，目前了解到如果你的`ImageView`设置了`scrollX`  `scrollY`或者将`scaleType`为`MATRIX`时又需要保证`padding`的效果，可以将`cropToPadding`设置为`true`。

####adjustViewBounds属性
我们可以通过`ScaleType`让显示的图片根据`ImageView`大小进行各种类型的调整，我们可不可以让`ImageView`根据图片内容调整大小呢？这时候我们就需要用到`adjustViewBounds`这个属性啦，能让显示的图片原有比例，调整`ImageView`到合适的大小，达到我们想要的图片与`ImageView`无缝衔接，我们从`ImageView`代码中看下是怎么实现的。老规矩搜索`mAdjustViewBounds`属性，就可以发现这个属性主要是在`onMeasure`方法中起作用。

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        resolveUri();
        int w;
        int h;
        
        // 期望的显示比例（不包括padding），也就是要显示图片的原有比例
        float desiredAspect = 0.0f;
        
        // 用于判断是否允许调整宽度，默认为false
        boolean resizeWidth = false;
        
        // 用于判断是否允许调整高度，默认为false
        boolean resizeHeight = false;
        
        //获取父容器为ImageView设置的尺寸测量模式
        final int widthSpecMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightSpecMode = MeasureSpec.getMode(heightMeasureSpec);

        if (mDrawable == null) {
            mDrawableWidth = -1;
            mDrawableHeight = -1;
            w = h = 0;
        } else {
            w = mDrawableWidth;
            h = mDrawableHeight;
            if (w <= 0) w = 1;
            if (h <= 0) h = 1;

            //如果设置mAdjustViewBounds为true，判断widthSpecMode和heightSpecMode，如果不是EXACTLY则支持调整，并获得期望的宽高比
            if (mAdjustViewBounds) {
                resizeWidth = widthSpecMode != MeasureSpec.EXACTLY;
                resizeHeight = heightSpecMode != MeasureSpec.EXACTLY;
                
                desiredAspect = (float) w / (float) h;
            }
        }
        
        int pleft = mPaddingLeft;
        int pright = mPaddingRight;
        int ptop = mPaddingTop;
        int pbottom = mPaddingBottom;

        int widthSize;
        int heightSize;

        //如果宽或者高可以调整则进行调整，并且只能调整一个
        if (resizeWidth || resizeHeight) {

            // 获取最大可能的宽度
            widthSize = resolveAdjustedSize(w + pleft + pright, mMaxWidth, widthMeasureSpec);

            // 获取最大可能的高度
            heightSize = resolveAdjustedSize(h + ptop + pbottom, mMaxHeight, heightMeasureSpec);

            if (desiredAspect != 0.0f) {
                // 查看实际的宽高比
                float actualAspect = (float)(widthSize - pleft - pright) /
                                        (heightSize - ptop - pbottom);
                // 如果实际比例与期望比例差距小于0.0000001则不进行调整
                if (Math.abs(actualAspect - desiredAspect) > 0.0000001) {
                    
                    boolean done = false;
                    
                    // 调整宽度去适应高
                    if (resizeWidth) {
                        int newWidth = (int)(desiredAspect * (heightSize - ptop - pbottom)) +
                                pleft + pright;

                        //是否允许调整宽度，即高度固定，另一个参数是和兼容性有关先不研究
                        if (!resizeHeight && !mAdjustViewBoundsCompat) {
                            //计算调整后的高度
                            widthSize = resolveAdjustedSize(newWidth, mMaxWidth, widthMeasureSpec);
                        }
                           
                        //调整高度符合规则，调整结束
                        if (newWidth <= widthSize) {
                            widthSize = newWidth;
                            done = true;
                        } 
                    }
                    
                    // 调整高度，思路和宽度一致
                    if (!done && resizeHeight) {
                        int newHeight = (int)((widthSize - pleft - pright) / desiredAspect) +
                                ptop + pbottom;

                        if (!resizeWidth && !mAdjustViewBoundsCompat) {
                            heightSize = resolveAdjustedSize(newHeight, mMaxHeight,
                                    heightMeasureSpec);
                        }

                        if (newHeight <= heightSize) {
                            heightSize = newHeight;
                        }
                    }
                }
            }
        } else {
            // 如果不需要尺寸调整，只做常规的测量计算
            w += pleft + pright;
            h += ptop + pbottom;
                
            w = Math.max(w, getSuggestedMinimumWidth());
            h = Math.max(h, getSuggestedMinimumHeight());
            // 计算高度和状态
            widthSize = resolveSizeAndState(w, widthMeasureSpec, 0);
            heightSize = resolveSizeAndState(h, heightMeasureSpec, 0);
        }

        setMeasuredDimension(widthSize, heightSize);
    }

上面遇到了`resolveAdjustedSize`  `resolveSizeAndState`之前没遇到过，接下看看下这两个方法。

    private int resolveAdjustedSize(int desiredSize, int maxSize,
                               int measureSpec) {
        int result = desiredSize;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize =  MeasureSpec.getSize(measureSpec);
        switch (specMode) {
            case MeasureSpec.UNSPECIFIED:
                // 父容器没有约束，如果期望值在最大值范围内则取期望值
                result = Math.min(desiredSize, maxSize);
                break;
            case MeasureSpec.AT_MOST:
                // 父容器给出调整的最大值，这个值与期望值，设置的最大值之间取最小值
                result = Math.min(Math.min(desiredSize, specSize), maxSize);
                break;
            case MeasureSpec.EXACTLY:
                // 没有选择，直接返回父容器分配的值
                result = specSize;
                break;
        }
        return result;
    }

`resolveAdjustedSize`就是通过父容器分配的`MeasureSpec`的约束下尽可能去取期望值。

    public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize =  MeasureSpec.getSize(measureSpec);
        switch (specMode) {
        case MeasureSpec.UNSPECIFIED:
            result = size;
            break;
        case MeasureSpec.AT_MOST:
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;
            } else {
                result = size;
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;
            break;
        }
        return result | (childMeasuredState&MEASURED_STATE_MASK);
    }

`resolveSizeAndState`是父类`View`中的一个静态方法，和上面`resolveAdjustedSize`差不多，也是根据父容器分配的`MeasureSpec`的约束下尽可能去取要显示`Drawable`的宽高，也包括一些位操作用于表示状态。
了解了原理我们可以结合实际情况分析一下。
(1)宽度设置为`match_parent`高度设置为`wrap_content`,第一张图为`adjustViewBounds`为  `false`,第二张图为`true`
![](http://upload-images.jianshu.io/upload_images/2401022-728ad1d29bd5a8a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
![](http://upload-images.jianshu.io/upload_images/2401022-41381e2c6795eeb8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`match_parent`对应的`specMode`为`EXACTLY`,`wrap_content`的`specMode`为`AT_MOST`，如果将`adjustViewBounds`设置为`true`则`ImageView`的高度依据图片的原有比例调整，否则`ImageView`的高度为显示图片的高度。
(2)宽度与高度设置均为`wrap_content`，无论`adjustViewBounds`取值如何，效果一样。
![](http://upload-images.jianshu.io/upload_images/2401022-fdc916d8529216aa.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`wrap_content`的`specMode`为`AT_MOST`,从`onMeasure`方法中可知，如果两个尺寸都不定的话，是不会进行调整。
(3)宽度与高度设置均为`match_parent`，无论`adjustViewBounds`取值如何，效果也都一样。
![](http://upload-images.jianshu.io/upload_images/2401022-3daad8c0e26749a8.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
`wrap_content`的`specMode`为`EXACTLY`，所以说宽和高均不可变，也就是说不能调整，同理如果宽高设置为具体尺寸，`adjustViewBounds`也是起不了作用的。

###ImageView拓展
`ImageView`是我们UI绘制过程一个必不可少的基础控件，然而在实际开发过程中我们发现有些情况下`ImageView`功能还不够，所以我们会基于`ImageView`基础上去拓展，这里找了两个小案例。
####RoundImageView
现在很多应用中，头像采用是圆形的，而`ImageVeiw`只能显示方形图片，最简单的方法是在加一层背景色遮罩，当然也可以通过自定View来解决。这是一个自定义控件用来显示圆形图片，当然现在也有很多图片库来显示圆形头像。具体代码就不贴了，实现思路主要是继承`ImageView`，重写`OnDraw`，通过设置`Xfermodes`(这里我理解为图形叠加模式)在圆形上绘制图片，达到绘制圆形图像的效果。

    public Bitmap getCroppedRoundBitmap(Bitmap bmp, int radius) {

        Bitmap scaledSrcBmp;

        int diameter = radius * 2;

       ...
        // 为了防止宽高不相等，造成圆形图片变形，因此截取长方形中处于中间位置最大的正方形图片

        int bmpWidth = bmp.getWidth();
        int bmpHeight = bmp.getHeight();
        int squareWidth = 0, squareHeight = 0;
        int x = 0, y = 0;

        Bitmap squareBitmap;

        if (bmpHeight > bmpWidth) {
        
            squareWidth = squareHeight = bmpWidth;
            x = 0;
            y = (bmpHeight - bmpWidth) / 2;
            
            squareBitmap = Bitmap.createBitmap(bmp, x, y, squareWidth, squareHeight);
        } else if (bmpHeight < bmpWidth) {
        
            squareWidth = squareHeight = bmpHeight;
            x = (bmpWidth - bmpHeight) / 2;
            y = 0;

            squareBitmap = Bitmap.createBitmap(bmp, x, y, squareWidth, squareHeight);
        } else {
            squareBitmap = bmp;
        }

        //将正方形图片缩放至与ImageView直径相同
        if (squareBitmap.getWidth() != diameter || squareBitmap.getHeight() != diameter) {

            scaledSrcBmp = Bitmap.createScaledBitmap(squareBitmap, diameter, diameter, true);
        } else {
            scaledSrcBmp = squareBitmap;
        }

        Bitmap output = Bitmap.createBitmap(scaledSrcBmp.getWidth(),
                scaledSrcBmp.getHeight(),
                Bitmap.Config.ARGB_8888);

        //获取处理图片的Canvas
        Canvas canvas = new
                Canvas(output);

        Paint paint = new Paint();

        Rect rect = new Rect(0, 0, scaledSrcBmp.getWidth(), scaledSrcBmp.getHeight());

        paint.setAntiAlias(true);//开启抗锯齿
        paint.setFilterBitmap(true);//开启过滤效果
        paint.setDither(true);//开启抖动

        canvas.drawARGB(0, 0, 0, 0);

        //绘制圆形
        canvas.drawCircle(scaledSrcBmp.getWidth() / 2, scaledSrcBmp.getHeight() / 2, scaledSrcBmp.getWidth() / 2, paint);

        //设置叠加模式为SRC_IN,即在圆形的返回内绘制图像
        paint.setXfermode(new PorterDuffXfermode(PorterDuff.Mode.SRC_IN));

        canvas.drawBitmap(scaledSrcBmp, rect, rect, paint);

        //释放内存
        bmp = null;
        squareBitmap = null;
        scaledSrcBmp = null;

        return output;

    }

`onDraw`绘制圆形图片

    @Override
    protected void onDraw(Canvas canvas) {
        ...
        
        Bitmap roundBitmap = getCroppedRoundBitmap(bitmap, radius);

        canvas.drawBitmap(roundBitmap, defaultWidth / 2
                - radius, defaultHeight / 2
                - radius, null);

    }

####AutoSquareImageView
这是我做Android项目遇到的第一个难题，四个`ImageView`是要通过`weight`属性来实现水平方向上的宽度均等，但是UI要求是让这个图形按钮正方形显示，不同手机水平宽度不同，需要自适应高度，为了达到正方形效果需要自定义`AutoSquareImageView`，修改思路其实很简单，将`ImageView`宽度通过`weight`去获取，高度设置为`wrap_content`，重写`onMeasure`方法，计算正常流程下的宽与高，然后取大值达到满足正方形图的最小边长。

    @Override
    protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        super.onMeasure(widthMeasureSpec, heightMeasureSpec);

        int measuredWidth = getMeasuredWidth();
        int measuredHeight = getMeasuredHeight();
        //保持长度大的一边
        if (measuredWidth > measuredHeight) {
            setMeasuredDimension(measuredWidth, measuredWidth);
        } else {
            setMeasuredDimension(measuredHeight, measuredHeight);
        }
    }

###总结
* `ImageView`显示的图片资源可以是通过XML属性或者Java代码指定显示的图片内容，可以通过设置`Resource`、`Uri` 、`Bitmap`对象、`Drawable`对象来指定。
* `ScaleType`属性的修改本质上根据设定的枚举将变换赋值给`mMatrix`，最终通过`onDraw`中的矩阵操作实现。
* `cropToPadding`属性可以让图片内容是在`padding`基础上进行变换（缩放，移动），根据实践如果使用了`mScrollX` `mScrollY`属性或者`scaleType`设置为`MATRIX`类型时会配合用到这个属性。
* `adjustViewBounds`属性主要是让`ImageView`根据图片原有比例的约束下将调整大小，注意必须要限定住宽或者高，即将其中一个属性设置为`match_parent`或者确定尺寸。

--------------------------------------------------------------------------------------------------------------------------------


笔记就写到这里了，这也是我第一次在网上写文章，写的也是基础知识，如果有什么不对的地方欢迎大家批评指正！写文章也是为了能把学习的知识点进行梳理，然后也会有学习的动力，内容可能不高级，但希望能一点点慢慢提高。