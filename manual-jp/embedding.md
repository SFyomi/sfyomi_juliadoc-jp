<!-- start --->

# Embedding Julia
# 埋め込みジュリア

<!-- end --->
<!-- start --->
As we have seen in [Calling C and Fortran Code](@ref), Julia has a simple and efficient way to call functions written in C.
> [CとFortranコードの呼び出し](@ref)で見たように、JuliaはCで書かれた関数を簡単かつ効率的に呼び出す方法を持っています。
<!-- end --->
<!-- start --->
But there are situations where the opposite is needed: calling Julia function from C code.
> しかし、反対が必要な状況があります：CコードからJulia関数を呼び出します。
<!-- end --->
<!-- start --->
This can be used to integrate Julia code into a larger C/C++ project, without the need to rewrite everything in C/C++.
> これは、C/C++のすべてを書き直す必要なしに、Juliaコードをより大きなC/C++プロジェクトに統合するために使用できます。
<!-- end --->
<!-- start --->
Julia has a C API to make this possible.
> Juliaにはこれを可能にするC APIがあります。
<!-- end --->
<!-- start --->
As almost all programming languages have some way to call C functions, the Julia C API can also be used to build further language bridges (e.g. calling Julia from Python or C#).
> ほとんどすべてのプログラミング言語にはC関数を呼び出す方法があるため、Julia C APIを使用してさらに言語ブリッジを構築することもできます(PythonやC＃からJuliaを呼び出すなど)。
<!-- end --->
<!-- start --->

## High-Level Embedding

> ## ハイレベル埋め込み

<!-- end --->
<!-- start --->
We start with a simple C program that initializes Julia and calls some Julia code:
> Juliaを初期化し、Juliaコードを呼び出す簡単なCプログラムから始めます。
<!-- end --->

```c
#include <julia.h>
JULIA_DEFINE_FAST_TLS() // only define this once, in an executable (not in a shared library) if you want fast code.

int main(int argc, char *argv[])
{
    /* required: setup the Julia context */
    jl_init();

    /* run Julia commands */
    jl_eval_string("print(sqrt(2.0))");

    /* strongly recommended: notify Julia that the
         program is about to terminate. this allows
         Julia time to cleanup pending write requests
         and run all finalizers
    */
    jl_atexit_hook(0);
    return 0;
}
```

<!-- start --->
In order to build this program you have to put the path to the Julia header into the include path and link against `libjulia`.
> このプログラムをビルドするには、Juliaヘッダへのパスをインクルードパスに入れ、 `libjulia` とリンクする必要があります。
<!-- end --->
<!-- start --->
For instance, when Julia is installed to `$JULIA_DIR`, one can compile the above test program `test.c` with `gcc` using:
> 例えば、Juliaが `$JULIA_DIR` にインストールされている場合、上記のテストプログラム `test.c` を `gcc` でコンパイルすることができます：
<!-- end --->

```Julia
gcc -o test -fPIC -I$JULIA_DIR/include/julia -L$JULIA_DIR/lib test.c -ljulia $JULIA_DIR/lib/julia/libstdc++.so.6
```

<!-- start --->
Then if the environment variable `JULIA_HOME` is set to `$JULIA_DIR/bin`, the output `test` program can be executed.
> 環境変数 `JULIA_HOME` が `$JULIA_DIR/bin` に設定されていれば、出力 `test` プログラムを実行することができます。
<!-- end --->

<!-- start --->
Alternatively, look at the `embedding.c` program in the Julia source tree in the `examples/` folder.
> あるいは、 `examples/` フォルダのJuliaソースツリーの `embedding.c` プログラムを見てください。
<!-- end --->
<!-- start --->
The file `ui/repl.c` program is another simple example of how to set `jl_options` options while linking against `libjulia`
> `ui/repl.c` ファイルは `libjulia` とのリンク時に `jl_options` オプションを設定する簡単な例です。
<!-- end --->

<!-- start --->
The first thing that has to be done before calling any other Julia C function is to initialize Julia.
> 他のJulia C関数を呼び出す前に最初にしなければならないことは、Juliaを初期化することです。
<!-- end --->
<!-- start --->
This is done by calling `jl_init`, which tries to automatically determine Julia's install location.
> これは `jl_init` を呼び出すことによって行われ、Juliaのインストール場所を自動的に決定しようとします。
<!-- end --->
<!-- start --->
If you need to specify a custom location, or specify which system image to load, use `jl_init_with_image` instead.
> カスタムの場所を指定する必要がある場合や、読み込むシステムイメージを指定する場合は、代わりに `jl_init_with_image` を使用してください。
<!-- end --->

<!-- start --->
The second statement in the test program evaluates a Julia statement using a call to `jl_eval_string`.
> テストプログラムの2番目のステートメントは、 `jl_eval_string` の呼び出しを使用してJuliaステートメントを評価します。
<!-- end --->

<!-- start --->
Before the program terminates, it is strongly recommended to call `jl_atexit_hook`.
> プログラムが終了する前に、 `jl_atexit_hook` を呼び出すことを強くお勧めします。
<!-- end --->
<!-- start --->
The above example program calls this before returning from `main`.
> 上記のプログラム例は `main` から戻る前にこれを呼び出します。
<!-- end --->

!!! note
  <!-- start --->
  Currently, dynamically linking with the `libjulia` shared library requires passing the `RTLD_GLOBAL` option.
  > 現在、 `libjulia` 共有ライブラリと動的にリンクするには、`RTLD_GLOBAL` オプションを指定する必要があります。
  <!-- end --->
  <!-- start --->
  In Python, this looks like:
  > Pythonでは、次のようになります。
  <!-- end --->

  ```python
  >>> julia=CDLL('./libjulia.dylib',RTLD_GLOBAL)
  >>> julia.jl_init.argtypes = []
  >>> julia.jl_init()
  250593296
  ```

!!! note
  <!-- start --->
  If the julia program needs to access symbols from the main executable, it may be necessary to add `-Wl,--export-dynamic` linker flag at compile time on Linux in addition to the ones generated by `julia-config.jl` described below.
  > juliaプログラムが主な実行可能ファイルからシンボルにアクセスする必要がある場合、 `julia-config.jl` によって生成されたものに加えて、` -Wl、 - export-dynamic` リンカフラグをLinux上でコンパイル時に追加する必要があります 以下で説明します。
  <!-- end --->
  <!-- start --->
  This is not necessary when compiling a shared library.
  > これは共有ライブラリをコンパイルするときは必要ありません。
  <!-- end --->
<!-- start --->

### Using julia-config to automatically determine build parameters

> ### julia-configを使ってビルドパラメータを自動的に決定する

<!-- start --->
The script `julia-config.jl` was created to aid in determining what build parameters are required by a program that uses embedded Julia. 
> `julia-config.jl`スクリプトは、埋め込まれたJuliaを使用するプログラムがどのビルドパラメータを必要としているかを判断するために作成されました。
<!-- end --->
<!-- start --->
This script uses the build parameters and system configuration of the particular Julia distribution it is invoked by to export the necessary compiler flags for an embedding program to interact with that distribution.  
> このスクリプトは、埋め込みプログラムがそのディストリビューションと対話するために必要なコンパイラフラグをエクスポートするために呼び出される特定のJuliaディストリビューションのビルドパラメータとシステム構成を使用します。
<!-- end --->
<!-- start --->
This script is located in the Julia shared data directory.
> このスクリプトはJulia共有データディレクトリにあります。
<!-- end --->

<!-- start --->

### Example

> ### 例

<!-- end --->

```c
#include <julia.h>

int main(int argc, char *argv[])
{
    jl_init();
    (void)jl_eval_string("println(sqrt(2.0))");
    jl_atexit_hook(0);
    return 0;
}
```

<!-- start --->

### On the command line

> ### コマンドラインで

<!-- end --->

<!-- start --->
A simple use of this script is from the command line.
> このスクリプトの簡単な使い方は、コマンドラインからのものです。
<!-- end --->
<!-- start --->
Assuming that `julia-config.jl` is located in `/usr/local/julia/share/julia`, it can be invoked on the command line directly and takes any combination of 3 flags:
>`julia-config.jl` が `/usr/local/julia/share/julia` にあると仮定すると、それはコマンドラインで直接呼び出すことができ、3つのフラグの任意の組み合わせをとります：
<!-- end --->

```sh
/usr/local/julia/share/julia/julia-config.jl
Usage: julia-config [--cflags|--ldflags|--ldlibs]
```

<!-- start --->
If the above example source is saved in the file `embed_example.c`, then the following command will compile it into a running program on Linux and Windows (MSYS2 environment), or if on OS/X, then substitute `clang` for `gcc`.:
> 上記のサンプルソースが `embed_example.c` ファイルに保存されている場合、次のコマンドはそれをLinuxおよびWindows(MSYS2環境)上の実行中のプログラムにコンパイルします。OS / Xの場合は、 `clang` を `gcc`：
<!-- end --->

```sh
/usr/local/julia/share/julia/julia-config.jl --cflags --ldflags --ldlibs | xargs gcc embed_example.c
```

<!-- start --->

### Use in Makefiles

> ### Makefileでの使用

<!-- end --->

<!-- start --->
But in general, embedding projects will be more complicated than the above, and so the following allows general makefile support as well – assuming GNU make because of the use of the **shell** macro expansions. 
> しかし、一般に、プロジェクトの埋め込みは上記よりも複雑になるので、**shell** マクロ展開の使用のためGNU make を想定して一般的な makefile のサポートも可能になります。
<!-- end --->
<!-- start --->
Additionally, though many times `julia-config.jl` may be found in the directory `/usr/local`, this is not necessarily the case, but Julia can be used to locate `julia-config.jl` too, and the makefile can be used to take advantage of that.
> さらに、 `/usr/local` ディレクトリに `julia-config.jl` が見つかることもありますが、必ずしもそうではありませんが、Juliaを使って `julia-config.jl` を探すことができます。 それを利用するためにmakefileを使うことができます。
<!-- end --->
<!-- start --->
The above example is extended to use a Makefile:
> 上記の例はMakefileを使うように拡張されています：
<!-- end --->

```Makefile
JL_SHARE = $(shell julia -e 'print(joinpath(JULIA_HOME,Base.DATAROOTDIR,"julia"))')
CFLAGS   += $(shell $(JL_SHARE)/julia-config.jl --cflags)
CXXFLAGS += $(shell $(JL_SHARE)/julia-config.jl --cflags)
LDFLAGS  += $(shell $(JL_SHARE)/julia-config.jl --ldflags)
LDLIBS   += $(shell $(JL_SHARE)/julia-config.jl --ldlibs)

all: embed_example
```

<!-- start --->
Now the build command is simply `make`.
> ビルドコマンドは単に `make` です。
<!-- end --->
<!-- start --->

## Converting Types

> ## 型の変換

<!-- end --->

<!-- start --->
Real applications will not just need to execute expressions, but also return their values to the host program. 
> 実際のアプリケーションは式を実行するだけでなく、その値をホストプログラムに返すだけです。
<!-- end --->
<!-- start --->
`jl_eval_string` returns a `jl_value_t*`, which is a pointer to a heap-allocated Julia object. 
> `jl_eval_string` は `jl_value_t*` を返します。これは、ヒープに割り当てられたJuliaオブジェクトへのポインタです。
<!-- end --->
<!-- start --->
Storing simple data types like [`Float64`](@ref) in this way is called `boxing`, and extracting the stored primitive data is called `unboxing`.
> このように [`Float64`](@ref) のような単純なデータ型を格納することを `ボクシング` と呼び、格納されたプリミティブデータを抽出することを `アンボックス化` といいます。
<!-- end --->
<!-- start --->
Our improved sample program that calculates the square root of 2 in Julia and reads back the result in C looks as follows:
> Juliaで2の平方根を計算し、Cで結果を読み戻した改善されたサンプルプログラムは次のようになります：
<!-- end --->

```c
jl_value_t *ret = jl_eval_string("sqrt(2.0)");

if (jl_typeis(ret, jl_float64_type)) {
    double ret_unboxed = jl_unbox_float64(ret);
    printf("sqrt(2.0) in C: %e \n", ret_unboxed);
}
else {
    printf("ERROR: unexpected return type from sqrt(::Float64)\n");
}
```

<!-- start --->
In order to check whether `ret` is of a specific Julia type, we can use the `jl_isa`, `jl_typeis`, or `jl_is_...` functions.
> `ret` が特定のJulia型であるかどうかを調べるために、 `jl_isa`, `jl_typeis`, または `jl_is _...` 関数を使うことができます。
<!-- end --->
<!-- start --->
By typing `typeof(sqrt(2.0))` into the Julia shell we can see that the return type is [`Float64`](@ref) (`double` in C).
> Juliaシェルに `typeof(sqrt(2.0))` と入力すると、戻り値の型は [`Float64`](@ref) (C の `double`) になります。
<!-- end --->
<!-- start --->
To convert the boxed Julia value into a C double the `jl_unbox_float64` function is used in the above code snippet.
> boxed Juliaの値をC double に変換するには、上記のコードスニペットで `jl_unbox_float64` 関数を使用します。
<!-- end --->

<!-- start --->
Corresponding `jl_box_...` functions are used to convert the other way:
> 対応する `jl_box _...` 関数は、逆の変換に使用されます：
<!-- end --->

```c
jl_value_t *a = jl_box_float64(3.0);
jl_value_t *b = jl_box_float32(3.0f);
jl_value_t *c = jl_box_int32(3);
```

<!-- start --->
As we will see next, boxing is required to call Julia functions with specific arguments.
> 次のように、ジュリア関数を特定の引数で呼び出すには、ボクシングが必要です。
<!-- end --->

<!-- start --->

## Calling Julia Functions

> ## ジュリア関数の呼び出し

<!-- end --->
<!-- start --->
While `jl_eval_string` allows C to obtain the result of a Julia expression, it does not allow passing arguments computed in C to Julia.
> `jl_eval_string`ではCがJulia式の結果を得ることができますが、Cで計算された引数をJuliaに渡すことはできません。
<!-- end --->
<!-- start --->
For this you will need to invoke Julia functions directly, using `jl_call`:
> このためには、 `jl_call`を使ってジュリア関数を直接呼び出す必要があります：
<!-- end --->

```c
jl_function_t *func = jl_get_function(jl_base_module, "sqrt");
jl_value_t *argument = jl_box_float64(2.0);
jl_value_t *ret = jl_call1(func, argument);
```

<!-- start --->
In the first step, a handle to the Julia function `sqrt` is retrieved by calling `jl_get_function`.
> 最初のステップでは、Julia関数 `sqrt`へのハンドルは` jl_get_function`を呼び出すことによって取得されます。
<!-- end --->
<!-- start --->
The first argument passed to `jl_get_function` is a pointer to the `Base` module in which `sqrt` is defined.
> `jl_get_function` に渡される最初の引数は、 `sqrt` が定義された `Base` モジュールへのポインタです。
<!-- end --->
Then, the double value is boxed using `jl_box_float64`.
> 次に、double値は `jl_box_float64`を使ってボックス化されます。
<!-- end --->
<!-- start --->
Finally, in the last step, the function is called using `jl_call1`. 
> 最後に、最後のステップで、関数は `jl_call1` を使って呼び出されます。
<!-- end --->
<!-- start --->
`jl_call0`, `jl_call2`, and `jl_call3` functions also exist, to conveniently handle different numbers of arguments. 
> `jl_call0`, `jl_call2`, `jl_call3` 関数も存在し、異なる数の引数を扱うことができます。
<!-- end --->
<!-- start --->
To pass more arguments, use `jl_call`:
> より多くの引数を渡すには、 `jl_call` を使います：
<!-- end --->
<!-- start --->

```Julia
jl_value_t *jl_call(jl_function_t *f, jl_value_t **args, int32_t nargs)
```

<!-- start --->
Its second argument `args` is an array of `jl_value_t*` arguments and `nargs` is the number of arguments.
> 第2引数 `args` は `jl_value_t*` 引数の配列で、 `nargs` は引数の数です。
<!-- end --->

<!-- start --->

## Memory Management

> ## メモリ管理

<!-- end --->
<!-- start --->
As we have seen, Julia objects are represented in C as pointers. 
> これまで見てきたように、JuliaオブジェクトはC言語でポインタとして表現されています。
<!-- end --->
<!-- start --->
This raises the question of who is responsible for freeing these objects.
> これにより、これらのオブジェクトを解放する責任を負っているのかという疑問が生じます。
<!-- end --->

<!-- start --->
Typically, Julia objects are freed by a garbage collector (GC), but the GC does not automatically know that we are holding a reference to a Julia value from C. 
> 通常、Juliaオブジェクトはガベージコレクタ (GC) によって解放されますが、 GC は自動的にCからジュリア値への参照を保持していることを認識しません
<!-- end --->
<!-- start --->
This means the GC can free objects out from under you, rendering pointers invalid.
> つまり、GCがあなたの下からオブジェクトを解放し、ポインタを無効にすることができます。
<!-- end --->

<!-- start --->
The GC can only run when Julia objects are allocated.
> GCはJuliaオブジェクトが割り当てられているときにのみ実行できます。
<!-- end --->
<!-- start --->
Calls like `jl_box_float64` perform allocation, and allocation might also happen at any point in running Julia code.
> `jl_box_float64` のような呼び出しは割り当てを行い、割り当ては実行中のJuliaコードのどの時点でも起こるかもしれません。
<!-- end --->
<!-- start --->
However, it is generally safe to use pointers in between `jl_...` calls.
> しかし、 `jl_...` の呼び出しの間にポインタを使うのは一般的に安全です。
<!-- end --->
<!-- start --->
But in order to make sure that values can survive `jl_...` calls, we have to tell Julia that we hold a reference to a Julia value.
> しかし、値が `jl_...` 呼び出しで生き残ることを保証するために、JuliaにJulia値への参照を保持していることを伝えなければなりません。
<!-- end --->
<!-- start --->
This can be done using the `JL_GC_PUSH` macros:
> これは `JL_GC_PUSH` マクロを使って行うことができます：
<!-- end --->

```c
jl_value_t *ret = jl_eval_string("sqrt(2.0)");
JL_GC_PUSH1(&ret);
// Do something with ret
JL_GC_POP();
```

<!-- start --->
The `JL_GC_POP` call releases the references established by the previous `JL_GC_PUSH`.
> `JL_GC_POP` 呼び出しは、前の `JL_GC_PUSH` によって確立された参照を解放します。
<!-- end --->
<!-- start --->
Note that `JL_GC_PUSH`  is working on the stack, so it must be exactly paired with a `JL_GC_POP` before
> `JL_GC_PUSH` はスタック上で動作しているので、前に `JL_GC_POP` と正確に対にする必要があります
<!-- end --->
<!-- start --->
the stack frame is destroyed.
> スタックフレームは破壊されます。
<!-- end --->

<!-- start --->
Several Julia values can be pushed at once using the `JL_GC_PUSH2` , `JL_GC_PUSH3` , and `JL_GC_PUSH4` macros.
> `JL_GC_PUSH2`, `JL_GC_PUSH3`, `JL_GC_PUSH4` マクロを使って、いくつかのJulia値を一度にプッシュすることができます。
<!-- end --->
<!-- start --->
To push an array of Julia values one can use the  `JL_GC_PUSHARGS` macro, which can be used as follows:
> Julia値の配列をプッシュするには、次のように `JL_GC_PUSHARGS` マクロを使用できます：
<!-- end --->

```c
jl_value_t **args;
JL_GC_PUSHARGS(args, 2); // args can now hold 2 `jl_value_t*` objects
args[0] = some_value;
args[1] = some_other_value;
// Do something with args (e.g. call jl_... functions)
JL_GC_POP();
```

<!-- start --->
The garbage collector also operates under the assumption that it is aware of every old-generation object pointing to a young-generation one.
> ガベージコレクタは、若い世代のオブジェクトを指しているすべての古い世代のオブジェクトを認識していることを前提として動作します。
<!-- end --->
<!-- start --->
Any time a pointer is updated breaking that assumption, it must be signaled to the collector with the `jl_gc_wb` (write barrier) function like so:
> その前提を破ってポインタが更新されると、次のように `jl_gc_wb` (書き込みバリア)関数を使ってコレクタに通知する必要があります：
<!-- end --->

```c
jl_value_t *parent = some_old_value, *child = some_young_value;
((some_specific_type*)parent)->field = child;
jl_gc_wb(parent, child);
```

<!-- start --->
It is in general impossible to predict which values will be old at runtime, so the write barrier must be inserted after all explicit stores.
> 実行時にどの値が古いかを予測することは一般的に不可能であるため、書き込みバリアはすべての明示的なストアの後に挿入する必要があります。
<!-- end --->
<!-- start --->
One notable exception is if the `parent` object was just allocated and garbage collection was not run since then.
> 注目すべき例外の1つは、 `parent` オブジェクトが割り当てられたばかりで、ガベージコレクションがそれ以来実行されていない場合です。
<!-- end --->
<!-- start --->
Remember that most `jl_...` functions can sometimes invoke garbage collection.
> たいていの `jl_...` 関数はガベージコレクションを呼び出すことがあります。
<!-- end --->

<!-- start --->
The write barrier is also necessary for arrays of pointers when updating their data directly.
> 書き込みバリアは、データを直接更新するときにポインタの配列にも必要です。
<!-- end --->
<!-- start --->
For example:
> 例えば：
<!-- end --->

```c
jl_array_t *some_array = ...; // e.g. a Vector{Any}
void **data = (void**)jl_array_data(some_array);
jl_value_t *some_value = ...;
data[0] = some_value;
jl_gc_wb(some_array, some_value);
```

<!-- start --->

### Manipulating the Garbage Collector

> ### ガベージコレクタを操作する

<!-- end --->

<!-- start --->
There are some functions to control the GC.
> GCを制御するいくつかの機能があります。
<!-- end --->
<!-- start --->
In normal use cases, these should not be necessary.
> 通常の使用の場合、これらは必要ではありません。
<!-- end --->

| Function             | Description                                  |
|:-------------------- |:-------------------------------------------- |
| `jl_gc_collect()`    | Force a GC run                               |
| `jl_gc_enable(0)`    | Disable the GC, return previous state as int |
| `jl_gc_enable(1)`    | Enable the GC,  return previous state as int |
| `jl_gc_is_enabled()` | Return current state as int                  |

<!-- start --->

## Working with Arrays

> ## 配列の操作

<!-- end --->

<!-- start --->
Julia and C can share array data without copying.
> JuliaとCはコピーせずに配列データを共有できます。
<!-- end --->
<!-- start --->
The next example will show how this works.
> 次の例は、これがどのように機能するかを示します。
<!-- end --->

<!-- start --->
Julia arrays are represented in C by the datatype `jl_array_t*`. Basically, `jl_array_t` is a struct that contains:
> ジュリア配列はCでデータ型 `jl_array_t *`で表されます。 基本的に、 `jl_array_t`は以下を含む構造体です：
<!-- end --->

  * Information about the datatype
  * A pointer to the data block
  * Information about the sizes of the array

<!-- start --->
To keep things simple, we start with a 1D array.
> 物事を単純にするために、1D配列から始めます。
<!-- end --->
<!-- start --->
Creating an array containing Float64 elements of length 10 is done by:
> 長さ10のFloat64要素を含む配列を作成するには、次のようにします。
<!-- end --->

```c
jl_value_t* array_type = jl_apply_array_type(jl_float64_type, 1);
jl_array_t* x          = jl_alloc_array_1d(array_type, 10);
```

<!-- start --->
Alternatively, if you have already allocated the array you can generate a thin wrapper around its data:
> また、配列をすでに割り当てている場合は、データの周りに薄いラッパーを生成することもできます。
<!-- end --->

```c
double *existingArray = (double*)malloc(sizeof(double)*10);
jl_array_t *x = jl_ptr_to_array_1d(array_type, existingArray, 10, 0);
```

<!-- start --->
The last argument is a boolean indicating whether Julia should take ownership of the data.
> 最後の引数は、Juliaがデータの所有権を取得するかどうかを示すブール値です。
<!-- end --->
<!-- start --->
If this argument is non-zero, the GC will call `free` on the data pointer when the array is no longer referenced.
> この引数がゼロでない場合、配列がもはや参照されていないとき、GCはデータポインタに対して `free`を呼び出します。
<!-- end --->

<!-- start --->
In order to access the data of x, we can use `jl_array_data`:
> xのデータにアクセスするために、 `jl_array_data`を使うことができます：
<!-- end --->

```c
double *xData = (double*)jl_array_data(x);
```

<!-- start --->
Now we can fill the array:
> 今度は配列を埋めることができます：
<!-- end --->

```c
for(size_t i=0; i<jl_array_len(x); i++)
    xData[i] = i;
```

<!-- start --->
Now let us call a Julia function that performs an in-place operation on `x`:
> 次に、 `x`に対してインプレース操作を実行するJulia関数を呼び出しましょう。
<!-- end --->

```c
jl_function_t *func = jl_get_function(jl_base_module, "reverse!");
jl_call1(func, (jl_value_t*)x);
```

<!-- start --->
By printing the array, one can verify that the elements of `x` are now reversed.
> アレイを表示し、`x` の要素が反転されたことを確認することができる。
<!-- end --->

<!-- end --->
<!-- start --->

### Accessing Returned Arrays

> ### 返された配列へのアクセス

<!-- end --->
<!-- start --->
If a Julia function returns an array, the return value of `jl_eval_string` and `jl_call` can be cast to a `jl_array_t*`:
> Julia関数が配列を返す場合、 `jl_eval_string`と` jl_call`の戻り値は `jl_array_t *`にキャストできます：
<!-- end --->

```c
jl_function_t *func  = jl_get_function(jl_base_module, "reverse");
jl_array_t *y = (jl_array_t*)jl_call1(func, (jl_value_t*)x);
```

<!-- start --->
Now the content of `y` can be accessed as before using `jl_array_data`.
> これで `yl 'の内容は` jl_array_data`を使って前と同じようにアクセスできます。
<!-- end --->
<!-- start --->
As always, be sure to keep a reference to the array while it is in use.
> いつものように、アレイが使用されている間は、アレイへの参照を保持してください。
<!-- end --->

### Multidimensional Arrays

> ### 多次元配列

<!-- end --->
<!-- start --->
Julia's multidimensional arrays are stored in memory in column-major order.
> Juliaの多次元配列は列優先順序でメモリに格納されます。
<!-- end --->
<!-- start --->
Here is some code that creates a 2D array and accesses its properties:
> 2D配列を作成し、そのプロパティにアクセスするコードを次に示します。
<!-- end --->

```c
// Create 2D array of float64 type
jl_value_t *array_type = jl_apply_array_type(jl_float64_type, 2);
jl_array_t *x  = jl_alloc_array_2d(array_type, 10, 5);

// Get array pointer
double *p = (double*)jl_array_data(x);
// Get number of dimensions
int ndims = jl_array_ndims(x);
// Get the size of the i-th dim
size_t size0 = jl_array_dim(x,0);
size_t size1 = jl_array_dim(x,1);

// Fill array with data
for(size_t i=0; i<size1; i++)
    for(size_t j=0; j<size0; j++)
        p[j + size0*i] = i + j;
```

<!-- start --->
Notice that while Julia arrays use 1-based indexing, the C API uses 0-based indexing (for example in calling `jl_array_dim`) in order to read as idiomatic C code.
> Julia配列は1ベースの索引付けを使用しますが、C APIは慣用のCコードとして読み取るために、0ベースの索引付け(たとえば、jl_array_dimの呼び出しなど)を使用しています。
<!-- end --->
<!-- start --->

## Exceptions

> ## 例外

<!-- end --->
<!-- start --->
Julia code can throw exceptions.
> ジュリアコードは例外をスローすることができます。
<!-- end --->
<!-- start --->
For example, consider:
> たとえば、次の点を考慮してください。
<!-- end --->

```c
jl_eval_string("this_function_does_not_exist()");
```

<!-- start --->
This call will appear to do nothing.
> この呼び出しは何もしないように見えます。
<!-- end --->
<!-- start --->
However, it is possible to check whether an exception was thrown:
> ただし、例外がスローされたかどうかを確認することは可能です。
<!-- end --->

```c
if (jl_exception_occurred())
    printf("%s \n", jl_typeof_str(jl_exception_occurred()));
```

<!-- start --->
If you are using the Julia C API from a language that supports exceptions (e.g. Python, C#, C++), it makes sense to wrap each call into `libjulia` with a function that checks whether an exception was thrown, and then rethrows the exception in the host language.
> 例外をサポートする言語(Python、C＃、C ++など)からJulia C APIを使用している場合は、例外がスローされたかどうかをチェックする関数を使用して各呼び出しを `libjulia`にラップし、例外を再発行します ホスト言語で書かれています。
<!-- end --->
<!-- start --->

## Throwing Julia Exceptions

> ## ジュリアの例外スロー

<!-- end --->
<!-- start --->
When writing Julia callable functions, it might be necessary to validate arguments and throw exceptions to indicate errors.
> Julia呼び出し可能関数を記述するときは、引数を検証し、エラーを示す例外をスローする必要があります。
<!-- end --->
<!-- start --->
A typical type check looks like:
> 典型的な型チェックは以下のようになります：
<!-- end --->

```c
if (!jl_typeis(val, jl_float64_type)) {
    jl_type_error(function_name, (jl_value_t*)jl_float64_type, val);
}
```

<!-- start --->
General exceptions can be raised using the functions:
> 関数を使用して一般的な例外を発生させることができます：
<!-- end --->

```c
void jl_error(const char *str);
void jl_errorf(const char *fmt, ...);
```

<!-- start --->
`jl_error` takes a C string, and `jl_errorf` is called like `printf`:
>`jl_error`はC文字列をとり、` jl_errorf` は `printf`のように呼び出されます：
<!-- end --->

```c
jl_errorf("argument x = %d is too large", x);
```

<!-- start --->
where in this example `x` is assumed to be an integer.
> この例では、 `x` は整数であると仮定されます。
<!-- end --->