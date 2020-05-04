# Annäherung `Send`

Some `async fn` state machines are safe to be sent across threads, while others are not. Whether or not an `async fn` `Future` is `Send` is determined by whether a non-`Send` type is held across an `.await` point. The compiler does its best to approximate when values may be held across an `.await` point, but this analysis is too conservative in a number of places today.

Betrachten Sie beispielsweise einen einfachen Nicht- `Send` Typ, möglicherweise einen Typ, der einen `Rc` enthält:

```rust
use std::rc::Rc;

#[derive(Default)]
struct NotSend(Rc<()>);
```

Variablen vom Typ `NotSend` kann als kurz erscheinen Provisorien in `async fn` s , selbst wenn die resultierende `Future` Typ durch die zurück `async fn` müssen `Send` :

```rust
async fn bar() {}
async fn foo() {
    NotSend::default();
    bar().await;
}

fn require_send(_: impl Send) {}

fn main() {
    require_send(foo());
}
```

Wenn wir jedoch `foo` ändern, um `NotSend` in einer Variablen zu speichern, wird dieses Beispiel nicht mehr kompiliert:

```rust
async fn foo() {
    let x = NotSend::default();
    bar().await;
}
```

```
error[E0277]: `std::rc::Rc<()>` cannot be sent between threads safely
  --> src/main.rs:15:5
   |
15 |     require_send(foo());
   |     ^^^^^^^^^^^^ `std::rc::Rc<()>` cannot be sent between threads safely
   |
   = help: within `impl std::future::Future`, the trait `std::marker::Send` is not implemented for `std::rc::Rc<()>`
   = note: required because it appears within the type `NotSend`
   = note: required because it appears within the type `{NotSend, impl std::future::Future, ()}`
   = note: required because it appears within the type `[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]`
   = note: required because it appears within the type `std::future::GenFuture<[static generator@src/main.rs:7:16: 10:2 {NotSend, impl std::future::Future, ()}]>`
   = note: required because it appears within the type `impl std::future::Future`
   = note: required because it appears within the type `impl std::future::Future`
note: required by `require_send`
  --> src/main.rs:12:1
   |
12 | fn require_send(_: impl Send) {}
   | ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

error: aborting due to previous error

For more information about this error, try `rustc --explain E0277`.
```

Dieser Fehler ist korrekt. Wenn wir `x` in einer Variablen speichern, wird es erst nach dem `.await` . Zu diesem Zeitpunkt wird das `async fn` möglicherweise auf einem anderen Thread ausgeführt. Da `Rc` nicht `Send` , wäre es nicht sinnvoll, es über Threads laufen zu lassen. Eine einfache Lösung für dieses Problem wäre, `drop` die `Rc` vor dem `.await` , aber leider , dass heute nicht arbeiten.

In order to successfully work around this issue, you may have to introduce a block scope encapsulating any non-`Send` variables. This makes it easier for the compiler to tell that these variables do not live across an `.await` point.

```rust
async fn foo() {
    {
        let x = NotSend::default();
    }
    bar().await;
}
```
