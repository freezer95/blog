# 排序

### 冒泡排序

```javascript
/**
 * 冒泡排序
 */
function bubbleSort (arr) {
  const max = arr.length - 1 // 外部 for 循环的最大值
  let finished // 标志位
  for (let i = 0; i < max; i++) {
    finished = true
    for (let j = 0; j < max - i; j++) {
      if (arr[j] > arr[j + 1]) {
        swap(arr, j, j + 1)
        finished = false
      }
    }
    if (finished) {
      break
    }
  }
}

function swap (arr, i, j) {
  if (i === j) {
    return
  }
  [arr[i], arr[j]] = [arr[j], arr[i]]
}

// test
const arr = [3, 4, 1, 2]
bubbleSort(arr)
console.log('arr: ', arr)
```

### 快速排序

```javascript
/**
 * 快速排序，平均时间复杂度O(nlogn)
 * 在源数组上排序，只需考虑递归出入栈的空间，空间复杂度为O(logn)
 */
function quickSort (arr, left, right) {
  if (left > right - 1) {
    return
  }
  let pivot = arr[left]
  let mid = left
  for (let i = left + 1; i <= right; i++) {
    if (arr[i] < pivot) {
      ++mid
      swap(arr, mid, i)
    }
  }
  swap(arr, left, mid)
  quickSort(arr, left, mid - 1)
  quickSort(arr, mid + 1, right)
}

function swap (arr, i, j) {
  if (i === j) {
    return
  }
  [arr[i], arr[j]] = [arr[j], arr[i]]
}

// test
const arr = [1, 2, 7, 9, 3, 4, 5, 10, 8]
quickSort(arr, 0, arr.length - 1)
console.log('arr: ', arr)
```
