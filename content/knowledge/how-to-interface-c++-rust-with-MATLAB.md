+++
title = "C++やRustをMATLABと連携させる方法．"
description = "MATLABのターミナル実行とその応用についてまとめます．"
date = 2026-04-23
updated = 2026-04-23
authors = ["Shingo Yamabe"]
tags = ["MATLAB"]
category = "knowledge"
draft = false
weight = 1
+++

# MATLAB 導入ガイド
### インストールから ターミナル実行・C++/Rust 連携まで

---

## 目次

1. [MATLABとは](#1-matlabとは)
2. [インストール](#2-インストール)
3. [ライセンス認証](#3-ライセンス認証)
4. [ディレクトリ構成と基本設定](#4-ディレクトリ構成と基本設定)
5. [ターミナルからの実行](#5-ターミナルからの実行)
6. [C++ との連携（MEX / Engine API）](#6-c-との連携mex--engine-api)
7. [Rust との連携（FFI 経由）](#7-rust-との連携ffi-経由)
8. [トラブルシューティング](#8-トラブルシューティング)
9. [参考リンク](#9-参考リンク)

---

## 1. MATLABとは

MATLAB（Matrix Laboratory）は MathWorks 社が提供する数値計算・データ解析・可視化のための統合開発環境です。

| 特徴 | 説明 |
|------|------|
| 行列演算 | ネイティブで行列・ベクトル演算をサポート |
| 豊富なツールボックス | 信号処理・機械学習・制御工学など |
| 可視化 | グラフ・3Dプロット等を即座に描画 |
| 外部言語連携 | C/C++, Python, Rust (FFI) と連携可能 |
| スクリプト実行 | `.m` ファイルをCLIからバッチ実行可能 |

---

## 2. インストール

### 2-1. システム要件（2024 R2 以降）

| 項目 | 要件 |
|------|------|
| OS | Windows 10/11, macOS 13+, Ubuntu 22.04+ |
| RAM | 最低 4 GB（推奨 8 GB 以上） |
| ストレージ | 最低 3 GB（フル環境 30 GB+） |
| CPU | 64-bit 対応（Intel/AMD/Apple Silicon） |

### 2-2. インストール手順

**① MathWorks アカウント作成**

```
https://www.mathworks.com/mwaccount/register
```

**② インストーラのダウンロード**

[https://www.mathworks.com/downloads/](https://www.mathworks.com/downloads/) から最新版を取得します。

**③ インストール実行**

```bash
# Linux の場合（ダウンロードしたインストーラを実行）
chmod +x matlab_R2024b_Linux.zip
unzip matlab_R2024b_Linux.zip -d matlab_installer
cd matlab_installer
sudo ./install
```

```powershell
# Windows の場合（管理者権限のPowerShell）
Start-Process .\matlab_R2024b_win64.exe -Verb RunAs
```

```bash
# macOS の場合
open matlab_R2024b_maci64.dmg
# ウィザードに従ってインストール
```

---

## 3. ライセンス認証

### 3-1. オンライン認証（推奨）

インストール完了後、MathWorks アカウントでログインすることで自動的にライセンスが紐付けられます。

### 3-2. ライセンスファイルによる認証（オフライン環境）

```bash
# ライセンスファイルの配置場所
# Linux / macOS
/usr/local/MATLAB/R2024b/licenses/license.lic

# Windows
C:\Program Files\MATLAB\R2024b\licenses\license.lic
```

### 3-3. ライセンス確認コマンド

```matlab
% MATLAB コンソール内で実行
license
% 例: 'MATLAB' が返れば有効
```

---

## 4. ディレクトリ構成と基本設定

### 4-1. 標準ディレクトリ

```
~/Documents/MATLAB/          ← デフォルトの作業ディレクトリ
├── startup.m                ← 起動時に自動実行されるスクリプト
├── scripts/                 ← 独自スクリプト置き場
└── mex/                     ← MEX ファイル（C++連携）置き場
```

### 4-2. PATH の設定（Linux / macOS）

```bash
# ~/.bashrc または ~/.zshrc に追記
export PATH="/usr/local/MATLAB/R2024b/bin:$PATH"

# 反映
source ~/.zshrc
```

```bash
# 動作確認
which matlab
# → /usr/local/MATLAB/R2024b/bin/matlab
```

---

## 5. ターミナルからの実行

### 5-1. 対話モードで起動

```bash
# GUI あり（デフォルト）
matlab

# GUI なし（ヘッドレス環境・SSH 接続時）
matlab -nodesktop -nosplash
```

### 5-2. スクリプトをバッチ実行

```bash
# .m ファイルを直接実行して終了
matlab -batch "run('~/Documents/MATLAB/scripts/my_script.m')"

# 関数を直接呼び出す
matlab -batch "myFunction(3.14, 42)"

# ログ出力しつつ実行
matlab -batch "run('analysis.m')" 2>&1 | tee output.log
```

### 5-3. インラインコマンドの実行

```bash
# 1行の計算
matlab -batch "disp(sum(1:100))"
# → 5050

# 複数コマンドをセミコロンで区切る
matlab -batch "A = magic(4); disp(det(A))"
```

### 5-4. 終了コードのハンドリング

```bash
matlab -batch "run('test.m')"
echo "Exit code: $?"
# 正常終了: 0, エラー: 1
```

### 5-5. サンプル：スクリプトファイル（`analysis.m`）

```matlab
%% analysis.m - ターミナルから実行するサンプルスクリプト
clc; clear; close all;

fprintf('=== MATLAB バッチ実行サンプル ===\n');

% データ生成
x = linspace(0, 2*pi, 1000);
y = sin(x) .* exp(-0.3 * x);

% 統計量の表示
fprintf('最大値: %.4f\n', max(y));
fprintf('最小値: %.4f\n', min(y));
fprintf('平均値: %.4f\n', mean(y));

% 結果をCSVに保存
data = [x', y'];
writematrix(data, 'result.csv');
fprintf('結果を result.csv に保存しました。\n');
```

```bash
# 実行
matlab -batch "run('analysis.m')"
```

---

## 6. C++ との連携（MEX / Engine API）

### 6-1. MEX とは

MEX（MATLAB Executable）は C/C++ で書いたコードを MATLAB から `.mex` として呼び出す仕組みです。計算コアを高速な C++ で実装し、MATLAB から呼び出すユースケースに向いています。

### 6-2. MEX のビルド環境確認

```matlab
% MATLAB コンソール内で確認
mex -setup C++
% → MEX configured to use 'g++' などが表示されればOK
```

### 6-3. C++ MEX サンプル：ベクトル二乗和

**`vec_square_sum.cpp`**

```cpp
// vec_square_sum.cpp
// 入力ベクトルの各要素を二乗した和を返す MEX 関数
// 呼び出し例: result = vec_square_sum([1, 2, 3, 4, 5])

#include "mex.hpp"
#include "mexAdapter.hpp"
#include <numeric>
#include <vector>

class MexFunction : public matlab::mex::Function {
public:
    void operator()(matlab::mex::ArgumentList outputs,
                    matlab::mex::ArgumentList inputs) override
    {
        // 引数チェック
        if (inputs.size() != 1) {
            throw std::runtime_error("入力引数は1つ（ベクトル）が必要です");
        }

        // MATLAB配列 → C++ vector に変換
        matlab::data::TypedArray<double> input = inputs[0];
        std::vector<double> vec(input.begin(), input.end());

        // 二乗和の計算
        double result = std::accumulate(
            vec.begin(), vec.end(), 0.0,
            [](double acc, double val) { return acc + val * val; }
        );

        // 出力
        matlab::data::ArrayFactory factory;
        outputs[0] = factory.createScalar<double>(result);
    }
};
```

**ビルドと実行**

```bash
# ターミナルから直接ビルド
matlab -batch "mex vec_square_sum.cpp"

# または MATLAB コンソール内で
# >> mex vec_square_sum.cpp
```

```matlab
% MATLAB から呼び出す
result = vec_square_sum([1, 2, 3, 4, 5]);
fprintf('二乗和: %.0f\n', result);
% → 55
```

### 6-4. C++ Engine API サンプル：C++側からMATLABを操作

C++ プログラムから MATLAB エンジンを起動して計算を実行する例です。

**`matlab_engine_demo.cpp`**

```cpp
// matlab_engine_demo.cpp
// C++ から MATLAB Engine API を使って行列演算を行うサンプル
// ビルド: g++ -std=c++17 matlab_engine_demo.cpp \
//           -I$(MATLAB_ROOT)/extern/include \
//           -L$(MATLAB_ROOT)/extern/bin/glnxa64 \
//           -lMatlabEngine -lMatlabDataArray \
//           -Wl,-rpath,$(MATLAB_ROOT)/extern/bin/glnxa64 \
//           -o matlab_engine_demo

#include "MatlabDataArray.hpp"
#include "MatlabEngine.hpp"
#include <iostream>
#include <memory>
#include <vector>

int main() {
    std::cout << "MATLAB Engine を起動中...\n";

    // MATLAB エンジンの起動
    std::unique_ptr<matlab::engine::MATLABEngine> matlabPtr =
        matlab::engine::startMATLAB();

    matlab::data::ArrayFactory factory;

    // ---- 例1: スカラー演算 ----
    matlabPtr->eval(u"x = sqrt(2);");
    matlab::data::TypedArray<double> x =
        matlabPtr->getVariable(u"x");
    std::cout << "sqrt(2) = " << x[0] << "\n";

    // ---- 例2: 行列の作成と乗算 ----
    // 2x2 行列 A を C++ 側で作成して MATLAB に渡す
    matlab::data::TypedArray<double> matA = factory.createArray<double>(
        {2, 2}, {1.0, 3.0, 2.0, 4.0}  // 列優先
    );
    matlabPtr->setVariable(u"A", matA);

    // MATLAB 側で A^2 を計算
    matlabPtr->eval(u"B = A * A;");
    matlab::data::TypedArray<double> matB =
        matlabPtr->getVariable(u"B");

    std::cout << "A^2 =\n";
    for (size_t i = 0; i < 2; ++i) {
        for (size_t j = 0; j < 2; ++j) {
            std::cout << "  " << matB[i][j];
        }
        std::cout << "\n";
    }

    // ---- 例3: FFT の実行 ----
    std::vector<double> signal = {1, 0, -1, 0, 1, 0, -1, 0};
    matlab::data::TypedArray<double> sig =
        factory.createArray({1, 8}, signal.begin(), signal.end());
    matlabPtr->setVariable(u"sig", sig);

    matlabPtr->eval(u"spectrum = abs(fft(sig));");
    matlab::data::TypedArray<double> spectrum =
        matlabPtr->getVariable(u"spectrum");

    std::cout << "FFT 絶対値: ";
    for (double v : spectrum) {
        std::cout << v << " ";
    }
    std::cout << "\n";

    std::cout << "完了。\n";
    return 0;
}
```

**ビルドスクリプト（`build_engine.sh`）**

```bash
#!/bin/bash
MATLAB_ROOT="/usr/local/MATLAB/R2024b"

g++ -std=c++17 matlab_engine_demo.cpp \
    -I"${MATLAB_ROOT}/extern/include" \
    -L"${MATLAB_ROOT}/extern/bin/glnxa64" \
    -lMatlabEngine -lMatlabDataArray \
    -Wl,-rpath,"${MATLAB_ROOT}/extern/bin/glnxa64" \
    -o matlab_engine_demo

echo "ビルド完了: ./matlab_engine_demo"
```

```bash
chmod +x build_engine.sh && ./build_engine.sh
./matlab_engine_demo
```

---

## 7. Rust との連携（FFI 経由）

Rust には公式の MATLAB バインディングがないため、**C FFI（Foreign Function Interface）** を使って MATLAB Engine C API を呼び出します。

### 7-1. プロジェクト構成

```
matlab_rust_demo/
├── Cargo.toml
├── build.rs           ← ビルドスクリプト（リンク設定）
└── src/
    ├── main.rs
    └── matlab_ffi.rs  ← FFI バインディング定義
```

### 7-2. `Cargo.toml`

```toml
[package]
name    = "matlab_rust_demo"
version = "0.1.0"
edition = "2021"

[dependencies]
libc = "0.2"

[build-dependencies]
# bindgen を使う場合は追加（オプション）
# bindgen = "0.69"
```

### 7-3. `build.rs`

```rust
// build.rs
// MATLAB Engine の共有ライブラリをリンクするビルドスクリプト

fn main() {
    let matlab_root = std::env::var("MATLAB_ROOT")
        .unwrap_or_else(|_| "/usr/local/MATLAB/R2024b".to_string());

    let lib_path = format!("{}/bin/glnxa64", matlab_root);

    println!("cargo:rustc-link-search=native={}", lib_path);
    println!("cargo:rustc-link-lib=dylib=eng");   // libeng（Engine API）
    println!("cargo:rustc-link-lib=dylib=mx");    // libmx（データ型）
    println!("cargo:rustc-env=LD_LIBRARY_PATH={}", lib_path);
}
```

### 7-4. `src/matlab_ffi.rs`（C API バインディング）

```rust
// src/matlab_ffi.rs
// MATLAB Engine C API の Rust FFI バインディング

use libc::{c_char, c_double, c_int, c_void};

// 不透明ポインタ型
#[repr(C)]
pub struct Engine {
    _private: [u8; 0],
}

#[repr(C)]
pub struct MxArray {
    _private: [u8; 0],
}

#[link(name = "eng")]
#[link(name = "mx")]
extern "C" {
    /// MATLAB エンジンを起動する
    pub fn engOpen(startcmd: *const c_char) -> *mut Engine;

    /// MATLAB エンジンを終了する
    pub fn engClose(ep: *mut Engine) -> c_int;

    /// MATLAB コマンドを実行する
    pub fn engEvalString(ep: *mut Engine, string: *const c_char) -> c_int;

    /// 変数を MATLAB ワークスペースから取得
    pub fn engGetVariable(ep: *mut Engine, name: *const c_char) -> *mut MxArray;

    /// 変数を MATLAB ワークスペースに設定
    pub fn engPutVariable(ep: *mut Engine, name: *const c_char, mp: *const MxArray) -> c_int;

    /// double スカラーの mxArray を作成
    pub fn mxCreateDoubleScalar(value: c_double) -> *mut MxArray;

    /// double 行列の mxArray を作成
    pub fn mxCreateDoubleMatrix(
        m: libc::size_t,
        n: libc::size_t,
        flag: c_int,
    ) -> *mut MxArray;

    /// mxArray のデータポインタを取得（double型）
    pub fn mxGetPr(pa: *const MxArray) -> *mut c_double;

    /// mxArray のスカラー値を取得
    pub fn mxGetScalar(pa: *const MxArray) -> c_double;

    /// mxArray のメモリを解放
    pub fn mxDestroyArray(pa: *mut MxArray);
}

/// mxArray フラグ: 実数
pub const MX_REAL: c_int = 0;
```

### 7-5. `src/main.rs`

```rust
// src/main.rs
// Rust から MATLAB Engine API を呼び出すデモ

mod matlab_ffi;

use matlab_ffi::*;
use std::ffi::CString;

fn main() {
    println!("=== Rust × MATLAB Engine デモ ===\n");

    unsafe {
        // ---- MATLAB エンジン起動 ----
        let ep = engOpen(std::ptr::null());
        if ep.is_null() {
            eprintln!("MATLAB エンジンの起動に失敗しました。");
            eprintln!("MATLAB_ROOT 環境変数と LD_LIBRARY_PATH を確認してください。");
            std::process::exit(1);
        }
        println!("✓ MATLAB エンジン起動完了");

        // ---- 例1: スカラー演算 ----
        // Rust 側でスカラーを作成し MATLAB に渡す
        let pi_val = mxCreateDoubleScalar(std::f64::consts::PI);
        let var_name = CString::new("r").unwrap();
        engPutVariable(ep, var_name.as_ptr(), pi_val);

        // 円の面積を計算
        let cmd = CString::new("area = pi * r^2;").unwrap();
        engEvalString(ep, cmd.as_ptr());

        // 結果を Rust 側に取得
        let area_name = CString::new("area").unwrap();
        let area_arr = engGetVariable(ep, area_name.as_ptr());
        if !area_arr.is_null() {
            let area = mxGetScalar(area_arr);
            println!("円の面積 (r=π): {:.6}", area);
            mxDestroyArray(area_arr);
        }
        mxDestroyArray(pi_val);

        // ---- 例2: 行列演算 ----
        // 2x2 行列を Rust で作成
        let mat = mxCreateDoubleMatrix(2, 2, MX_REAL);
        let ptr = mxGetPr(mat);
        // MATLAB は列優先（Column-major）
        // | 1  2 |
        // | 3  4 |
        std::ptr::write(ptr.add(0), 1.0_f64);
        std::ptr::write(ptr.add(1), 3.0_f64);
        std::ptr::write(ptr.add(2), 2.0_f64);
        std::ptr::write(ptr.add(3), 4.0_f64);

        let mat_name = CString::new("M").unwrap();
        engPutVariable(ep, mat_name.as_ptr(), mat);

        // 行列式を計算
        let det_cmd = CString::new("d = det(M); ev = eig(M);").unwrap();
        engEvalString(ep, det_cmd.as_ptr());

        let d_name = CString::new("d").unwrap();
        let d_arr = engGetVariable(ep, d_name.as_ptr());
        if !d_arr.is_null() {
            println!("det(M) = {:.1}", mxGetScalar(d_arr));
            mxDestroyArray(d_arr);
        }
        mxDestroyArray(mat);

        // ---- 例3: 数値積分 ----
        // integral(@(x) sin(x), 0, pi) ≈ 2.0
        let integ_cmd = CString::new(
            "I = integral(@(x) sin(x), 0, pi);"
        ).unwrap();
        engEvalString(ep, integ_cmd.as_ptr());

        let i_name = CString::new("I").unwrap();
        let i_arr = engGetVariable(ep, i_name.as_ptr());
        if !i_arr.is_null() {
            let integral_val = mxGetScalar(i_arr);
            println!("∫₀^π sin(x)dx = {:.10}", integral_val);
            mxDestroyArray(i_arr);
        }

        // ---- エンジン終了 ----
        engClose(ep);
        println!("\n✓ MATLAB エンジン終了");
    }
}
```

### 7-6. ビルドと実行

```bash
# 環境変数を設定
export MATLAB_ROOT="/usr/local/MATLAB/R2024b"
export LD_LIBRARY_PATH="${MATLAB_ROOT}/bin/glnxa64:$LD_LIBRARY_PATH"

# ビルド
cargo build --release

# 実行
cargo run --release
```

**期待される出力**

```
=== Rust × MATLAB Engine デモ ===

✓ MATLAB エンジン起動完了
円の面積 (r=π): 31.006277
det(M) = -2.0
∫₀^π sin(x)dx = 2.0000000000

✓ MATLAB エンジン終了
```

> **ヒント:** `bindgen` クレートを使うと `mex.h` / `engine.h` から FFI バインディングを自動生成できます。大規模プロジェクトでは検討してください。

---

## 8. トラブルシューティング

### Q1. `matlab: command not found`

```bash
# PATH が通っているか確認
echo $PATH | grep -i matlab

# 手動でフルパスを試す
/usr/local/MATLAB/R2024b/bin/matlab -batch "disp('hello')"

# .zshrc / .bashrc に PATH を追記して再読み込み
echo 'export PATH="/usr/local/MATLAB/R2024b/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc
```

### Q2. MEX ビルドで `mex.hpp` が見つからない

```bash
# ヘッダーの場所を確認
find /usr/local/MATLAB -name "mex.hpp" 2>/dev/null

# include パスを明示してビルド
matlab -batch "mex -I'/usr/local/MATLAB/R2024b/extern/include' vec_square_sum.cpp"
```

### Q3. Rust で `engOpen` が `null` を返す

```bash
# ライブラリパスを確認
ls /usr/local/MATLAB/R2024b/bin/glnxa64/libeng.so

# LD_LIBRARY_PATH を確認
echo $LD_LIBRARY_PATH

# 再設定
export LD_LIBRARY_PATH="/usr/local/MATLAB/R2024b/bin/glnxa64:$LD_LIBRARY_PATH"
```

### Q4. ライセンスエラー（`License checkout failed`）

```bash
# ライセンスサーバーの状態確認
/usr/local/MATLAB/R2024b/bin/matlab -batch "license"

# ネットワーク接続を確認
ping www.mathworks.com

# ライセンスファイルのパーミッション確認
ls -la /usr/local/MATLAB/R2024b/licenses/
```

### Q5. Apple Silicon (M1/M2/M3) での注意点

```bash
# MATLAB R2023b 以降は Apple Silicon ネイティブ対応
# それ以前は Rosetta 2 を使用
softwareupdate --install-rosetta

# MEX のコンパイラは Xcode Command Line Tools が必要
xcode-select --install
```

---

## 9. 参考リンク

| リソース | URL |
|----------|-----|
| MathWorks 公式ドキュメント | https://www.mathworks.com/help/matlab/ |
| MEX ファイル作成ガイド | https://www.mathworks.com/help/matlab/matlab_external/introducing-mex-files.html |
| MATLAB Engine API for C++ | https://www.mathworks.com/help/matlab/apiref/matlab.engine.matlabengine.html |
| Rust libc クレート | https://docs.rs/libc/latest/libc/ |
| Rust bindgen | https://rust-lang.github.io/rust-bindgen/ |
| MATLAB バッチモード | https://www.mathworks.com/help/matlab/ref/matlablinux.html |

---

*最終更新: 2025年 / MATLAB R2024b 対応*
