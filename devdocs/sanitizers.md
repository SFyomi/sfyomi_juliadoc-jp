# Sanitizer support

## General considerations

<!-- EN -->
Using Clang's sanitizers obviously require you to use Clang (`USECLANG=1`), but there's another catch: most sanitizers require a run-time library, provided by the host compiler, while the instrumented code generated by Julia's JIT relies on functionality from that library.
> JuliaのJITによって生成されたインストルメントコードは、その機能に依存しているのに対し、Clangのサニタイザを使用すると、明らかにClang（USECLANG = 1）を使用する必要がありますが、別のキャッチがあります。 としょうかん。
<!-- EN -->
This implies that the LLVM version of your host compiler matches that of the LLVM library used within Julia.
> これは、ホストコンパイラのLLVMバージョンがJulia内で使用されているLLVMライブラリのLLVMバージョンと一致することを意味します。

<!-- EN -->
An easy solution is to have an dedicated build folder for providing a matching toolchain, by building with `BUILD_LLVM_CLANG=1`.
> 簡単な解決策は、 `BUILD_LLVM_CLANG = 1`でビルドすることによって、対応するツールチェーンを提供する専用のビルドフォルダを用意することです。
<!-- EN -->
You can then refer to this toolchain from another build folder by specifying `USECLANG=1` while overriding the `CC` and `CXX` variables.
> 次に、 `USECLANG = 1`を指定して` CC`と `CXX`変数をオーバーライドして、このツールチェーンを別のビルドフォルダから参照することができます。

## Address Sanitizer (ASAN)

<!-- EN -->
For detecting or debugging memory bugs, you can use Clang's [address sanitizer (ASAN)](http://clang.llvm.org/docs/AddressSanitizer.html).
> メモリのバグを検出またはデバッグするには、Clangの[アドレスサニタイザ（ASAN）]（http://clang.llvm.org/docs/AddressSanitizer.html）を使用できます。
<!-- EN -->
By compiling with `SANITIZE=1` you enable ASAN for the Julia compiler and its generated code.
> `SANITIZE = 1`でコンパイルすると、Juliaコンパイラとその生成コードに対してASANを有効にできます。
<!-- EN -->
In addition, you can specify `LLVM_SANITIZE=1` to sanitize the LLVM library as well.
> さらに、 `LLVM_SANITIZE = 1`を指定すると、LLVMライブラリをサニタイズすることもできます。
<!-- EN -->
Note that these options incur a high performance and memory cost.
> これらのオプションは、高いパフォーマンスとメモリコストを必要とすることに注意してください。
<!-- EN -->
For example, using ASAN for Julia and LLVM makes `testall1` takes 8-10 times as long while using 20 times as much memory (this can be reduced to respectively a factor of 3 and 4 by using the options described below).
> たとえば、JuliaとLLVMにASANを使用すると、 `testall1`は20倍のメモリを使用しながら8-10倍の時間がかかります（これは、後述のオプションを使用することでそれぞれ3と4倍に減らすことができます）。

<!-- EN -->
By default, Julia sets the `allow_user_segv_handler=1` ASAN flag, which is required for signal delivery to work properly.
> デフォルトでは、Juliaはシグナルの配信が正常に動作するために必要な `allow_user_segv_handler = 1` ASANフラグを設定します。
<!-- EN -->
You can define other options using the `ASAN_OPTIONS` environment flag, in which case you'll need to repeat the default option mentioned before.
> `ASAN_OPTIONS`環境フラグを使用して他のオプションを定義することができます。この場合、前述のデフォルトオプションを繰り返す必要があります。
<!-- EN -->
For example, memory usage can be reduced by specifying `fast_unwind_on_malloc=0` and `malloc_context_size=2`, at the cost of backtrace accuracy.
> 例えば、バックトレース精度を犠牲にして、 `fast_unwind_on_malloc = 0`と` malloc_context_size = 2`を指定することでメモリ使用量を減らすことができます。
<!-- EN -->
For now, Julia also sets `detect_leaks=0`, but this should be removed in the future.
> 今のところ、Juliaは `detect_leaks = 0`も設定しますが、これは将来削除されるべきです。

## Memory Sanitizer (MSAN)

<!-- EN -->
For detecting use of uninitialized memory, you can use Clang's [memory sanitizer (MSAN)](http://clang.llvm.org/docs/MemorySanitizer.html) by compiling with `SANITIZE_MEMORY=1`.
> 初期化されていないメモリの使用を検出するには、 `SANITIZE_MEMORY = 1` でコンパイルしてClangの [メモリサニタイザ（MSAN）](http://clang.llvm.org/docs/MemorySanitizer.html)を使用できます。
