## Item1. 가변성을 제한하라

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

## Item2. 변수의 스코프를 최소화하라

- 프로퍼티보다는 지역 변수가 좋다. 
- 최대한 좁은 스코프를 갖게 변수를 사용해야 한다.

```kotlin
//나쁜 예  
var user: User  
for (i in users.indices){  
    user = users[i]  
    print("User at $i is $user")  
}  
//조금 더 좋은 예  
for (i in users.indices){  
    val user = users[i]  
    print("User at $i is $user")  
}  
//좋은 예  
for ((i, user) in users.withIndex()){  
    print("User at $i is $user")  
}
```

스코프를 좁게 만들 때 가장 좋은 것은 프로그램을 추적하고 관리하기 쉽게 한다. 애플리케이션이 간단할수록 읽기도 쉽고 안전하다. mutable 프로퍼티보다 immutable 프로퍼티가 더 좋은 이유와 마찬가지다.

변수의 스코프가 좁으면 변경을 추적하는 것이 쉽고, 변수의 스코프가 넓으면 다른 개발자에 의해 변수가 잘 못 사용되는 등 추적이 어렵다.

변수는 최대한 정의할 때 초기화하는 것이 좋다.

```kotlin
//나쁜 예
val user2: User  
if (hasValue){  
    user2 = getValue()  
} else {  
    user2 = User()  
}  
//조금 더 좋은 예
val user3: User = if (hasValue) getValue() else User()
```


#### 캡처링

```kotlin
val user3: User = if (hasValue) getValue() else User()  
  
var numbers = (2..100).toList()  
val primes = mutableListOf<Int>()  
while(numbers.isNotEmpty()){  
    val prime = numbers.first()  
    primes.add(prime)  
    numbers = numbers.filter { it % prime != 0 }  
}  
  
val primes2: Sequence<Int> = sequence {  
    var numbers = generateSequence(2) { it + 1 }  
  
    while(true){  
        val prime = numbers.first()  
        yield(prime)  
  
        numbers = numbers.drop(1).filter { it % prime != 0 }  
    }  
}  
print(primes.take(10).toList()) // [2,3,5,7...]  
  
val primes3: Sequence<Int> = sequence {  
    var numbers = generateSequence(2) { it + 1}  
  
    var prime: Int  
    while(true){  
        prime = numbers.first()  
        yield(prime)  
  
        numbers = numbers.drop(1).filter { it % prime != 0 }  
    }  
}  
print(primes.take(10).toList()) // [2,3,5,6,7,8...]
```
에라토스체네스의 채를 구현할 때 1번처럼 해도 되지만, 시퀀스를 활용하면 2번처럼도 가능하다.

~~하지만 3번처럼 한다면 실패하는데, prime 변수를 캡처했기 때문이다.
반복문 내부에서 filter 를 활용해서 prime 으로 숫자를 필터링한다. 그런데 시퀀스이므로 필터링이 지연된다. 따라서 최종적인 prime 값으로만 필터링 된 것이다. 
prime 이 2로 설정되어있을 때 필터링된 4를 제외하면, drop 만 동작하므로 그냥 연속된 숫자가 나와 버린다.~~
걍 1도 이해 안 됨.

## Item3. 최대한 플랫폼 타입을 사용하지 마라

null-safety 는 코틀린의 주요 기능 중 하나이다. 그러나 이러한 메커니즘이 없는 자바, C 등의 언어와 코틀린을 연결해서 사용할 때는 NPE 가 발생할 수 있다.

```kotlin
fun statedType(){  
    val value: String = JavaClass().value //NPE
    println(value.length)  
}  
fun platformType(){  
    val value = JavaClass().value 
    println(value.length) //NPE
}
```
오류 발생 위치가 다르다.


## Item4. inferred 타입으로 리턴하지 마라

할당 때 inferred 타입은 정확하게 오른쪽에 있는 피연산자에 맞게 설정된다.

```kotlin
open class Animal  
class Zebra : Animal()  
fun main(){  
    var animal = Zebra()  
    animal = Animal() // error  
}
```

타입을 확실하게 지정해야 하는 경우엔 명시적으로 타입을 지정 해야 한다.
리턴 타입은 외부에서 확인할 수 있게 명시적으로 지정해야 한다.

## Item5. 예외를 활용해 코드에 제한을 걸어라

확실하게 어떤 형태로 동작해야 한다면, 제한을 걸어야 한다.
- require : argument 를 제한할 수 있다.
- check : 상태를 제한할 수 있다.
- assert : 테스트에서 어떤 것이 true 인지 확인할 수 있다.
- return or throw 와 함께 활용하는 Elvis 연산자

함수를 정의할 때 타입 시스템을 통해 argument 에 제한을 건다.
```kotlin
fun factorial(n: Int): Long {  
    require(n >= 0)  
    return if (n <= 1) 1 else factorial(n - 1)  
}  
  
fun findClusters(points: List<Point>): List<Cluster> {  
    require(points.isNotEmpty())  
    return listOf(Cluster())  
}  
  
fun sendEmail(user: User, message: String){  
    requireNotNull(user.email)  
    require(isValidEmail(user.email))  
}
```

require 함수는 조건을 만족하지 못할 때 무조건적으로 IllegalArgumentException 을 발생시키므로 제한을 무시할 수 없다.


상태와 관련된 제한을 걸 때는 check 를 사용한다.

```kotlin
fun speak(text: String){
	check(isInitialized)
}

fun getUserInfo(): UserInfo {
	checkNotNull(token)
}

fun next(): T {
	check(isOpen)
}
```

check 함수는 require 와 비슷하지만 조건을 만족하지 못할 때 IllegalStateException 을 throw 하기 때문에 상태를 확인할 때 사용한다.

이러한 확인은 사용자가 잘 못 사용한다고 의심될 때 한다. 항상 문제를 예측하고 에외를 던져야 한다.


사용자가 아닌 스스로 구현한 내용을 확인할 땐 assert 를 사용한다.

```kotlin

class StackTest {
	@Test
	fun 'Stack pops correct number of elements'() {
		val stack = Stack(20) { it }
		val ret = stack.pop(10)
		assertEquals(10, ret.size)
	}
}
```

단위 테스트는 구현의 정확성을 확인하는 가장 기본적인 방법이다. 하지만 이것만으로는 괜찮을지 모른다.

```kotlin
fun pop(num: Int = 1): List<T> {
	assert(ret.size == num)
	return ret
}
```

단위 테스트 대신 함수에서 테스트를 한다면,
- 코드를 자체 점검할 수 있다.
- 모든 상황에 테스트를 할 수 있다.
- 실제 코드가 더 빠른 시점에 실패하게 만든다.

정말 심각한 오류라면 check 를 사용해야 한다.


require 와 check 블록으로 조건을 확인해서 true 가 나왔다면, 이후로도 true 일 것이라고 가정한다. 따라서 타입 비교를 했다면, 스마트 캐스트가 작동한다.

```kotlin
class Person(  
    val email: String?  
)  
fun validateEmail(email: String) {}  
fun sendEmail(person: Person, message: String){  
    require(person.email != null)  
    val email: String = person.email  
}  
fun sendEmail2(person: Person, message: String){  
    val email = requireNotNull(person.email)  
    validateEmail(email)  
}  
fun sendEmail3(person: Person, message: String){  
    requireNotNull(person.email)  
    validateEmail(person.email)  
}
```
