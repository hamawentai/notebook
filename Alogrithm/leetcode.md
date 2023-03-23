

## 3. 无重复字符的最长字串

窗口滑动条件

```cpp
int lengthOfLongestSubstring(string s) {
        set<char> st;
        int k=0,l=0;
        for(int i=0; i<s.size(); ++i) {
           // 保证窗口中的元素都不重复
            while(st.find(s[i]) != st.end()) {
                st.erase(s[l++]);
            }
            st.insert(s[i]);
            k = max(k, i-l+1);
        }
        return k;
    }
```

## 5. 最长回文子串

```cc
    string longestPalindrome(string s) {
        int k=s.size();
        vector<vector<int>> dp(k, vector<int>(k, 0));
        int l=0,x=1;
        for(int i=0; i<k; ++i) dp[i][i] = 1;
        for(int i=0; i<k; ++i) {
            for(int j=i-1; j>=0; j--) {
                if(s[i] == s[j]) {
                    if(i == j+1) {
                        dp[j][i] = 1;
                    } else {
                        dp[j][i] = dp[j+1][i-1];
                    }
                }
                if(dp[j][i] && x < i-j+1) {
                    x = i-j+1;
                    l = j;
                }
            }
        }
        return s.substr(l, x);
    }
```



## 7. 整数反转

```cc
int reverse(int x) {
        int res=0;
        while(x) {
          	// 溢出判断
            if(res>INT_MAX/10 || res<INT_MIN/10) return 0;
            res = res*10 + x%10;
            x /= 10;
        }
        return res;
    }
```



## 8. 字符串转换整数

## 11. 盛最多水的容器*

```cc
int maxArea(vector<int>& h) {
        int l=0, r=h.size()-1, x=0;
        while(l<r) {
            int t = r - l;
          // 那边小就移动那边，盛水量由短板决定，移动大端只会使得盛水量减少
            if(h[l] < h[r]) {
                t *= h[l++];
            }else {
                t *= h[r--];
            }
            x = max(t, x);
        }
        return  x;
    }
```



## 15. 三数之和*

```cc
vector <vector<int>> threeSum(vector<int> &a) {
        int k = a.size();
        sort(a.begin(), a.end());
        vector <vector<int>> res;
        for (int i = 0; i < k; ++i) {
            if (i != 0 && a[i] == a[i - 1]) continue;
            int l = i + 1, r = k - 1;
            while (l < r) {
                while (l != i + 1 && l < r && a[l] == a[l - 1]) ++l;
                if (l >= r) break;
                int t = a[i] + a[l] + a[r];
                if (t == 0) {
                    res.push_back({a[i], a[l], a[r]});
                    r--;
                    l++;
                } else if (t > 0) r--;
                else l++;
            }
        }
        return res;
    }
};
```



## 24. 两两交换链表节点



## 28. 找出字符串中第一个匹配项的下标(kmp*)

## 29. 两数相除*

## 31. 下一个排列

## 33. 搜索旋转排序数组*

## 34. 在排序数组中查找元素的第一个和最后一个位置*

## 45. 跳跃游戏 II*

## 61. 旋转链表*

## 73. 矩阵置零

## 75. 颜色分类

## 92. 反转链表 II*

## 97. 交错字符串









