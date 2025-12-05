---
title: "C++ の std::minmax_element は速いのか"
emoji: "📊"
type: "tech"
topics: ["cpp", "simd", "最適化", "高速化"]
published: false
---

配列の最小値と最大値を見つける操作を行う際、配列の全要素に対して `std::min`/`std::max` を取っていく素朴な実装が思いつくかもしれません。

```cpp
auto mn = arr[0];
auto mx = arr[0];
for (size_t i = 1; i < arr.size(); ++i) {
    mn = std::min(mn, arr[i]);
    mx = std::max(mx, arr[i]);
}
```

あるいは、`std::min_element`/`std::max_element` を使う実装かもしれません。

```cpp
auto mn = *std::min_element(arr.begin(), arr.end());
auto mx = *std::max_element(arr.begin(), arr.end());
```

しかし、C++ には `std::minmax_element` という関数があり、これを使うと一発で最小・最大の両方を得られます。
この関数は、与えられた範囲の中で最小の値、最大の値それぞれへのイテレータを組にした `std::pair` を返します。

```cpp
auto [mn_it, mx_it] = std::minmax_element(arr.begin(), arr.end());
```

さらに、この `std::minmax_element` のリファレンス [^1] を読んでみると、探索する範囲の要素数を $N$ として高々 $\left\lfloor \frac{3}{2}(N-1) \right\rfloor$ 回の比較を行うと書かれていることがわかります。

素朴な実装では、ループの各ステップで `std::min` と `std::max` による 2 回の比較を行うため、全体で $2(N-1)$ 回の比較が必要になります。そのため、`std::minmax_element` では比較回数を減らすための工夫がされていることがわかります。

[^1]: https://cppreference.com/w/cpp/algorithm/minmax_element.html

## `std::minmax_element` のアルゴリズム

`std::minmax_element` が比較回数を減らせているのは、そのアルゴリズムに由来します。
要素をペアにして比較し、小さい方を用いて最小値を、大きい方を用いて最大値を更新するという手法を用います。

![std::minmax_element のアルゴリズムを図示したもの](/images/std-minmax-speed/minmax_element.png)

これにより、25% 程度の比較回数の削減が達成されるようです。

実装例としては、[cpprefjp の `minmax_element` のページ](https://cpprefjp.github.io/reference/algorithm/minmax_element.html)にあるコードが、日本語のコメント付きで参考になります。

```cpp
template <class ForwardIterator, class Compare>
std::pair<ForwardIterator, ForwardIterator>
minmax_element(ForwardIterator first, ForwardIterator last, Compare comp)
{
  // 結果用オブジェクト
  std::pair<ForwardIterator, ForwardIterator> result(first, first);

  // イテレータ範囲の要素数が 0 か 1 だったら、そのまま result を返す
  if (first != last && ++first != last) {
    // 最初の 2 個を比較して、小さい方を .first に、大きい方を .second に設定
    if (comp(*first, *result.first))
      result.first = first;
    else
      result.second = first;

    // 残りの要素を 2 個ずつ繰り返し
    while (++first != last) {
      ForwardIterator prev = first;

      // 残りの要素が 1 個しか無かったら、.first と .second の両方の要素と比較して、
      // 必要に応じて結果を更新後、ループを抜ける
      if (++first == last) {
        if (comp(*prev, *result.first))
          result.first = prev;
        else if (!comp(*prev, *result.second))
          result.second = prev;
        break;
      }

      // 残りの要素が 2 個以上あれば、まずその 2 個を比較してから、小さい方を .first と、
      // 大きい方を .second と比較して、必要に応じて結果を更新
      if (comp(*first, *prev)) {
        if (comp(*first, *result.first))
          result.first = first;
        if (!comp(*prev, *result.second))
          result.second = prev;
      } else {
        if (comp(*prev, *result.first))
          result.first = prev;
        if (!comp(*first, *result.second))
          result.second = first;
      }
    }
  }
  return result;
}
```

（https://cpprefjp.github.io/reference/algorithm/minmax_element.html より一部引用）

## 本当に std::minmax_element は速いのか

比較回数では素朴な実装よりも `std::minmax_element` のほうが定数倍で有利であることは明らかです。
しかし、実装はかなり複雑そうに見えます。

特に、比較関数として `int` 同士の `<` のような単純なものが渡された場合はどうでしょうか。
現代のコンパイラは単純なループをかなり上手に自動ベクトル化します。
`min` だけ、`max` だけを回すループは、SIMD 命令へ落ちやすいように思えます。メモリアクセス帯域がボトルネックになっている場合には「二回走査だけど速い」という結果になることもあるのではないでしょうか。
対して `std::minmax_element` のペア処理は、分岐やデータ依存が増え、自動ベクトル化が入りにくいように思えます。

## 実際にベンチマークを取ってみる

実際に検証してみることにしました。

:::details 検証に用いたコード

```cpp
#include <algorithm>
#include <chrono>
#include <cstdint>
#include <iostream>
#include <random>
#include <string_view>
#include <vector>

volatile int tmp;  // 結果を逃がすため

template <class F>
void bench(std::string_view name, F f, const std::vector<int>& arr, int iteration) {
    using namespace std::chrono;

    f(arr);  // warm up

    const auto start = high_resolution_clock::now();
    for (int i = 0; i < iteration; ++i) {
        auto [mn, mx] = f(arr);
        tmp ^= mn;
        tmp ^= mx;
    }
    const auto end = high_resolution_clock::now();
    const auto us = duration_cast<microseconds>(end - start).count();

    std::cout << name << ": " << static_cast<double>(us) / iteration << " us / call\n";
}

// manual (1-pass)
std::pair<int, int> minmax_manual(const std::vector<int>& arr) {
    int mn = arr[0];
    int mx = arr[0];
    for (size_t i = 1; i < arr.size(); ++i) {
        mn = std::min(mn, arr[i]);
        mx = std::max(mx, arr[i]);
    }
    return {mn, mx};
}

// min_element + max_element (2-pass)
std::pair<int, int> minmax_two_pass(const std::vector<int>& arr) {
    auto mn = *std::min_element(arr.begin(), arr.end());
    auto mx = *std::max_element(arr.begin(), arr.end());
    return {mn, mx};
}

// minmax_element
std::pair<int, int> minmax_std(const std::vector<int>& arr) {
    auto [mn_it, mx_it] = std::minmax_element(arr.begin(), arr.end());
    return {*mn_it, *mx_it};
}

int main() {
    int n;
    std::cin >> n;
    const int Iteration = 200;

    std::vector<int> arr(n);
    std::random_device seed_gen;
    std::uint32_t seed = seed_gen();
    std::mt19937 engine(seed);
    std::uniform_int_distribution<int> dist(-1'000'000, 1'000'000);

    for (auto& x : arr) {
        x = dist(engine);
    }

    bench("manual                   ", minmax_manual, arr, Iteration);
    bench("min_element + max_element", minmax_two_pass, arr, Iteration);
    bench("minmax_element           ", minmax_std, arr, Iteration);

    return 0;
}
```

:::

| Column1 | Column2 | Column3 |
| ------- | ------- | ------- |
| Item1   | Item1   | Item1   |

## まとめ
