# A Minimal Rust Kernel

## The Boot Process

전원이 켜졌을 때 컴퓨터는 젤 먼저 마더보드의 롬(ROM)에 저장된 펌웨어 코드를 실행한다.



`x86_64-riir_os.json` 을 활용하여 cargo build를 하면 아래와 같은 에러가 발생한다. 
```text
error[E0463]: can't find crate for `core`
  |
  = note: the `x86_64-riir_os` target may not be installed
  = help: consider downloading the target with `rustup target add x86_64-riir_os`
  = help: consider building the standard library from source with `cargo build -Zbuild-std`

For more information about this error, try `rustc --explain E0463`.
```
