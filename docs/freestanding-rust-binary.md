# Freestanding rust binary

Rust로 운영체제 커널을 작성하려면, 운영체제 없이도 실행가능한 실행파일이 필요하다. 이런 실행파일은 보통 "freestanding" 혹은 "bare-metal" 실행파일이라고 한다.  
운영체제를 만든다는 것은 운영체제에 의존하지 않는 다는 것이다. 모든 Rust 프로그램들은 Rust 표준 라이브러리를 링크하는데, 이 라이브러리는 스레드, 파일, 네트워킹 등의 기능을 제공하기 위해 운영체제에 의존적이다.

## Disabling the Standard Library

일단 이 의존성부터 끊어보자!

`#![no_std]` 라는 속성을 사용하면 Rust 표준 라이브러리가 링크되는 것을 막을 수 있다. 이걸 붙이고 `cargo build`를 하면 아래와 같은 오류가 난다.

```text
   Compiling riir_os v0.1.0 (C:\Users\User\RustroverProjects\riir_os)
error: cannot find macro `println` in this scope
 --> src\main.rs:3:5
  |
3 |     println!("Hello, world!");
  |     ^^^^^^^

error: `#[panic_handler]` function required, but not found

error: unwinding panics are not supported without std
  |
  = help: using nightly cargo, use -Zbuild-std with panic="abort" to avoid unwinding
  = note: since the core library is usually precompiled with panic="unwind", rebuilding your crate with panic="abort" may not be enough to fix the problem

error: could not compile `riir_os` (bin "riir_os") due to 3 previous errors
```

println 을 찾을 수 없다는 에러와 `#[panic_handler]`를 구현하라는 에러가 나온다.

println 매크로는 Rust 표준 라이브러리가 제공해주는 표준 입출력(운영체제가 제공하는 특별한 파일 서술자)으로 데이터를 쓰기 때문에 사용할 수가 없다. 


## Panic Implementation

운영체제로부터 독립하기 참 어렵다. 일단 그럼 println 매크로는 사용하지 말고, panic_handler를 구현해보자.

```rust
#![no_std]
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! { // PanicInfo는 패닉이 발생한 위치, 패닉 메시지를 가진 구조체이다. 이 함수는 절대로 반환하지 않기에 never 타입 ! 을 반환하여 컴파일러에게 이 함수가 반환 함수임을 알린다.
    loop {} // 현재 여기서 할 마땅한 일은 없기에 일단 무한 루프를 돌려둔다.
}

fn main() {}
```

panic이 일어날 때 저 panic_handler가 호출된다. 표준 라이브러리는 패닉 시 호출되는 함수가 제공되는데, 이제는 우리가 직접 구현해줘야 한다.
일단 저렇게 코드를 수정하고 `cargo build`를 하면 이런 에러를 마주한다.

```text
error: unwinding panics are not supported without std
  |
  = help: using nightly cargo, use -Zbuild-std with panic="abort" to avoid unwinding
  = note: since the core library is usually precompiled with panic="unwind", rebuilding your crate with panic="abort" may not be enough to fix the problem

error: could not compile `riir_os` (bin "riir_os") due to 1 previous error
```

에러 메시지를 살펴보면 std 없이 패닉 되감기를 지원하지 않는다고 한다. 기본적으로 Rust는 패닉이 일어났을 때 스택 되감기를 통해 스택에 살아있는 각 변수의 소멸자를 호출한다. 이를 통해
자식 스레드에서 사용중이던 모든 메모리 리소스가 반환되고, 부모 스레드가 패닉에 대처한 후 계속 실행될 수 있게 한다.
스택 되감기는 복잡한 과정으로 이뤄지며 OS마다 특정한 라이브러리를 필요로 하기에 우리가 구현할 OS에서는 이 기능을 사용하지 않기로 하자.

이 스택 되감기를 해제 하는 방법은 여럿 있지만 가장 간단한 방법인 `Cargo.toml`에 `panic="abort"`를 추가하는 방법을 사용하자.


```toml
[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

이렇게 하고 실행하면 또 새로운 오류가 우리를 마주해준다.


```text
   Compiling riir_os v0.1.0 (C:\Users\User\RustroverProjects\riir_os)
error: using `fn main` requires the standard library
  |
  = help: use `#![no_main]` to bypass the Rust generated entrypoint and declare a platform specific entrypoint yourself, usually with `#[no_mangle]`
```

우리의 프로그램에는 프로그램 실행 시 최초 실행 시작 지점을 지정해주는 `start` language item이 필요하다.

프로그램 실행 시 언제나 main 함수가 호출되는 것이 아니다. 대부분의 프로그래밍 언어들은 런타임 시스템을 가지고 있는데 이 런타임 시스템은 프로그램 실행 이전에 초기화 되어야 하기에 main 함수 호출 이전에 먼저 호출된다.  
러스트 표준 라이브러리를 링크하는 전형적인 러스트 실행 파일의 경우, 프로그램 실행 시 C 런타임 라이브러리인 `crt0` (C runtime zero)에서 실행시 시작된다.  
`crt0`는 C 프로그램의 환경을 설정하고 초기화하는 작업을 마친 후 `start` language item으로 지정된 Rust 런타임의 실행 시작 함수를 호출한다. Rust는 최소한의 런타임 시스템을 가지며, 주요 기능은 스택 오버플로우 가드를
초기화하고 패닉 시 역추적 정보를 출력하는 것이다. Rust 런타임의 초기화 작업이 끝난 후에야 `main` 함수가 호출된다.

우리 프로그램은 Rust 런타임이나 `crt0`에 접근할 수 없기에 지정해줘야 한다. `#![no_main]` 속성을 이용해 Rust 컴파일러에게 우리가 일반적인 호출 단계를 이용하지 않겠다고 선언해준다.

그리고 우리의 새로운 `_start` 함수를 실행 시작 지점으로 대체한다.

```rust
#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

`#[unsafe(no_mangle)]`으로 name mangling을 해제하여 Rust 컴파일러가 `_start` 라는 이름 그대로 함수를 만들도록 한다. 이 속성이 없다면 이 함수 이름을 다른 이름으로 바꿔 생성해버리게 되는데, 그러면
추후에 우리가 원하는 실제 시작 함수의 이름을 정확히 파악하고 linker에게 전달해줘야 하지만 그렇지 못하게 된다.

코드보면 `extern "C"`라는 문법을 사용했는데, 이 함수가 Rust 함수 호출 규약대신에 C 함수 호출 규약을 사용하도록 한 것이다.


`!` 반환 타입은 이 함수가 발산 함수라는 것을 의미한다. 시작 지점 함수는 오직 OS나 부트로더에 의해서만 직접 호출된다. 따라서 시작 지점 함수는 반환하는 대신의 운영체제의 `exit 시스템콜`을 이용해 종료된다.

이러고 실행하면 링커 오류를 마주하게 된다.

## Linker Errors

링커는 컴파일러가 생성한 코드들을 묶어 실행파일로 만드는 프로그램이다. 실행파일은 OS마다 다르기에 각 OS는 자신만의 링커를 가지고 있다. 현재 링커 오류가 발생하는 이유는 해당 프로그램이 C 런타임 시스템을 이용할 것이라고 가정하여,
그렇게 동작하는데 우리의 크레이트는 그렇지 않는다. 이 링커 오류를 해결하려면 링커에게 C 런타임을 링크하지 말라고 해야한다.

linker 오류를 해결하기 위해 `rustup target add thumbv7em-none-eabihf` 을 설치하고, `cargo build --target thumbv7em-none-eabihf`로 실행시킨다.  

`thumbv7em-none-eabihf`는 운영체제가 없는 bare metal 시스템 환경의 한 예시이다. (임베디드 ARM 시스템이다) 

이렇게 실행하면 이제 해당 target triple을 목표로 하는 freestanding 실행 파일을 만들 수 있다. `--target` 인자를 통해 우리가 해당 bare metal 시스템을 
목표로 크로스 컴파일할 것을 cargo에게 알려준다. 


## Summary

* `#![no_std]` attribute를 사용하여 운영체제에 의존적인 Rust 표준 라이브러리와 연을 끊었다. 
* `[panic_handler]` attribute를 사용하여 패닉 핸들러를 구현하였고 패닉 되감기 기능을 사용하지 않도록 하였다. 
* 시작 함수를 지정해주었다. 

**main.rs**

```rust
#![no_std]
#![no_main]
use core::panic::PanicInfo;

#[panic_handler]
fn panic(_info: &PanicInfo) -> ! {
    loop {}
}

#[unsafe(no_mangle)]
pub extern "C" fn _start() -> ! {
    loop {}
}
```

**Cargo.toml**

```toml
[package]
name = "riir_os"
version = "0.1.0"
edition = "2024"

[profile.dev]
panic = "abort"

[profile.release]
panic = "abort"
```

```shell
cargo build --target thumbv7em-none-eabihf
```

이렇게 freestanding rust 실행 파일 생성을 위한 첫걸을 내딛었다. 
