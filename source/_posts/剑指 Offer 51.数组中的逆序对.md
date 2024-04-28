---
title: 数组中的逆序对
date: 2024-04-29 10:03
tags:
  - 归并排序
categories:
  - 算法
---

```java
/**
 * 剑指 Offer 51.数组中的逆序对
 * <a href="https://leetcode.cn/problems/shu-zu-zhong-de-ni-xu-dui-lcof/description/"/>
 * 此题的关键在于理解归并的思想：
 * 假设有数组[7, 5, 6, 1]
 * 针对最小的四个子数组：[7], [5], [6], [1]
 * 只需要在合并时计算逆序对，即当 [7] 和 [5] 合并时，计算逆序对数量为 1，那么将 [7] 和 [5] 合并为 [5, 7] 就对结果没有影响了，因为已经计算过这两个子数组的逆序对
 * 同理，[6] [1] 这两个子数组也可以合并，同时由于 [6] [1] 都在 [7] [5] 子数组的后面，所以前面的子数组并不关注后面子数组的顺序
 * 所以能够得到 [5, 7] 和 [1, 6] 两个子数组
 * 再分别计算 5, 7 两个数分别对于 1, 6 的逆序对数量，最后加起来便可
 * PS：这里不一定要采用从小到大的顺序，从大到小的顺序合并也可以
 */
public class ReversePairs {

    static class Solution {

        private int count;

        private int[] temp;

        public int reversePairs(int[] nums) {
            if (nums == null || nums.length == 0) {
                return 0;
            }
            this.count = 0;
            this.temp = new int[nums.length];
            sort(nums, 0, nums.length - 1);
            return count;
        }

        public void sort(int[] nums, int start, int end) {
            if (start >= end) {
                return;
            }
            int mid = start + (end - start) / 2;
            sort(nums, start, mid);
            sort(nums, mid + 1, end);
            merge(nums, start, mid, end);
        }

        public void merge(int[] nums, int start, int mid, int end) {
            if (end + 1 - start >= 0) {
                System.arraycopy(nums, start, temp, start, end + 1 - start);
            }

            int right = mid + 1;
            for (int left = start; left <= mid; left++) {
                while (right <= end && nums[left] > nums[right]) {
                    right++;
                }
                count += right - mid - 1;
            }

            int i = start, j = mid + 1;
            for (int index = 0; index <= end; index++) {
                if (i == mid + 1) {
                    nums[index++] = temp[j++];
                } else if (j == end + 1) {
                    nums[index++] = temp[i++];
                } else if (temp[i] > temp[j]) {
                    nums[index++] = temp[j++];
                } else {
                    nums[index++] = temp[i++];
                }
            }
        }
    }

    public static void main(String[] args) {
        int[] nums = new int[3];

        Solution solution = new Solution();
        int result = solution.reversePairs(nums);
        System.out.println(result);
    }

}
```
