基础算法相关
1. 编程题：按树的结构打印数字。 树的基本遍历
2. 编程题：输入两个16进制数字，打印十六进制之和。
3. 1，2，5，10四种面值的人民币，输入一个数值输出多少种组合方式  39.组合总和
```go
// 39.组合总和
// 标准的回溯
func combinationSum2(candidates []int, target int) [][]int {
	sort.Ints(candidates)
	n := len(candidates)
	ret := [][]int{}

	var helper func(index, tmpSum int, tmp []int)
	helper = func(index, tmpSum int, tmp []int) {
		if tmpSum > target || index == n {
			return
		}
		if tmpSum == target {
			ret = append(ret, tmp)
			return
		}
		for j:=index; j<n; j++ {
			if tmpSum + candidates[j] > target {
				break
			}
			helper(j, tmpSum+candidates[j], append(tmp, candidates[j]))
		}
	}
	helper(0, 0, []int{})
	return ret
}
```


4. 36进制的运算（算法coding）
5. 二叉树相关（层次遍历, 求深度, 求两个节点距离, 翻转二叉树, 前中后序遍历）
    + 层次遍历：BFS
    + 求深度：DFS
    + 求两个节点的距离：找到最近的公共祖先LCA算法，再计算2个深度-2*(公共祖先的深度)
    + 翻转二叉树：156.上下翻转二叉树
```go
// 156.翻转二叉树
/* 根据题目描述，树中任何节点的右子节点若存在一定有左子节点，因此思路是向左遍历树进行转化；
   规律是：左子节点变父节点；父节点变右子节点；右子节点变父节点。
   对于某节点root，修改root.left，root.right之前，需要将三者都存下来：
   root.left是下一轮递归的主节点；
   root是下一轮递归root的root.right；
   root.right是下一轮递归root的root.left。
   返回parent。
*/
func upsideDownBinaryTree(root *TreeNode) *TreeNode {
    var parent *TreeNode
	var parentRight *TreeNode

	for root != nil {
		rootLeft := root.Left
		root.Left = parentRight
		parentRight = root.Right
		root.Right = parent
		parent = root
		root = rootLeft
	}
	return parent
}

// 寻找最近公共祖先
func findLCA(root *TreeNode, n1, n2 int) *TreeNode {
    if root == nil {
        return root
    }
    if root.Val == n1 || root.Val == n2 {
        return root
    }
    // 分别在左右两个子树查找
    l := findLCA(root.Left, n1, n2)
    r := findLCA(root.Right, n1, n2)
    if l != nil && r != nil {
        // 说明两节点在左右两个子树，当前节点就是LCA
        return root
    }
    if l != nil {
        return l
    }
    return r
}
```

   + 前中后序遍历：递归(简单)&迭代(栈), 105.从前序和中序构造二叉树 106.从中序和后序构造二叉树
```go
// 105.从前序和中序构造二叉树
// 思路：前序遍历将树分为左、右子树，然后依次递归下去即可
func buildTree(preorder []int, inorder []int) *TreeNode {
	if len(preorder) <= 0 {
		return nil
	}
	root := &TreeNode{Val:preorder[0]}
	// 找到root在inorder中的位置，因为root将树划分为左、右子树
	loc := 0
	for i := range inorder {
		if inorder[i] == preorder[0] {
			loc = i
			break
		}
	}
	root.Left = buildTree(preorder[1:loc+1], inorder[:loc])
	root.Right = buildTree(preorder[loc+1:], inorder[loc+1:])
	return root
}

// 106.从中序和后序构造二叉树
func buildTree(inorder []int, postorder []int) *TreeNode {
	if len(inorder) <= 0 {
		return nil
	}
	postLen := len(postorder)
	root := &TreeNode{Val:postorder[postLen-1]}
	// 找到root在inorder中的位置，因为root将树划分为左、右子树
	loc := 0
	for i := range inorder {
		if inorder[i] == postorder[postLen-1] {
			loc = i
			break
		}
	}
	root.Left = buildTree(inorder[:loc], postorder[:loc])
	root.Right = buildTree(inorder[loc+1:], postorder[loc+1:len(inorder)-1])
	return root
}
```
   

链表相关（插入节点. 链表逆置. 使用链表进行大数字的加减，双向链表实现队列. 寻找链表中的环）
+ 链表逆置：
```go
// 链表原地逆置
func reverse(head *ListNode) *ListNode {
	if head == nil {
		return head
	}
	tempHead := &ListNode{Next:head}

	p := tempHead
	i := tempHead.Next
	if i.Next == nil {
		// 链表只有1个元素
		return i
	}
	for i != nil {
		if i.Next == nil {
			break
		}
		j := i.Next
		i.Next = j.Next
		j.Next = p.Next
		p.Next = j
	}

	return tempHead.Next
}
```
+ 寻找链表中的环
142.环形链表
```go
// 寻找链表中的环 
// 思路：快慢指针
func detectCycle2(head *ListNode) *ListNode {
	slow, fast := head, head
	for  {
		if fast == nil || fast.Next == nil {
			return nil
		}
		slow = slow.Next
		fast = fast.Next.Next
		if fast == slow {
			break
		}
	}
	fast = head
	for fast != slow {
		fast = fast.Next
		slow = slow.Next
	}
	return fast
}
```

6. 堆（大量数据中寻找最大N个数字几乎每次都会问，还有堆在插入时进行的调整） 215.数组中的第K大的元素
```go
// 堆排序-最大堆
func adjust(nums []int, parent, l int)  {
	for {
		child := 2*parent+1
		if child > l {
			break
		}
		if child < l && nums[child+1] > nums[child] {
			child++
		}
		if nums[parent] >= nums[child] {
			break
		}
		nums[parent], nums[child] = nums[child], nums[parent]
		parent = child
	}
}

func insert(nums []int, v int)  {
	nums = append(nums, v)
	// 自底向上调整
	i := len(nums) - 1
	for ;nums[i/2] > v; i/=2 {
		nums[i] = nums[i/2]
	}
	nums[i] = v
}

// 最大堆
func heapSort(nums []int)  {
	n := len(nums)
	// 从最后一个非叶子节点开始 自底向上构建
	for i:=n/2-1; i>=0; i-- {
		adjust(nums, i, n)
	}

	// 排序, 每次将最后一个元素和第一个元素交换
	// 然后调整堆
	for i:=n-1; i>=0; i++ {
		nums[0], nums[i] = nums[i], nums[0]
		adjust(nums, 0, i)
	}
}

```

寻找最大N个数字：设置大小为N的最小堆，遍历数据，每次往堆里插入元素，上浮调整。当堆超过大小时，删除堆顶元素。
这样遍历m条数据，删除堆顶元素m-N次，这样，最后堆顶的元素就是原数据中第N大的

7. 排序（八大排序，各自的时间复杂度. 排序算法的稳定性。快排几乎每次都问）
```go
// 1.冒泡 O(N^2) 稳定
// 依次比较相邻两个元素，若前>后，交换，直到最后一个元素即为最大；
// 然后从头开始，到倒数第二个即为最大元素，依次类推
func bubbleSort(nums []int, start, end int)  {
	noChange := true
	for i:=start; noChange == true && i<end; i++ {
		noChange = true
		for j:=i+1; j<end-i; j++ {
			if nums[j] < nums[j-1] {
				nums[j], nums[j-1] = nums[j-1], nums[j]
				noChange = false
			}
		}
	}
}

// 2.选择排序 O(N^2) 不稳定
// 首先初始化最小元素为首元素，依次遍历待排序数列，若遇到小于最小元素，
// 则刷新最小索引为该较小元素的位置，直到尾元素，并将最小元素与首元素交换，以此类推
func selectSort(nums []int, start, end int)  {
	for i:=start; i<end; i++ {
		minIndex := i
		for j:=i+1; j<end; j++ {
			if nums[j] < nums[minIndex] {
				minIndex = j
			}
		}
		if minIndex != i {
			nums[i], nums[minIndex] = nums[minIndex], nums[i]
		}
	}
}

// 3.快速排序 O(NlogN) 不稳定
// 选一基准元素，依次将剩余元素中小于该基准元素的值放左侧，大于的放右侧
// 然后，对基准元素的前半部分和后半部分做同样的处理
// 依次类推，直到各子序列只剩一个元素，即排序完成
func quickSort(nums []int, start, end int) {
	if start <= end {
		return
	}
	i, j := start, end
	p := start
	std := nums[start]
	for i<=j {
		// 从右边找到第一个小于基准元素的
		for j>=p && nums[j]>=std {
			j--
		}
		// 将其放到左边
		if j>=p {
			nums[p] = nums[j]
			p = j
		}
		fmt.Printf("%v %v %v %v\n", nums, p, i, j)

		// 从左边找到第一个大于基准元素的
		for i<=p && nums[i]<=std {
			i++
		}
		// 将其放到右边
		if i<=p {
			nums[p] = nums[i]
			p = i
		}
		fmt.Printf("%v %v %v %v\n", nums, p, i, j)
	}

	// 将基准元素放入合适的位置
	nums[p] = std
	//fmt.Printf("%v %v %v %v\n", nums, p, i, j)
	if p-start > 1 {
		// 递归处理其左侧
		quickSort(nums, start, p-1)
	}
	if end - p > 1 {
		// 递归处理其右侧
		quickSort(nums, p+1, end)
	}
}

// 4. 插入排序 O(N^2) 稳定
// 数组前面看成有序，依次将后面的无序数列元素插入到前面的有序数列中，初始状态有序数列为首元素
func insertSort(nums []int, start, end int)  {
	for i:=start+1; i<end; i++ {
		j := i-1
		for ; j>=start; j-- {
			if nums[j] <= nums[i] {
				break
			}
		}

		if j != i-1 {
			tmp := nums[i]
			for k:=i; k>j+1; k-- {
				nums[k] = nums[k-1]
			}
			nums[j+1] = tmp
		}
	}
}

// 5. 希尔排序 O(N^3/2) 不稳定
// 插入排序的改进版，减少数据的移动次数，在初始序列时取得较大的步长，通常取序列长度的一半
// 略


// 6. 归并排序 O(NlogN) 稳定
// 递归&分治 排序整个数列如同排序两个有序数列，依次执行这个过程直到排序末端的两个元素
func mergeSort(nums []int, start, end int) {
	if start >= end -1 {
		return
	}
	mid := (start+end)/2
	mergeSort(nums, start, mid)
	mergeSort(nums, mid, end)
	mergeSortInOrder(nums, start, mid, end)
}

func mergeSortInOrder(nums []int, start, middle, end int)  {
	buf := make([]int, end-start)
	l, r := start, end
	index := 0
	for l<middle && r<middle {
		if nums[l] <= nums[r] {
			buf[index] = nums[l]
		} else {
			buf[index] = nums[r]
		}
		index++
		l++
		r++
	}

	for l < middle {
		buf[index] = nums[l]
		l++
		index++
	}

	for r < middle {
		buf[index] = nums[r]
		r++
		index++
	}

	index = 0
	for i:=start; i<end; i++ {
		nums[i] = buf[index]
		index ++
	}
}

// 堆排序见上
```

8. 二分查找（一般会深入，如寻找数组总和为K的两个数字）
     两个栈实现队列。
     
+ 寻找数组总和为K的两个数组：1.两数之和
```go
// 1.两数之和 哈希表
func twoSum(nums []int, target int) []int {
	size := len(nums)
	if size <= 1 {
		return []int{}
	}
	m := map[int]int{}
	for i, num := range nums {
		another := target - num
		if v, ok := m[another]; ok {
			return []int{i, v}
		}
		m[num] = i
	}
	return []int{}
}
```
9. 图（深度广度优先遍历. 单源最短路径. 最小生成树）
      动态规划问题。
+ 最小生成树
```go
// 最小生成树 Kruskal & Prime算法
+ Kruskal算法:
1. 把所有边按代价从小到大排序； 
2. 把n个顶点看成独立的n棵树组成的森林； 
3. 按权值从小到大选择边，所选的边连接的两个顶点ui,viui,vi,应属于两颗不同的树，则成为最小生成树的一条边，并将这两颗树合并作为一颗树。 
4. 重复(3),直到所有顶点都在一颗树内或者有n-1条边为止。

+ Prime算法：算法从某一个顶点s开始
1. 图的所有顶点集合为VV；初始令集合u={s},v=V−uu={s},v=V−u;
2. 在两个集合u,vu,v能够组成的边中，选择一条代价最小的边(u0,v0)(u0,v0)，加入到最小生成树中，并把v0v0并入到集合u中。
3. 重复上述步骤，直到最小生成树有n-1条边或者n个顶点为止。
```
      
10. 深入
     红黑树性质
      分治法和动态规划的区别
       计算时间复杂度
       二叉树和哈希表查找的时间复杂度
11. 16进制加法或者乘法，不可以转成10进制
12. 比较两个字符串大小，字符串可能存在压缩存储，比如3a与aaa相等   
13. 两个字符串相同部分的最长值
```go
// 思路1：暴力求解，依次比较，略
// 思路2：动态规划 dp[i][j]表示以s1[i], s2[j]结尾的公共子串的长度
func longestCommonSubString(s1, s2 string) int {
	size1, size2 := len(s1), len(s2)
	if size1 == 0 || size2 == 0 {
		return 0
	}
	dp := make([][]int, size1)
	for i:=0; i<size1; i++ {
		dp[i] = make([]int, size2)
	}
	longest := 1

	for i:=0; i<size2; i++ {
		if s1[0] == s2[i] {
			dp[0][i] = 1
		} else {
			dp[0][i] = 0
		}
	}

	for i:=1; i<size1; i++ {
		if s1[i] == s2[0] {
			dp[i][0] = 1
		} else {
			dp[i][0] = 0
		}

		for j:=1; j<size2; j++ {
			if s1[i] == s2[j] {
				dp[i][j] = dp[i-1][j-1] + 1
			}
		}
	}

	// 最大的dp即为最长
	for i:=0; i<size1; i++ {
		for j:=0; j<size2; j++ {
			if dp[i][j] > longest {
				longest = dp[i][j]
				start1 := i+1-longest
				start2 := j+1-longest
				println(start1, start2)
			}
		}
	}
	return longest
}
```
14. 1.在数组里找出现次数大于长度一半的数字，2.找两数之和等于sum。
+ 在数组中找出现次数大于长度一半的数字
```go
// 思路1，排序，找中间，略
// 思路2，哈希计数，略
// 思路3，
更进一步，考虑到这个问题本身的特殊性，我们可以在遍历数组的时候保存两个值：
一个candidate，用来保存数组中遍历到的某个数字；
一个nTimes，表示当前数字的出现次数，其中，nTimes初始化为1。当我们遍历到数组中下一个数字的时候：
func findAppreasMoreThanHalf(nums []int) int {
	size := len(nums)
	if size <= 1 {
		return -1
	}

	ret, count := nums[0], 1
	for i:=1; i<size; i++ {
		if nums[i] == ret {
			count++
		} else {
			count--
		}
		if count == 0 {
			ret = nums[i]
			count = 1
		}
	}
	// 再遍历确认是否超过了一半
	count = 0
	for i:=0; i<size; i++ {
		if nums[i] == ret {
			count ++
		}
	}
	if count > (size / 2) {
		return count
	}
	return -1
}

```
15. 16进制相加相乘
16. 给两个十六进制字符串，做加法，不转成10进制，给个map key从00到ff做字典，value为加的结 果，以上为条件计算。
17. 按照二叉树的结构打印数字，每层节点的数字在同一行
18. 给出一个数字，查找有序队列中该数字出现的次数（二分查找）
```go
// 思路1：遍历，直到找到该数开始计数，到不是该数停止计数，返回总计数，略
// 思路2：既然是有序队列，二分查找降低时间复杂度 
// 1. 直接二分查找
// 2. 改进版二分查找
func getFirst(nums []int, target, start, end int) int {
	if start > end {
		return -1
	}

	mid := (start + end) / 2
	if midItem := nums[mid]; midItem == target {
		if mid > 0 && nums[mid-1] != target || mid == 0 {
			return mid
		} else {
			end = mid - 1
		}
	} else if midItem < target {
		// 在后半段
		start = mid + 1
	} else if midItem > target {
		// 在前半段
		end = mid - 1
	}
	return getFirst(nums, target, start, end)
}

func getLast(nums []int, target, start, end int) int {
	if start > end {
		return -1
	}

	mid := (start + end) / 2
	if midItem := nums[mid]; midItem == target {
		if mid > 0 && nums[mid+1] != target || mid == len(nums)-1 {
			return mid
		} else {
			start = mid - 1
		}
	} else if midItem < target {
		// 在后半段
		start = mid + 1
	} else if midItem > target {
		// 在前半段
		end = mid - 1
	}
	return getLast(nums, target, start, end)
}

func getCount(nums []int, target int) int {
	if len(nums) == 0 {
		return -1
	}
	first, last := getFirst(nums, target, 0, len(nums)-1), getLast(nums, target, 0, len(nums)-1)
	if first == -1 || last == -1 {
		return -1
	}
	return last - first + 1
}

```
19. 给出一个数字串，只交换一次，就可以得到一个新的数字串，该数字串的值比原来的数字串大，并且这个新的数字串的值是所有比原来的数字串大的中的最小的一个
+ 31. 下一个排列
```go
func nextPermutation(nums []int)  {
	// 找到最右边的比后一个小的数
	size := len(nums)
	if size <= 1 {
		return
	}
	dest := -1
	for i:=0; i<size-1; i++ {
		if nums[i] < nums[i+1] {
			dest = i
		}
	}

	if dest == -1 {
		sort.Ints(nums)
		return
	}

	// 找到最右边比dest大的数
	j := dest + 1
	min := nums[j]
	for ;j < size && nums[j] <= min && nums[j] > nums[dest]; {
		min = nums[j]
		j++
	}
	j--

	nums[dest], nums[j] = nums[j], nums[dest]
	// 其实应该是直接翻转nums[k+1:]
	sort.Ints(nums[dest+1:])
}
```
20. 给定一个多行字符串，有多列内容，第一列是path路径，第二列是%百分比值，如何统计出指定的path（全匹配或者部分匹配）的第二列百分比的平均值
21. （1）给一个数字，只允许交换一次，使得交换后的数字比原数字大，并且是所有情况中最小的。（2）01字符串问题。525.连续数组?
23. 将一个有序的有重复元素的单链表变成一个有序的无重复元素的单链表。
```go
// 保留重复的1次 快慢指针
func deleteDuplicates(head *ListNode) *ListNode {
	if head == nil {
		return head
	}
	tmpHead := &ListNode{Next:head}
	slow, fast := head, head.Next

	for fast != nil {
		if slow.Val == fast.Val {
			fast = fast.Next
			slow.Next = fast
		} else {
			slow = slow.Next
			fast = fast.Next
		}
	}
	return tmpHead.Next
}

// 删除所有重复的
func deleteDuplicates(head *ListNode) *ListNode {
    if head == nil || head.Next == nil {
		return head
	}

	tempHead := &ListNode{Next:head}

	left := tempHead
	right := tempHead
	q := tempHead.Next
	lastVal := q.Val

	for ;q != nil; q = q.Next {
		if q.Next != nil && q.Next.Val != lastVal {
			lastVal = q.Next.Val
			left = q
		} else if q.Next != nil && q.Next.Val == lastVal {
			// 一直向右扩展到不相等节点
			right = q.Next
			for right != nil && right.Val == lastVal {
				right = right.Next
			}
			left.Next = right
			q = left
			if q.Next != nil {
				lastVal = q.Next.Val
			}
		}
	}
	return tempHead.Next
}

```
24. 有序链表合并，二叉树翻转，海量64位数字去重。
25. 已知二叉树的前序遍历和中序遍历，求后序遍历
26. 实现加权轮询的负载均衡策略
27. 单例模式的实现？手写一个Double Check单例，排序算法？手写一个归并排序
28. topK问题，K(值. 出现次数)，使用有限的内存？两个十六进制串相加，使用递归？
