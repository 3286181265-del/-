/**
 * 问题I：计算给定n*n矩阵的每一列与给定向量的内积
 * a) 逐列访问元素的平凡算法
 * b) cache优化算法
 */

#include <stdio.h>
#include <stdlib.h>
#include <windows.h>

 // 高精度计时函数
double get_time_ms() {
    return (double)GetTickCount();
}

// 分配矩阵内存
double** allocate_matrix(int n) {
    double** mat = (double**)malloc(n * sizeof(double*));
    for (int i = 0; i < n; i++) {
        mat[i] = (double*)malloc(n * sizeof(double));
    }
    return mat;
}

// 释放矩阵内存
void free_matrix(double** mat, int n) {
    for (int i = 0; i < n; i++) {
        free(mat[i]);
    }
    free(mat);
}

// 初始化测试数据
void init_data(double** matrix, double* vector, int n) {
    for (int i = 0; i < n; i++) {
        vector[i] = i + 1;
        for (int j = 0; j < n; j++) {
            matrix[i][j] = i * n + j + 1;
        }
    }
}

/*a：逐列访问元素的平凡算法
 * 思路：外层循环遍历列，内层循环遍历行
 * 缺点：按列访问导致内存不连续，cache miss率高
 */
double column_major_naive(double** matrix, double* vector, int n) {
    double result = 0.0;

    for (int j = 0; j < n; j++) {          // 遍历每一列
        double col_sum = 0.0;
        for (int i = 0; i < n; i++) {      // 遍历该列的所有行
            col_sum += matrix[i][j] * vector[j];
        }
        result += col_sum;
    }

    return result;
}

/*b：cache优化算法
 * 思路：按行遍历矩阵，利用空间局部性
 * 优点：内存访问连续，cache命中率高
 */
double cache_optimized(double** matrix, double* vector, int n) {
    double* col_sums = (double*)calloc(n, sizeof(double));
    double result = 0.0;

    for (int i = 0; i < n; i++) {          // 按行遍历
        for (int j = 0; j < n; j++) {
            col_sums[j] += matrix[i][j] * vector[j];
        }
    }

    for (int j = 0; j < n; j++) {
        result += col_sums[j];
    }

    free(col_sums);
    return result;
}

int main() {
    printf("========================================\n");
    printf("矩阵每一列与向量的内积\n");
    printf("  a: 逐列访问的平凡算法\n");
    printf("  b: Cache优化算法\n");
    printf("========================================\n\n");

    int sizes[] = { 100, 200, 400, 800 };
    int num_sizes = sizeof(sizes) / sizeof(sizes[0]);

    for (int idx = 0; idx < num_sizes; idx++) {
        int n = sizes[idx];
        printf("--- 矩阵规模 n = %d ---\n", n);

        double** matrix = allocate_matrix(n);
        double* vector = (double*)malloc(n * sizeof(double));
        init_data(matrix, vector, n);

        int repeat = (n <= 200) ? 100 : 20;
        double start, end;
        double naive_time, opt_time;
        double naive_result, opt_result;

        // 测试算法a（平凡算法）
        start = get_time_ms();
        for (int r = 0; r < repeat; r++) {
            naive_result = column_major_naive(matrix, vector, n);
        }
        end = get_time_ms();
        naive_time = (end - start) / repeat;

        // 测试算法b（cache优化算法）
        start = get_time_ms();
        for (int r = 0; r < repeat; r++) {
            opt_result = cache_optimized(matrix, vector, n);
        }
        end = get_time_ms();
        opt_time = (end - start) / repeat;

        // 验证结果正确性
        printf("  结果验证: %s\n",
            (naive_result == opt_result) ? "通过" : "失败");
        printf("  a 平凡算法: %.4f ms\n", naive_time);
        printf("  b 优化算法: %.4f ms\n", opt_time);
        printf("  加速比: %.2fx\n\n", naive_time / opt_time);

        free(vector);
        free_matrix(matrix, n);
    }

    printf("========================================\n");
    return 0;
}





/**
 * 问题II：计算n个数的和
 * a) 逐个累加的平凡算法（链式）
 * b) 超长量优化算法（指令级并行）
 *    - 两路链式累加
 *    - 递归算法（两两相加）
 */

#include <stdio.h>
#include <stdlib.h>
#include <windows.h>

 // 高精度计时函数
double get_time_ms() {
    return (double)GetTickCount();
}

// 初始化测试数据
void init_array(double* arr, int n) {
    for (int i = 0; i < n; i++) {
        arr[i] = i + 1;
    }
}

/*a：逐个累加的平凡算法（链式）
 * 思路：依次将每个元素累加到sum变量
 * 缺点：每条加法依赖前一条结果，无法利用指令级并行
 */
double chain_addition(double* arr, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; i++) {
        sum += arr[i];
    }
    return sum;
}

/*b-1：两路链式累加（指令级并行）
 */
double two_way_ilp(double* arr, int n) {
    double sum1 = 0.0, sum2 = 0.0;
    int i = 0;

    for (i = 0; i < n - 1; i += 2) {
        sum1 += arr[i];
        sum2 += arr[i + 1];
    }
    for (; i < n; i++) {
        sum1 += arr[i];
    }

    return sum1 + sum2;
}

/*b-2：递归算法（两两相加）
 */
double recursive_pairwise(double* arr, int n) {
    if (n == 0) return 0.0;
    if (n == 1) return arr[0];

    int half = n / 2;
    double left_sum = recursive_pairwise(arr, half);
    double right_sum = recursive_pairwise(arr + half, n - half);

    return left_sum + right_sum;
}

/*b-3：循环实现的两两相加（避免递归开销）
 */
double pairwise_reduction(double* arr, int n) {
    if (n == 0) return 0.0;

    double* work = (double*)malloc(n * sizeof(double));
    for (int i = 0; i < n; i++) {
        work[i] = arr[i];
    }

    int size = n;
    while (size > 1) {
        int new_size = (size + 1) / 2;
        for (int i = 0; i < new_size; i++) {
            if (2 * i + 1 < size) {
                work[i] = work[2 * i] + work[2 * i + 1];
            }
            else {
                work[i] = work[2 * i];
            }
        }
        size = new_size;
    }

    double result = work[0];
    free(work);
    return result;
}

int main() {
    printf("========================================\n");
    printf("问题II：计算n个数的和\n");
    printf("  II-a: 逐个累加的平凡算法（链式）\n");
    printf("  II-b: 超长量优化算法（指令级并行）\n");
    printf("========================================\n\n");

    int sizes[] = { 10000, 50000, 100000, 500000, 1000000 };
    int num_sizes = sizeof(sizes) / sizeof(sizes[0]);

    for (int idx = 0; idx < num_sizes; idx++) {
        int n = sizes[idx];
        printf("--- 数组规模 n = %d ---\n", n);

        double* arr = (double*)malloc(n * sizeof(double));
        init_array(arr, n);

        // 根据规模调整重复次数
        int repeat = (n <= 100000) ? 500 : (n <= 500000) ? 100 : 20;

        double start, end;
        double time_chain, time_two_way, time_recursive, time_pairwise;
        double result_chain, result_two_way, result_recursive, result_pairwise;

        // 测试算法II-a（链式累加）
        start = get_time_ms();
        for (int r = 0; r < repeat; r++) {
            result_chain = chain_addition(arr, n);
        }
        end = get_time_ms();
        time_chain = (end - start) / repeat;

        // 测试算法II-b-1（两路链式累加）
        start = get_time_ms();
        for (int r = 0; r < repeat; r++) {
            result_two_way = two_way_ilp(arr, n);
        }
        end = get_time_ms();
        time_two_way = (end - start) / repeat;

        // 测试算法II-b-2（递归两两相加）
        start = get_time_ms();
        for (int r = 0; r < repeat; r++) {
            result_recursive = recursive_pairwise(arr, n);
        }
        end = get_time_ms();
        time_recursive = (end - start) / repeat;

        // 测试算法II-b-3（循环pairwise归约）
        start = get_time_ms();
        for (int r = 0; r < repeat; r++) {
            result_pairwise = pairwise_reduction(arr, n);
        }
        end = get_time_ms();
        time_pairwise = (end - start) / repeat;

        // 验证结果正确性（所有算法结果应相等）
        int valid = (result_chain == result_two_way &&
            result_chain == result_recursive &&
            result_chain == result_pairwise);
        printf("  结果验证: %s (和 = %.0f)\n", valid ? "通过" : "失败", result_chain);

        printf("  II-a 链式累加:      %.4f ms\n", time_chain);
        printf("  II-b 两路累加:      %.4f ms (加速比 %.2fx)\n",
            time_two_way, time_chain / time_two_way);
        printf("  II-b 递归两两相加:  %.4f ms (加速比 %.2fx)\n",
            time_recursive, time_chain / time_recursive);
        printf("  II-b 循环归约:      %.4f ms (加速比 %.2fx)\n\n",
            time_pairwise, time_chain / time_pairwise);

        free(arr);
    }

    printf("========================================\n");
    return 0;
}
