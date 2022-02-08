# optimization

refer [如何优雅地利用c++编程从1乘到20？](https://www.zhihu.com/question/365763395/answer/1033786970)

代码上没什么花好玩的，来搞一些 premature 优化吧。

先用单核SIMD。SSE和AVX好像缺一些64位整数的功能，先用32位整数了，有溢出风险。

## AVX：
```c++
#include <iostream>
#include <immintrin.h>

inline long long reduce_product(__m256i vec) {
    __m256i half = _mm256_mul_epi32(_mm256_shuffle_epi32(vec, 0x31), _mm256_shuffle_epi32(vec, 0x20));
    /* may overflow */
    __m128i quarter = _mm_mul_epi32(_mm256_extracti128_si256(half, 1), _mm256_castsi256_si128(half));
    return _mm_extract_epi64(quarter, 1) * _mm_extract_epi64(quarter, 0);
}

long long factorial(int x) {
    __m256i a = _mm256_set_epi32(8,7,6,5,4,3,2,1);
    __m256i c = _mm256_set1_epi32(1);
    for(int i = 0; i < (x & (~7)); i += 8) {
        c = _mm256_mullo_epi32(c, a);  // may overflow
        a = _mm256_add_epi32(a, _mm256_set1_epi32(8));
    }
    __m256i mask = _mm256_cmpgt_epi32(a, _mm256_set1_epi32(x));
    a = _mm256_blendv_epi8 (a, _mm256_set1_epi32(1), mask);
    c = _mm256_mullo_epi32(c, a);      // may overflow
    return reduce_product(c);
}

int main() {
    std::cout << factorial(20) << std::endl;
}
```

## AVX512：

``` c++
#include <iostream>
#include <immintrin.h>

long long factorial(int x) {
    __m512i a = _mm512_set_epi64(8,7,6,5,4,3,2,1);
    __m512i c = _mm512_set1_epi64(1);
    for(int i = 0; i < (x & (~7)); i += 8) {
        c = _mm512_mullo_epi64(c, a);
        a = _mm512_add_epi64(a, _mm512_set1_epi64(8));
    }
    __mmask8 mask = _mm512_cmpgt_epi64_mask(a, _mm512_set1_epi64(x));
    a = _mm512_mask_mov_epi64(a, mask, _mm512_set1_epi64(1));
    c = _mm512_mullo_epi64(c, a);
    return _mm512_reduce_mul_epi64(c);
}

int main() {
    std::cout << factorial(20) << std::endl;
}

```

再来多线程

## OpenMP

好像就加一句pragma：

``` c++
#include <iostream>

long long factorial(int x) {
    long long res = 1;

    #pragma omp parallel for reduction(* : res)
    for(int i = 1; i <= x; ++i) {
        res *= i;
    }

    return res;
}

int main() {
    std::cout << factorial(20) << std::endl;
}

```

## Thread Building Blocks：

``` c++
#include <iostream>
#include <numeric>
#include <functional>
#include <tbb/parallel_reduce.h>
#include <tbb/iterators.h>
#include <tbb/blocked_range.h>

long long factorial(int x) {
    using iter_t  = tbb::counting_iterator<long long>;
    using range_t = tbb::blocked_range<iter_t>;
    using mul_t   = std::multiplies<long long>;

    return tbb::parallel_reduce(range_t(iter_t(1), iter_t(x + 1)), 1ll, \
        [](const range_t &r, long long init) {
            return std::accumulate(r.begin(), r.end(), init, mul_t());
        },
        mul_t()
    );
}

int main() {
    std::cout << factorial(20) << std::endl;
}

```

最后上GPU

## CUDA + Thrust：
``` c++ 
#include <iostream>
#include <thrust/iterator/counting_iterator.h>
#include <thrust/reduce.h>

long long factorial(int x) {
    thrust::counting_iterator<long long> begin(1), end(x + 1);
    return thrust::reduce(begin, end, 1ll, thrust::multiplies<long long>());
}

int main() {
    std::cout << factorial(20) << std::endl;
}

```
