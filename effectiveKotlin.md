
### 가변성을 제한하라

코틀린의 요소 중 일부는 상태를 가진다. 요소가 상태를 가지면 요소의 동작은 사용 방법 뿐만 아니라 이력에도 의존한다.
상태가 있으면,
또한 프로그램을 이해하고 디버그하기 힘들어진다. 
코드의 실행을 추론하기 어려워진다. 
멀티 스레드일 때 동기화가 필요하다
테스트가 어렵다
상태가 바뀔 때, 변경을 다른 부분에 알려줘야 한다.

가변성은 일관성, 복잡성 증가 문제가 있다. 그러므로 신중히 사용해야 한다.

**코틀린에서 가변성 제어하기**

val
```kotlin
var name: String = "mar"  
var surname: String = "mos"  
  
val fullName  
    get() = "$name $surname"
```
fullName 은 val 이지만 값을 추출할 때 마다 custom getter 가 호출 되므로, 값이 계속 바뀔 수 있다.

```kotlin
fun calculate(): Int {  
    println("Calculating...")  
    return 42  
}  
  
val fizz = calculate()  
val buzz  
    get() = calculate()
```
fizz 는 선언할 때 바로 calculate() 호출하고 결과 저장한다. 나중에 fizz 를 부르면 그때 저장한 값만 반환한다. 하지만 buzz 는 부를 때 마다 custom getter 를 호출 해 calculate() 를 계산하고 값을 반환한다.

이와 같은 custom getter 덕분에 API 변경과 정의에 유연하다.

```kotlin
val fullName1: String?  
    get() = name1?.let { "$name1 $surname2" }  
  
val fullName2: String?= name1?.let { "$name1 $surname2" }

if (fullName1 != null) {  
    println(fullName1.length) // error
}  
  
if (fullName2 != null) {  
    println(fullName2.length)  
}
```
1 은 custom getter 를 가지고 있기 때문에 null 이 아니라고 해서 바로 스마트 캐스트 할 수 없다. 값을 사용하는 시점의 name 에 따라 다른 결과가 나올 수 있기 때문이다. 하지만 2 는 null 이 아니면 바로 String 으로 스마트 캐스트 가능하기 때문에 length 에 접근 할 수 있다.


mutable or imutable collection 구분하기

읽기 전용 컬렉션은 대부분 내부적으로는 값을 변경할 수 있는 컬렉션이다. 하지만 읽기 전용 인터페이스로 이를 감싸고 있기 때문에 겉으로 변경이 불가능 한 것이다.
내부 컬렉션을 진짜 불변하게 만들지 않는 것은 굉장히 중요한데, 이는 실제 컬렉션을 리턴할 수 있기 때문이다. 따라서 Java, Js, native 와 같은 플랫폼 고유의 컬렉션을 사용할 수 있다.
이것이 내부적으로 immutable 하지 않은 컬렉션을 외부적으로 immutable 하게 만들어서 얻어지는 안정성이다.

```kotlin
val list = listOf(1,2)  
if (list is MutableList) {  
    list.add(4) // error
}
```
이 코드의 실행 결과는 플랫폼에 따라 다르다. JVM 에서 listOf 는 자바의 List 를 구현한 ArrayList 를 리턴한다. 따라서 코틀린의 MutableList 로 변경할 수 있다. 하지만 ArrayList 는 이러한 연산을 구현하지 않기 때문에 에러가 생긴다.

코틀린에서 listOf 는 읽기 전용 List 인터페이스를 리턴한다. 내부 구현은 플랫폼 마다 다른데 JVM 에선 ArrayList 를 사용한다. 즉 수정 가능한 JVM 의 ArrayList 를 코틀린의 읽기 전용 List 인터페이스로 감싸는 것이다. 그래서 이 컬렉션에선 수정이 불가능 하다. ~~하지만 ArrayList 는 자바의 List 인터페이스를 구현하는데, 이때 타입 시스템의 애매함? 으로 코틀린의 Mutable 로 변경할 수 있게 된다. ~~

=> 읽기 전용 컬렉션을 mutable 컬렉션으로 다운캐스팅 하면 안 된다. 대신 copy 통해 새 mutable 컬렉션 만드는 걸로


데이터 클래스의 copy

immutable 쓰면 웬만해선 좋다. 

immutable 객체는 세트 또는 맵의 키로도 사용할 수 있다. 이들은 내부적으로 해시 테이블을 사용하기 때문에 mutable 한 객체를 사용하면, 값이 변경하면 더 이상 찾을 수 없다.

```kotlin
data class User(  
    val name: String,  
    val surname: String  
)
var user = User("maja", "mojo")  
user = user.copy(surname = "moska")
```
데이터 클래스는 copy 사용하면 좋다.


**다른 종류의 변경 가능 지점**

```kotlin
val list1: MutableList<Int> = mutableListOf()  
var list2: List<Int> = listOf()  
  
list1.add(1)  
list2 = list2 + 1  
  
list1 += 1 // list1.plusAssign(1)  
list2 += 1 // list2 = list2.plus(1)
```
1 은 리스트 내부에 변경 가능 지점이 있다. 따라서 멀티스레드 처리가 이루어질 때 위험할 수 있다. 하지만 2 는 프로퍼티 자체가 변경 가능 지점이다. 따라서 멀티스레드 처리의 안정성은 저 좋다.

변경 할 수 있는 리스트를 만들어야 할 땐,
mutable 컬렉션을 사용하기 보단,
immutable 컬렉션을 사용하고 mutable 프로퍼티를 사용하는게 좋다.


**변경 가능 지점 노출하지 말기**

```kotlin
class UserRepository {  
    private val storedUsers: MutableMap<Int, String> =  
        mutableMapOf()  
      
    fun loadAll(): Map<Int, String> {  
        return storedUsers  
    }  
}
```
mutable 객체를 외부에 노출 해야 할 땐,
컬렉션을 읽기 전용 슈퍼타입으로 업캐스트 한다.
혹은 copy 메서드를 활용한다.

**정리**

웬만하면 immutable 로 가자.
변경이 필요한 대상은 immutable 데이터 클래스 + copy
컬렉션은 읽기 전용 컬렉션 + mutable 프로퍼티
변이 지점은 최소한으로 적절하게




---
커스텀게터
델리게이트



스레드
코루틴