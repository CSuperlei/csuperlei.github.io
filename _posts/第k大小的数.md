# 第k大小的数

### 思想

* 快速选择算法：每次快排完，如果当前left区间个数 s(l) >= k 则只需要递归左半边就可以，如果left区间个数s(l) < k, 则递归右半边的数，右边第k - s(l)的数恰好是整个数列第k大的数。

* 时间复杂度 n(1 + 1/2 + 1/4 + ...) >= n * 2 时间复杂度O(2n)

### 特点


```c++

#include <iostream>
#include <algorithm>

using namespace std;

const int N = 100010;
int q[N];

// 快速选择算法
int quick_sort(int q[], int l, int r, int k)
{
    if (l >= r) return q[l];

    int x = q[l + r >> 1];
    int i = l - 1;
    int j = r + 1;

    while (i < j)
    {
        do{
            i ++;
        }while (q[i] < x);

        do {
            j --;
        }while (q[j] > x);

        if (i < j) swap(q[i], q[j]);
    }

    int sl = j - l + 1;
    
    if (k <= sl) 
        // k 控制了递归的方向，由于第k大的数一定存在，所以最后一定可以递归到第k大的数。当l == r 的时候就是第k大的数
        return quick_sort(q, l, j, k);
    else 
        return quick_sort(q, j + 1, r, k - sl);
}

int main()
{
    int n, k;
    cin >> n >> k;
    for (int i = 0; i < n; i++) cin >> q[i];
    cout << quick_sort(q, 0, n - 1, k);
    return 0;
}

```

