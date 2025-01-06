# Rustでのコードを書いて見て感じること

型安全やメモリ安全、マルチスレッドでの安全性がRustのメリットで良く上げられる。これはもちろんだけど、プロブラマーとしては生産性や書き味にも注目したいと思いメモしておく。

私のようにC++を書いていた人間がRustでコードを記載していくとかゆい所に手が届くと思うことが非常に多い。人によって何を始めに感じるか色々あると思うけど例えば、

- crate(C++のライブラリ相当)の追加が簡単
    `Cargo.toml`で以下のように書くとよい
    ```
    [dependencies]
    serde = { version = "1.0", features = ["derive"] } 
    ```
    C++みたいにh/lib/dllをコピーしたりパスを通す必要はない

- コードにメタ情報を追加できる
    コード中にメタ情報を記載すると自動生成が強力
    ```
    #[derive(Debug)]
    struct Point {
        x: i32,
        y: i32,
    }
    ```
    は
    ```
    impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
    ```
    を自動生成する

といったことができるため非常にコードがサクサク書ける。

# エラーハンドリング

Rustでは以下のようなサンプルコードをよくみると思う。

```
fn to_number(str: &str) {
    let n = str.parse::<i32>().unwrap();
    print!("number {}", n);
}
```
`unwrap`は成功時は値を取り出し失敗時は`panic`する。回復不可能なエラーのときに`panic`を用いるプロダクションコードでは`panic`を適用できる範囲が狭いためにエラーハンドリングが必要になる。

```
fn to_number(str: &str) {
    let n = match str.parse::<i32>() {
        Ok(v) => v,
        Err(_) => return,
    };

    print!("number {}", n);
}
```

C++をやっていると上のようにエラーハンドリングしていくことがしみついている。もうちょっとこなれた書き方をすると、

```
fn to_number(input: &str) {
    let n = input.parse::<i32>().map_err(|_| return) ?;

    println!("number {}", n);
}
```
`map_err`はResult型のエラー型が返っている場合に実行される処理であり、ここでは
即リターンするコードにしている。


ただ、上記のコードも若干問題があり、普通は`to_number`はエラー時はエラーを返すとしたい。

```
fn main() {
    match to_number("1123") {
        Ok(n) => println!("Parsed number: {}", n),
        Err(e) => eprintln!("Error: {}", e),
    }
}

fn to_number(input: &str) -> Result<i32, String> {
    input.parse::<i32>().map_err(|e| format!("Failed to parse '{}': {}", input, e))
}
```

今は文字列だけど標準のエラー型`std::error`を用いると以下のように書ける

```
use std::error::Error;

fn main() {
    match to_number("1123") {
        Ok(n) => println!("Parsed number: {}", n),
        Err(e) => eprintln!("Error: {}", e),
    }
}

fn to_number(input: &str) -> Result<i32, Box<dyn Error>> {
    input.parse::<i32>().map_err(|e| {
        Box::new(e) as Box<dyn Error>
    })
}
```

`thiserror`を用いると
```
use thiserror::Error;
use std::num::ParseIntError;

#[derive(Error, Debug)]
pub enum ParseError {
    #[error("Parse error: {0}")]
    ParseIntError(#[from] ParseIntError),
}

fn main() {
    match to_number("1123") {
        Ok(n) => println!("Parsed number: {}", n),
        Err(e) => eprintln!("Error: {}", e),
    }
}

fn to_number(input: &str) -> Result<i32, ParseError> {
    let n = input.parse::<i32>()?;
    Ok(n)
}
```

`?`演算子はResultに対してはOkの場合は値をそのまま返し、Errの場合は呼び出し元へErr型を返すような処理になる。

※余談
?演算子はerror propagation operator/question mark operatorと呼ばれる昔はtry!マクロを使用していた
```
macro_rules! try {
    // `Result` 型が `Err` の場合、エラーを返す
    ($e:expr) => ({
        match $e {
            Ok(val) => val,
            Err(e) => return Err(e),
        }
    });
}
```
