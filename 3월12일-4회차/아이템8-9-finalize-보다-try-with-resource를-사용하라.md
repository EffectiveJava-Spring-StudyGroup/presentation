# 아이템8,9 finalize 보다 try-with-resource를 사용하라

- 아이템8 finalizer와 cleaner 사용을 피하라
- 아이템9 try-finally 보다 try-with-resource를 사용하라



## 개념을 간단히 설명드립니다.

```java
static String firstLineOfFile(String path) throws IOException {
	BufferedReader br = new BufferedReader(new FileReader(path));
	try {
		return br.readLine();
	} finally {
		br.close();
	}
}
```

- 만약 위 코드에서 자원을 명시적으로 close하지 않으면 
  - finalizer로 처리되지만, 
  - 최후의 최후의 안전망으로만 여겨야한다.

- 위와 같이 try/finally로 직접 자원을 close하게 되면, 
  - 자원 객체에 오류가 있어 readLine 메서드에서 예외가 발생해도 
  - close 구문의 예외로 덮어씌워진다.
  - 참고) [book-effective-java/9_try-finally보다는 try-with-resources를 사용하라_김보배.md at main · Meet-Coder-Study/book-effective-java (github.com)](https://github.com/Meet-Coder-Study/book-effective-java/blob/main/2장/9_try-finally보다는 try-with-resources를 사용하라_김보배.md)
- 대신 autoclosable 인터페이스를 구현한 경우 try/resource 구문으로 자동 close하면
  - close에서 발생한 예외는 suppressed되었다고 표기되고
  - readLine에서 발생한 예외가 잘 출력된다.



## 회사에서 이렇게 사용해봤어요.

- 원격지 서버에서 파워쉘 스크립트를 실행하고 표준출력/에러를 읽어오는 자바 프로그램
  - try/resource를 사용해서 InputStream 자원을 해제했고
  - Process에도 적용해보려했는데 안돼서 try/finally로 처리함
  - 이제 autoclosable 인터페이스를 구현해 확장하면 된다는걸 알게됨, 다음번에 개선해볼 예정



## 더 공부해볼 주제들이 많이 있었습니다.

- **만약 자원을 close하지 않는다면?**
  - 파일 입출력의 경우 lock이 걸려 새로운 작업을 하지 못하거나
  - out of memory가 발생할 수도 있다 (운영체제/JVM 벤더사에 따라 다르지만)

- **자원 레퍼런스가 유효범위 밖으로 벗어나도 계속 유지될까?**
  - :bulb: 레퍼런스가 유효범위 밖으로 밀려나도 힙영역의 객체는 gc때 mark/sweap된다.
  - gc되기 전에 다른 프로그램이나 스레드에서 접근할 수 있으므로 제때제때 해제해야한다.
- **finalizer는 어떤 방식으로 호출되길래 의도한대로 동작을 안할까?**
  - finalizer와 cleaner는 gc때 수행된다. gc 시점은 정확히 알 수 없다.
    - gc에서 메모리를 해제하려고 할 때
    - finalize 메서드가 정의되어 있으면 finalization queue에 들어간다
    - 이후 finalizer가 finalize 메서드를 실행하고 메모리 정리 작업을 수행한다
  - :bulb: 이때 finalize 메서드 실행중 발생하는 예외는 무시되며, 처리할 작업이 남아도 그순간 종료된다.
    - 수행여부가 보장되지 않는다.
    - 게다가 autoclosable을 사용하여 처리할때와 비교하면 50배 가량 느리다
  - finalizer 공격에 노출되어 보안 문제를 일으킬 수 있다. 생성자/직렬화 과정에서 예외가 발생하면 생성되다만 객체에서 악의적인 하위 클래스의 finalizer가 수행될 수 있다.
  - 그럼에도 불구하고 누군가 자원을 close하지 않았을 때를 대비한 최후의 최후의 안전망이다.

- **cleaner는 또 어떤 방식이길래 의도한대로 동작을 안할까?**
  - cleaner 역시 gc때 수행된다.
  - clenaer에서 새로운 Runnable 스레드를 실행해 자원을 정리한다.
  - :bulb: 스레드에서 대상 인스턴스에 접근하면 순환참조가 발생해 gc 대상이 되지 못한다.
    - 아래 코드를 보면
    - Cleaner 인스턴스에서 CleaningExample 객체의 State 스레드를 실행했다.
    - 그런데 State 스레드에서 다시 CleaningExample 객체를 참조하면 순환참조가 발생한다.
    - Cleaner 인스턴스가 메모리에서 해제되기까지 gc 대상이 되지 못한다.
  - 그래서 Statesms static 클래스로 선언했는데, 정적이 아닌 중첩클래스는 자동으로 바깥 객체의 참조를 가지기 때문이다.

```java

public class CleaningExample implements AutoCloseable {
    // A cleaner, preferably one shared within a library
    private static final Cleaner cleaner = <cleaner>;

    static class State implements Runnable {

        State(...) {
            // initialize State needed for cleaning action
        }

        public void run() {
            // cleanup action accessing State, executed at most once
        }
    }

    private final State;
    private final Cleaner.Cleanable cleanable

    public CleaningExample() {
        this.state = new State(...);
        this.cleanable = cleaner.register(this, state);
    }

    public void close() {
        cleanable.clean();
    }
}
```

- **그런데 finalizer와 cleaner가 꼭 필요할 때가 있다. gc가 해제하는 방법을 모를 때!**

  - 바로 네이티브 피어 자원을 해제할 때다.
  - 네이티브 피어란 
    - C/C++, 어셈블리 프로그램을 컴파일한 기게어 프로그램으로서
    - 이를 라이브러리로서 자바가 실행할 수 있게 해주는 인터페이스를 JNI라고 한다
    - 자바 피어가 로딩될때 정적으로 System.loadLibrary() 메서드를 호출해 로딩

  ```c
  #include <stdio.h>
  #include <jni.h>
  #include "HelloJNI.h"
  
  JNIEXPORT void JNICALL Java_HelloJNI_helloFromC(JNIEnv *env, jclass ojb) {
      printf("%s", "Hello from C!\n");
  }
  
  ```

  ```java
  public class HelloJNI {
  
      static {
          System.loadLibrary("native");
      }
  
      public static native void helloFromC();
  
      public static void main(String[] args) {
          helloFromC();
      }
  
  }
  ```

  



















