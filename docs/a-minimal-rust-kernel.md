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

## 실행 명령어

* 이전에 수행되어야 하는 것

**`bootimage` 설치**
```shell
cargo install bootimage
```

**`qemu` 설치**
[qemu](https://www.qemu.org/download/)가서 각자 OS에 맞게 설치
난 윈도우니까 msys2 를 통해서 설치하였다.  

```shell
pacman -S mingw-w64-x86_64-qemu 
```

그러면 이제 아래 명령어를 순차적으로 입력하면 영롱한 Hello World를 마주할 수 있게 된다. 

```shell
cargo build
cargo bootimage
qemu-system-x86_64 -drive format=raw,file=target\x86_64-riir_os\debug\bootimage-riir_os.bin
```

![안녕 세상아](../image/hello_world.png)
