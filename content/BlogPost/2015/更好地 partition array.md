很多算法或者题目里面都有 partition array 这步。所谓 partition array，指的就是把元素按某种条件分开，一部分放前面，另一部分放后面。

比如快速排序，这个条件就是“元素是否小于 pivot”，小于放前面，大于放后面。

大部分人写快排，其中 partition 的一步都喜欢这么写：

	def partition(alist, first, last):
		pivot = alist[first]
		
		left = first + 1
		right = last
		
		while left <= right:
			while left <= right and alist[left] <= pivote:
			    left += 1
			
			while left <= right and alist[right] >= pivot:
			    right += 1
			
			if left <= right:
				alist[left], alist[right] = alist[right], alist[left]
				left += 1
				right -= 1
		
		alist[first], alist[right] = alist[right], alist[first]
		return right

弄两根指针 `left`, `right` 分别放在 array 的左端和右端，然后 `left` 右移，`right` 左移，如果能交换就交换元素，直到 `left > right`。

上面是一个比较标准的 partition 实现，也有一些变体，不过凡是使用 left/right 指针的都是一类。这类实现当然没有问题，但是不优雅，而且难记。

哪里不优雅？仔细看就会发现，**代码里有 4 次 `left <= right` 的判断！**而且这 4 次都是必要的。

为什么难记？因为代码多，所以难记！而且稍不注意，最后一步就会写成 `alist[first], alist[left] = alist[left], alist[first]`。

优雅的写法是什么呢？

	def partition(alist, first, last):
		pivot = alist[first]		
		i = first
		
		for j in range(first + 1, last):
			if alist[j] <= pivot:
				i += 1
				alist[i], alist[j] = alist[j], alist[i]
		
		alist[first], alist[i] = alist[i], alist[first]
		return i

短了这么多，更重要的是再也没有重复的 `left <= right` 判断了！这段代码也很好理解，`j` 指针把 array 扫一遍，把 `<= pivot` 的放到前面，`i` 用来记录放的地方。停止的时候，`i` 的位置是 `<= pivot` 部分的最后一个元素的 index。当然，不要忘记把 pivot 和这个元素交换。 

需要注意的地方有两个，首先是初始化：`i = first, j = first + 1`。还有，每次是先 `i += 1`，然后再交换i, j元素。

其实这种区别不仅仅在于快排，所有 partition array 都是一样。比如这题：

>给出一个整数数组nums和一个整数k。划分数组（即移动数组nums中的元素），使得：

>所有小于k的元素移到左边  
>所有大于等于k的元素移到右边    

不优雅的写法这么写：

    int partitionArray(vector<int> &nums, int k) {
        int i = 0, j = nums.size() - 1;
        while (i <= j) {
            while (i <= j && nums[i] < k) i++;
            while (i <= j && nums[j] >= k) j--;
            if (i <= j) {
                int temp = nums[i];
                nums[i] = nums[j];
                nums[j] = temp;
                i++;
                j--;
            }
        }
    }

还是 4 次比较╮(╯▽╰)╭

优雅的写法：

    def partitionArray(nums):        
        i = -1
        
        for j in range(len(nums)):
            if nums[j] < k:
                i = i + 1
                nums[i], nums[j] = nums[j], nums[i]

当然拿Py和C++比是不合适的，但关键是少了 4 次比较，代码就好看多了。  
这一题不存在真实的 pivot，所以我们对快排做了改进，初始化时 `i` 不再是 0，而是 -1。快排的时候，因为第一次交换时 `i = first + 1`，所以初始化 `i=first` 实际上跳过了 pivot 也就是 `array[first]`。这里因为没有需要跳过的元素，所以 `i=-1`，然后 `j` 还是等于 `i + 1` 也就是 0。其它的没有区别。

熟悉没有真实 pivot 的写法之后，不论按何种规则划分，都只改一行就可以。  
比如按照奇偶划分：

    def partitionArray(self, nums):
        n = len(nums)
        i = -1
        
        for j in range(n):
            if nums[j] % 2 == 1:  # changed this line
                i = i + 1
                nums[i], nums[j] = nums[j], nums[i]

其实到这里本来是写完了，但是那天和同学吃饭的时候聊天，又让我意识到优雅写法不光只是优雅而已，有时候只能用它。同学出这题考我：

>单链表排序能不能用快排？

我想了想，没有不能用的理由吧，就说能用。他说，不行，因为是单链表，所以不能用快排。我没反应过来，就问为什么单链表不能用快排？他很不解我居然提出这个问题，说你要移动左右两个指针啊，但是单链表你没法把右边的指针往左移动，所以没法用快排。

于是我瞬间懂了，他只知道快排可以用左右指针来写，却不知道另一种写法。原本我只是觉得第二种写法更优雅，没想到在不知道的情况下，居然会误认为单链表没法用快排。

不只是更优雅，而是更好。