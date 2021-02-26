---
title: Android导航栏实现之TabLayout+ViewPager
date: 2019-11-07 14:11:18
tags: 晴(立冬前一天)
---

# Android导航栏实现

## 1. TabLayout

    <com.google.android.material.tabs.TabLayout
        android:id="@+id/tl_personal_menu"
        android:layout_width="match_parent"
        android:layout_height="@dimen/qb_px_195"
        app:layout_constraintTop_toBottomOf="@+id/rl_personal_message"
        app:tabBackground="@color/white"
        app:tabIndicatorColor="@color/green"
        app:tabIndicatorHeight="@dimen/qb_px_10"
        app:tabMode="scrollable"
        app:tabSelectedTextColor="@color/red"
        app:tabTextAppearance="@android:style/TextAppearance.Holo.Medium"
        app:tabTextColor="@color/black">

    </com.google.android.material.tabs.TabLayout>

可能需要在build.gradle中添加依赖，但不知道添加啥，关系太乱了

## 2. ViewPager

**ViewPager的布局和TabLayout是并列关系，并不是将ViewPager放在TabLayout中，TabLayout只是用来存放导航栏Tabs的**

	<androidx.viewpager.widget.ViewPager
        android:id="@+id/vp_personal_list"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintTop_toBottomOf="@+id/tl_personal_menu" />

## 3. PagerAdapter

**ViewPager的Adapter，注意重写方法：**

    @Override
    public CharSequence getPageTitle(int position) {
        return null;                          //页卡标题
    }

## 4. TabLayout与ViewPager相关联

	v.tl_personal_menu.setupWithViewPager(v.vp_personal_list)

**4-1 关联前，自已可以利用如下方法设置导航栏样式：**

    if (personalTabImgs.size == personalTabTxts.size) {
        for (index in personalTabImgs.indices) {
            val tabView = LayoutInflater.from(context)
                 .inflate(R.layout.tab_personal, v.tl_personal_menu, false)
            tabView.iv_tab_personal.setImageResource(personalTabImgs[index])
            tabView.tv_tab_personal.text = personalTabTxts[index]

            val tab = v.tl_personal_menu.newTab()  // 重点是这里的newTab方法
			// tab.text = personalTabTxts[index]
			// tab.icon = context!!.resources.getDrawable(personalTabImgs[index], null)
            tab.customView = tabView

            v.tl_personal_menu.addTab(tab)
        }
    }

**4-2 关联后，之前设置的导航栏样式将被PagerAdapter中的getPageTitle方法返回的字符串所顶替，就是说之前设置的有图片有文字的导航栏Tab会变成只有文字的。**

**4-3 想要在关联后导航栏Tab也是自定义的样式，需要在PagerAdapter中写一个自己的方法返回View对象，例如：**

	// 在自己的PagerAdapter中写
    @SuppressLint("InflateParams")
    public View getPageTitleView(int position) {
        View tabView = LayoutInflater.from(context)
                .inflate(R.layout.tab_personal, null);
        ImageView ivTab = tabView.findViewById(R.id.iv_tab_personal);
        ivTab.setImageResource(personalTabImgs[position]);
        TextView tvTab = tabView.findViewById(R.id.tv_tab_personal);
        tvTab.setText(personalTabTxts[position]);
        return tabView;
    }
	
**然后在关联之后利用for循环做如下设置，即使在关联后再执行一次 `4-1` 中的自定义导航栏设置，也是错误的，newTabs只会在Adapter决定的导航Tabs之后继续添加新建的这些Tabs**

    //设置自定义tab
    for (i in 0 until v.tl_personal_menu.tabCount) {
        val tab = v.tl_personal_menu.getTabAt(i)  // 重点是这里的getTabAt方法
        tab!!.customView = adapter.getPageTitleView(i)
    }


附: 图片和文字自定义的数组

    private var personalTabImgs: IntArray =
        intArrayOf(R.drawable.avater1, R.drawable.avater2, R.drawable.avater3, R.drawable.avater4)
    private var personalTabTxts: Array<String> = arrayOf("tab one", "tab two", "tab three", "tab four")	