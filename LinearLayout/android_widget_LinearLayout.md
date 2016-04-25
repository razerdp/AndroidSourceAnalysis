##android.widget.LinearLayout 源码分析

###源码版本：Api 23（为更方便理解，本文采取重要代码上写注释加上一些简要分析的方法来说明）

在Android里面，LinearLayout是我们最常用的布局之一，另外一个估计就是RelativeLayout。

了解LinearLayout，就先从它和其他布局的异同开始，其实这些属性我们日常都有使用：

- `orientation(int)`：作为LinearLayout必须使用的属性之一，支持`纵向排布`或者`水平排布`子控件

- `weightSum(缺省值为1.0)/weight`：指定weight的总和/权重

- `baselineAligned(boolean)`：基线对齐

- `baselineAlignedChildIndex(int)`：该LinearLayout下的view以某个**继承TextView**的View的基线对齐

- `measureWithLargestChild(boolean)`：当值为true，所有带权重属性的View都会使用最大View的最小尺寸

- `divider(drawable in java/reference in xml)`**[api>11]**：如同您常在ListView使用一样，为LinearLayout添加分割线

	+ 其附加属性为showDividers(middle|end|beginning|none):
		* middle 在每一项中间添加分割线
		* end 在整体的最后一项添加分割线
		* beginning 在整体的最上方添加分割线
		* none 无


本篇主要也是针对以上几个LinearLayout独有属性进行分析，其他的因为基本是ViewGroup共有或者View共有的，所以就不会进行详细阐述。
***
#Measure

对View的分析永远都脱离不了测量->布局->绘制。

而对于ViewGroup，由于通常不需要绘制（ps:setWillNotDraw方法或者给ViewGroup添加背景可以触发ViewGroup绘制），所以我们着重于测量和布局

因此此处略过构造器以及一大堆的attrs属性获取/赋值，我们直接从Measure开始。

首先我们看看**onMeasure**方法

![](https://github.com/razerdp/AndroidSourceAnalysis/blob/master/LinearLayout/img/img_onMeasure.png)

可以看到，measure里面根据不同的orientation会进行不同的测量。

接下来我们看看垂直排布的测量（代码很长，超过300行，因此分段截取）。

【如果您懒得打开SDK，可以到[GrepCode(api 22)](http://www.grepcode.com/file/repository.grepcode.com/java/ext/com.google.android/android/5.1.1_r1/android/widget/LinearLayout.java#LinearLayout.measureVertical%28int%2Cint%29) 查看】

首先看看方法的第一部分，一些值的定义：

```java

    void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {

		// mTotalLength作为LinearLayout成员变量，其主要目的是在测量的时候通过累加得到所有子控件的高度和（Vertical）或者宽度和（Horizontal）
        mTotalLength = 0;
		// maxWidth用来记录所有子控件中控件宽度最大的值。
        int maxWidth = 0;
		// 子控件的测量状态，会在遍历子控件测量的时候通过combineMeasuredStates来合并上一个子控件测量状态与当前遍历到的子控件的测量状态，采取的是按位相或
        int childState = 0;
		
		/**
		 * 以下两个最大宽度跟上面的maxWidth最大的区别在于matchWidthLocally这个参数
		 * 当matchWidthLocally为真，那么以下两个变量只会跟当前子控件的左右margin和相比较取大值
		 * 否则，则跟maxWidth的计算方法一样
		 */
		// 子控件中layout_weight<=0的View的最大宽度
        int alternativeMaxWidth = 0;
		// 子控件中layout_weight>0的View的最大宽度
        int weightedMaxWidth = 0;
		// 是否子控件全是match_parent的标志位，用于判断是否需要重新测量
        boolean allFillParent = true;
		// 所有子控件的weight之和
        float totalWeight = 0;

		// 如您所见，得到所有子控件的数量，准确的说，它得到的是所有同级子控件的数量
		// 在官方的注释中也有着对应的例子
		// 比如TableRow，假如TableRow里面有N个控件，而LinearLayout（TableLayout也是继承LinearLayout哦）下有M个TableRow，那么这里返回的是M，而非M*N
		// 但实际上，官方似乎也只是直接返回getChildCount()，起这个方法名的原因估计是为了让人更加的明白，毕竟如果是getChildCount()可能会让人误认为为什么没有返回所有（包括不同级）的子控件数量
        final int count = getVirtualChildCount();
        
		// 得到测量模式
        final int widthMode = MeasureSpec.getMode(widthMeasureSpec);
        final int heightMode = MeasureSpec.getMode(heightMeasureSpec);

		// 当子控件为match_parent的时候，该值为ture，同时判定的还有上面所说的matchWidthLocally，这个变量决定了子控件的测量是父控件干预还是填充父控件（剩余的空白位置）。
        boolean matchWidth = false;
		
        boolean skippedMeasure = false;

        final int baselineChildIndex = mBaselineAlignedChildIndex;        
        final boolean useLargestChild = mUseLargestChild;

        int largestChildHeight = Integer.MIN_VALUE;
	}

```

这一大堆变量之所以要注释一遍，是因为这里是最容易混淆，从而导致后面的分析不明所以。

接下来就是我们比较熟悉的步骤了，一个大的for循环，各种主要操作都在这里了。

```java

void measureVertical(int widthMeasureSpec, int heightMeasureSpec) {   

	...接上面那段

	// 遍历所有子控件
   	// See how tall everyone is. Also remember max width.
        for (int i = 0; i < count; ++i) {
            final View child = getVirtualChildAt(i);

            if (child == null) {
				// 目前而言，measureNullChild()方法返回的永远是0，估计是谷歌留下来以后或许有补充的。
                mTotalLength += measureNullChild(i);
                continue;
            }

            if (child.getVisibility() == View.GONE) {
			   // 同上，返回的都是0，事实上这里的意思应该是当前遍历到的View为Gone的时候，就跳过这个View，下一句的continue关键字也正是这个意思。
			   // 忽略当前的View，这也就是为什么Gone的控件不占用布局资源的原因。
               i += getChildrenSkipCount(child, i);
               continue;
            }
			
			// 判断divider是否在每个item前面绘制
			// 关于这里，可以查看附录下的对应方法 
			// 假如有divider，由于是测量纵向的，那么divider肯定是水平的（类似于ListView的divider）
			// 因此总高度需要加上divider的高度
            if (hasDividerBeforeChildAt(i)) {
                mTotalLength += mDividerHeight;
            }

            LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams) child.getLayoutParams();
			
			// 得到每个子控件的LayoutParams后，累加权重和，后面用于跟weightSum相比较
            totalWeight += lp.weight;
            
			// 我们都知道，测量模式有三种：
			// * UNSPECIFIED：父控件对子控件无约束
			// * Exactly：父控件对子控件强约束，子控件永远在父控件边界内，越界则裁剪。如果要记忆的话，可以记忆为有对应的具体数值或者是Match_parent
			// * AT_Most：子控件为wrap_content的时候，测量值为AT_MOST。

			/**下面的if/else就是用来测量子控件有weight同时高度为0的情况*/
            if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
                // Optimization: don't bother measuring children who are going to use
                // leftover space. These views will get measured again down below if
                // there is any leftover space.

				// 假如我们的LinearLayout的测量模式为Exactly，也就是对子控件有强约束
				// 同时当前的子控件的height=0同时又有权重
				// 那么暂时先给个标志位，当前的控件暂不测量（注意，这里不用continue跳出循环，因为该View并非Gone掉，这个标志可以理解为“稍后处理”）
				// 虽然暂时不测量这个子控件，但是总高度还是会取子控件的top/bottom的margin与当前总高度的和。
				// （当然，这个也是可以稍后再加上的，但在这里写上估计是设计者怕遗忘了或者为了让代码更容易读懂）
                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
				// 标志
                skippedMeasure = true;
            } else {
				// 这里的这个oldHeight也可以当作标志，后面会用到
                int oldHeight = Integer.MIN_VALUE;
				
				// 假如我们的LinearLayout的测量模式不是Exactly 
                if (lp.height == 0 && lp.weight > 0) {
                    // heightMode is either UNSPECIFIED or AT_MOST, and this
                    // child wanted to stretch to fill available space.
                    // Translate that to WRAP_CONTENT so that it does not end up
                    // with a height of 0

					// 当子控件恰好又符合height=0同时拥有weight
                    oldHeight = 0;
					// 则强制将当前子控件的高度设为WRAP_CONTENT
					// 这里为何要将高度设为WRAP_CONTENT，其实在上面官方的注释中我们也可以看到
					// 由于我们给定了View的高度为0，但又拥有weight，那么意味着
					// 这个子控件是需要显示的，如果高度设置为0，那意味着这个weight有跟
					// 没有是没啥两样
					// 我们知道，自定义一个控件，通常在onMeasure的时候，都会针对
					// WRAP_CONTENT测量模式给定一个值，这就是为了让系统可以针对这个模式
					// 来进行恰当的布局。而不是说总是给0.所以将子控件的高度设为	
					// WRAP_CONTENT，倒不如说是为了拿到默认的那个值
                    lp.height = LayoutParams.WRAP_CONTENT;
                }

                // Determine how big this child would like to be. If this or
                // previous children have given a weight, then we allow it to
                // use all available space (and we will shrink things later
                // if needed).

				// 接下来的这个方法，实际上就是measureChildWithMargins，ViewGroup对子控件测量的方法，这个方法这里不详细描述，毕竟ViewGroup基本都用这个方法
				// 不过值得注意的是高度的测量，高度的测量里面是根据权重给定的
				// 拥有权重的时候，高度直接给0，因为有权重的话，需要LinearLayout自己做处理
                measureChildBeforeLayout(
                       child, i, widthMeasureSpec, 0, heightMeasureSpec,
                       totalWeight == 0 ? mTotalLength : 0);

				// 由于还在LinearLayout模式不为Exactly的else李main
				// 所以oldHeight=0，然而当前子控件的height给定WRAP_CONTENT
				// 那么意味着肯定符合这个if，这里又将当前子控件的height给0了
				// 咋一看，这不多余么。实则不然，因为在这一条的上面，执行了一次measure语句
				// 也就是说，子控件带着WRAP_CONTENT执行了measure，此时子控
				// 的measureHeight不会是0，而是WRAP_CONTENT指定的那个默认值
				// 接下来则是重新给0，开始计算权重并赋予高度。
                if (oldHeight != Integer.MIN_VALUE) {
                   lp.height = oldHeight;
                }

				// 得到measure后的高度
                final int childHeight = child.getMeasuredHeight();
                final int totalLength = mTotalLength;
				// 这时候的总高度则是加上子控件的高度，getNextLocationOffset这个方法目前
				// 依然返回的是0，可能是设计者留下以后有需要的话补吧
				// 另外，此时LinearLayout的测量模式不是Exactly，所以总高度没有加上子控件的
				// 这也是为什么LinearLayout在WRAP_CONTENT的时候高度是子控件的高度和的原因
                mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                       lp.bottomMargin + getNextLocationOffset(child));

				// useLargesChild与LinearLayout特有属性measureWithLargestChild相关
				// 在遍历控件测量的过程中，不断的取控件高度的最大值并赋予给useLargesChild，然后稍后处理
                if (useLargestChild) {
                    largestChildHeight = Math.max(childHeight, largestChildHeight);
                }
            }
        }
}

```
写到这里，我们的这个遍历控件的大循环还没结束，但我们在这里总结一下，看看设计者对于权重是如何**初步**处理的。

之所以说初步处理，是因为这时候还没有对子控件的高度进行赋值，此时子控件（拥有权重）的高度其实依然为0.（当然，期间有measure一次，但最后都再次赋值为0）

上面的代码绕来绕去，其实改变的只有几个值：

- mTotalLength：所有子控件的总高度
- totalWeight：当前的权重和
- 如果要算，也可以把useLargestChild算上，虽然这个参数我们用的很少

总的来说，上面的代码就是根据当前LinearLayout的测量模式（准确的说是高度的测量模式），来进行子控件在不同情况下拥有weight时的预处理。当然，没有weight的话就只是很正常的测量而已，其调用就是`measureChildBeforeLayout`

接下来我们继续看看这个大循环剩下的内容

```java

    // See how tall everyone is. Also remember max width.
        for (int i = 0; i < count; ++i) {
		    ...接上面

            /**
             * If applicable, compute the additional offset to the child's baseline
             * we'll need later when asked {@link #getBaseline}.
             */

			
            if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
               mBaselineChildTop = mTotalLength;
            }

            // if we are trying to use a child index for our baseline, the above
            // book keeping only works if there are no children above it with
            // weight.  fail fast to aid the developer.
            if (i < baselineChildIndex && lp.weight > 0) {
                throw new RuntimeException("A child of LinearLayout with index "
                        + "less than mBaselineAlignedChildIndex has weight > 0, which "
                        + "won't work.  Either remove the weight, or don't set "
                        + "mBaselineAlignedChildIndex.");
            }

            boolean matchWidthLocally = false;
            if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
                // The width of the linear layout will scale, and at least one
                // child said it wanted to match our width. Set a flag
                // indicating that we need to remeasure at least that view when
                // we know our width.
                matchWidth = true;
                matchWidthLocally = true;
            }

            final int margin = lp.leftMargin + lp.rightMargin;
            final int measuredWidth = child.getMeasuredWidth() + margin;
            maxWidth = Math.max(maxWidth, measuredWidth);
            childState = combineMeasuredStates(childState, child.getMeasuredState());

            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
            if (lp.weight > 0) {
                /*
                 * Widths of weighted Views are bogus if we end up
                 * remeasuring, so keep them separate.
                 */
                weightedMaxWidth = Math.max(weightedMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            } else {
                alternativeMaxWidth = Math.max(alternativeMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            }

            i += getChildrenSkipCount(child, i);
        }
```



对应跳转链接：

- [hasDividerBeforeChildAt](#hasDividerBeforeChildAt)



#附录 - LinearLayout下被调用方法的解析

<span id="hasDividerBeforeChildAt">
```java

	/**
     * Determines where to position dividers between children.
     *
     * @param childIndex Index of child to check for preceding divider
     * @return true if there should be a divider before the child at childIndex
     * @hide Pending API consideration. Currently only used internally by the system.
     */
    protected boolean hasDividerBeforeChildAt(int childIndex) {
        if (childIndex == getVirtualChildCount()) {
            // Check whether the end divider should draw.
            return (mShowDividers & SHOW_DIVIDER_END) != 0;
        }
        boolean allViewsAreGoneBefore = allViewsAreGoneBefore(childIndex);
        if (allViewsAreGoneBefore) {
            // This is the first view that's not gone, check if beginning divider is enabled.
            return (mShowDividers & SHOW_DIVIDER_BEGINNING) != 0;
        } else {
            return (mShowDividers & SHOW_DIVIDER_MIDDLE) != 0;
        }
    }

```
</span>

上面的代码不多，其主要控制点在于mShowDividers，mShowDividers之前也说过，它属于divider的附加属性，其值从None/Beginning/Middle/End 分别为0/1/2/3/4 而上面的代码的作用就是比较mShowDividers与这四个值，采取按位相与。在上面的代码我们可以看到有两个if分支。

 - 第一个if比较好容易理解，假如传进来的View，刚好是最后一个View，同时showDividers也正好是END,那么返回true，不满足任意一个，则返回false.
 
 - 第二个if则涉及到一个方法 allViewsAreGoneBefore，这个方法只是一个很简单的子控件遍历，用来判断是否所有子控件都是Gone状态。如果都是Gone，那么只有绘制在所有View的前面，如果不全是Gone，那么才能够绘制在View与View的中间。（当然，前提是设置的showDivider模式匹配）
