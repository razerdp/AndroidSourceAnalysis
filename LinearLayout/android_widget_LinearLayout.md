##android.widget.LinearLayout 源码分析

####声明.本项目源码基于Api 23，令因为本人能力有限，如果错误，欢迎督促指正。
***
###1.谈谈LinearLayout
Android的常用布局里，LinearLayout属于使用频率很高的布局。RelativeLayout也是，但相比于RelativeLayout每个子控件都需要给上ID以供另一个相关控件摆放位置来说，LinearLayout两个方向上的排列规则在明显垂直/水平排列情况下使用更加方便。

同时，出于性能上来说，一般而言功能越复杂的布局，性能也是越低的（不考虑嵌套的情况下）。

相比于RelativeLayout无论如何都是两次测量的情况下，LinearLayout只有子控件设置了weight属性时，才会有二次测量，其余情况都是一次。

另外，LinearLayout的高级用法除了weight，还有divider，baselineAligned等用法，虽然用的不常见就是了。

以下是LinearLayout相比于其他布局所拥有的特性：

| 属性| 值类型| 描述|备注|
| -----|:----:| :----:|:-----:|
| orientation | int | 作为LinearLayout必须使用的属性之一，支持纵向排布或者水平排布子控件 ||
| weightSum | float | 指定权重总和 |缺省值为1.0|
| baselineAligned | boolean | 基线对齐||
| baselineAlignedChildIndex| int|该LinearLayout下的view以某个**继承TextView**的View的基线对齐||
| measureWithLargestChild| boolean | 当值为true，所有带权重属性的View都会使用最大View的最小尺寸||
| divider（需要配合showDividers使用）| drawable in java/reference in xml| 如同您常在ListView使用一样，为LinearLayout添加分割线|**[api>11]**  同时如果是自己建立的drawable，请指定size |

【注意】divider附加属性为showDividers(middle|end|beginning|none):
 - middle 在每两项之间添加分割线
 - end 在整体的最后一项添加分割线
 - beginning 在整体的首项添加分割线
 - none 无


**本篇主要针对LinearLayout两个方向的测量、weight和divider进行分析，其余属性因为比较冷门，因此不会详说**
***
###2.使用方法
对于LinearLayout的使用，相信您闭着眼睛都能写出来，因此这里就略过了。
***
###3.源码分析

源码分析阶段主要针对这几个地方：
 - measure流程
 - weight的计算
 - divider的插入

后两者的主要工作其实都是被包含在measure里面的，因此对于LinearLayout来说，最重要的，依然是measure.
####3.1 measure

在LinearLayout的onMeasure()里面，所有的测量都根据mOrientation这个int值来进行水平或者垂直的测量计算。

我们都知道，java中int在初始化不分配值的时候，都是默认的0，因此如果我们不指定orientation，measure则会按照水平方向来测量【水平orientation=0/垂直orientation=1】

接下来我们主要看看**measureVertical**方法，了解了垂直方向的测量之后，水平方向的也就不难理解了，为了篇幅，我们主要分析垂直方向的测量。

measureVertical方法除去注释，大概200多行，因此我们分段分析。

方法主要分为三大块：
 - 一大堆变量
 - 一个主要的for循环来不断测量子控件
 - 其余参数影响以及根据是否有weight再次测量

#####3.1.1
**一大堆变量**
为何这里要说说变量，因为这些变量都会极大的影响到后面的测量，同时也是十分容易混淆的，所以这里需要贴一下。

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
这里有很多变量和值，事实上，直到现在，我依然没有完全弄明白这些值的意义。

在这一大堆变量里面，我们主要留意的是三个方面：
 - mTotalLength：这个就是最终得到的整个LinearLayout的高度（子控件高度累加及自身padding）
 - 三个跟width相关的变量
 - weight相关的变量

***
#####3.1.2 
**测量**

通过for循环不断的得到子控件然后根据自己的定义进行赋值，这就是LinearLayout测量里面最重要的一步。

这里的代码比较长，去掉注释后有100行左右，因此这里采取重要地方注释结合文字描述来分析。

```java
void measureHorizontal(int widthMeasureSpec, int heightMeasureSpec) {
		// ...接上面的一大堆变量
        for (int i = 0; i < count; ++i) {

            final View child = getVirtualChildAt(i);

            if (child == null) {
				// 目前而言，measureNullChild()方法返回的永远是0，估计是设计者留下来以后或许有补充的。
                mTotalLength += measureNullChild(i);
                continue;
            }
           
            if (child.getVisibility() == GONE) {
			   // 同上，返回的都是0。
			   // 事实上这里的意思应该是当前遍历到的View为Gone的时候，就跳过这个View，下一句的continue关键字也正是这个意思。
               // 忽略当前的View，这也就是为什么Gone的控件不占用布局资源的原因。（毕竟根本没有分配空间）
                i += getChildrenSkipCount(child, i);
                continue;
            }

			// 根据showDivider的值（before/middle/end）来决定遍历到当前子控件时，高度是否需要加上divider的高度
			// 比如showDivider为before，那么只会在第0个子控件测量时加上divider高度，其余情况下都不加
            if (hasDividerBeforeChildAt(i)) {
                mTotalLength += mDividerWidth;
            }

            final LinearLayout.LayoutParams lp = (LinearLayout.LayoutParams)
                    child.getLayoutParams();
			// 得到每个子控件的LayoutParams后，累加权重和，后面用于跟weightSum相比较
            totalWeight += lp.weight;
            
			// 我们都知道，测量模式有三种：
            // * UNSPECIFIED：父控件对子控件无约束
            // * Exactly：父控件对子控件强约束，子控件永远在父控件边界内，越界则裁剪。如果要记忆的话，可以记忆为有对应的具体数值或者是Match_parent
            // * AT_Most：子控件为wrap_content的时候，测量值为AT_MOST。
			
			// 下面的if/else分支都是跟weight相关
            if (heightMode == MeasureSpec.EXACTLY && lp.height == 0 && lp.weight > 0) {
				// 这个if里面需要满足三个条件：
				// * LinearLayout的高度为match_parent(或者有具体值)
				// * 子控件的高度为0
				// * 子控件的weight>0
				// 这其实就是我们通常情况下用weight时的写法
				// 测量到这里的时候，会给个标志位，稍后再处理。此时会计算总高度
                final int totalLength = mTotalLength;
                mTotalLength = Math.max(totalLength, totalLength + lp.topMargin + lp.bottomMargin);
                skippedMeasure = true;
            } else {
				// 到这个分支，则需要对不同的情况进行测量
                int oldHeight = Integer.MIN_VALUE;

                if (lp.height == 0 && lp.weight > 0) {
					// 满足这两个条件，意味着父类即LinearLayout是wrap_content，或者mode为UNSPECIFIED
					// 那么此时将当前子控件的高度置为wrap_content
					// 为何需要这么做，主要是因为当父类为wrap_content时，其大小实际上由子控件控制
					// 我们都知道，自定义控件的时候，通常我们会指定测量模式为wrap_content时的默认大小
					// 这里强制给定为wrap_content为的就是防止子控件高度为0.
                    oldHeight = 0;
                    lp.height = LayoutParams.WRAP_CONTENT;
                }
				
				/**【1】*/
				// 下面这句虽然最终调用的是ViewGroup通用的同名方法，但传入的height值是跟平时不一样的
				// 这里可以看到，传入的height是跟weight有关，关于这里，稍后的文字描述会着重阐述
                measureChildBeforeLayout(
                       child, i, widthMeasureSpec, 0, heightMeasureSpec,
                       totalWeight == 0 ? mTotalLength : 0);

				// 重置子控件高度，然后进行精确赋值
                if (oldHeight != Integer.MIN_VALUE) {
                   lp.height = oldHeight;
                }

                final int childHeight = child.getMeasuredHeight();
                final int totalLength = mTotalLength;
				// getNextLocationOffset返回的永远是0，因此这里实际上是比较child测量前后的总高度，取大值。
                mTotalLength = Math.max(totalLength, totalLength + childHeight + lp.topMargin +
                       lp.bottomMargin + getNextLocationOffset(child));

                if (useLargestChild) {
                    largestChildHeight = Math.max(childHeight, largestChildHeight);
                }
            }

            if ((baselineChildIndex >= 0) && (baselineChildIndex == i + 1)) {
               mBaselineChildTop = mTotalLength;
            }

            if (i < baselineChildIndex && lp.weight > 0) {
                throw new RuntimeException("A child of LinearLayout with index "
                        + "less than mBaselineAlignedChildIndex has weight > 0, which "
                        + "won't work.  Either remove the weight, or don't set "
                        + "mBaselineAlignedChildIndex.");
            }

            boolean matchWidthLocally = false;
			
			// 还记得我们变量里又说到过matchWidthLocally这个东东吗
			// 当父类（LinearLayout）不是match_parent或者精确值的时候，但子控件却是一个match_parent
			// 那么matchWidthLocally和matchWidth置为true
			// 意味着这个控件将会占据父类（水平方向）的所有空间
            if (widthMode != MeasureSpec.EXACTLY && lp.width == LayoutParams.MATCH_PARENT) {
                matchWidth = true;
                matchWidthLocally = true;
            }

            final int margin = lp.leftMargin + lp.rightMargin;
            final int measuredWidth = child.getMeasuredWidth() + margin;
            maxWidth = Math.max(maxWidth, measuredWidth);
            childState = combineMeasuredStates(childState, child.getMeasuredState());

            allFillParent = allFillParent && lp.width == LayoutParams.MATCH_PARENT;
			
            if (lp.weight > 0) {
                weightedMaxWidth = Math.max(weightedMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            } else {
                alternativeMaxWidth = Math.max(alternativeMaxWidth,
                        matchWidthLocally ? margin : measuredWidth);
            }

            i += getChildrenSkipCount(child, i);
        }
	}
```
在代码中我注释了一部分，其中最值得注意的是`measureChildBeforeLayout`方法。这个方法将会决定子控件可用的剩余分配空间。

measureChildBeforeLayout最终调用的实际上是ViewGroup的measureChildWithMargins，不同的是，在传入高度值的时候（垂直测量情况下），会对weight进行一下判定

假如当前子控件的weight加起来还是为0，则说明在当前子控件之前还没有遇到有weight的子控件，那么LinearLayout将会进行正常的测量，若之前遇到过有weight的子控件，那么LinearLayout传入0。

那么measureChildWithMargins的最后一个参数，也就是LinearLayout在这里传入的这个高度值是用来干嘛的呢？

如果我们追溯下去，就会发现，这个函数最终其实是为了结合父类的MeasureSpec以及child自身的LayoutParams来对子控件测量。而最后传入的值，在子控件测量的时候被添加进去。
```java
	
	 protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    }
```
在官方的注释中，我们可以看到这么一句：
>* @param heightUsed Extra space that has been used up by the parent vertically (possibly by other children of the parent)

事实上，我们在代码中也可以很清晰的看到，在getChildMeasureSpec中，子控件需要把父控件的padding，自身的margin以及一个可调节的量三者一起测量出自身的大小。

那么假如在测量某个子控件之前，weight一直都是0，那么该控件在测量时，需要考虑在本控件之前的总高度，来根据剩余控件分配自身大小。而如果有weight，那么就不考虑已经被占用的控件，因为有了weight，该子控件的高度将会在后面重新赋值。

