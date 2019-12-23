kotlin 의 scope function (apply, with, let, also, run)

일단 모두 비슷함. receiver와 code block을 받음. 그리고 받은 receiver로 받은 code block을 실행함.

with를 보면 이렇게 정의되어 있음
<code>
inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}
</code>


5가지 함수의 차이

예를 들면 with와 also의 구현은 이렇게 다름

<code>
inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}
inline fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}
</code>

with 에서는 receiver가 explicit parameter로 들어감, also 는 implicit parameter(this) 로 들어감

with에서는 block argument가 implicit receiver T의 함수로 전달됨
also에서는 받는 block은 그냥 this를 받아서 실행됨(이런 식을 explicit 하다고 하나봄ㅜ 잘 이해못함)

with에서는 return을 할 때 code block을 실행한 결과를, also 는 code block을 돌릴때 들어갔던 receiver를 리턴함


그래서 with는 이런 식으로 쓰일 수 있는 반면에,
<code>
val person: Person = getPerson()
with(person) {
    print(name)
    print(age)
}
</code>
also는 이런 방식으로 쓰여야 함
<code>
val person: Person = getPerson().also {
    print(it.name)
    print(it.age)
}
</code>

with, also, apply, let, run은 대충 위에서 말했던 3가지 중에 차이점이 보인다고함

이를테면 일단 이렇게 구현이 다 다름
<code>
inline fun <T, R> with(receiver: T, block: T.() -> R): R {
    return receiver.block()
}
inline fun <T> T.also(block: (T) -> Unit): T {
    block(this)
    return this
}
inline fun <T> T.apply(block: T.() -> Unit): T {
    block()
    return this
}
inline fun <T, R> T.let(block: (T) -> R): R {
    return block(this)
}
inline fun <T, R> T.run(block: T.() -> R): R {
    return block()
}
</code>

표로 보면 쉬움. https://docs.google.com/spreadsheets/d/1P2gMRuu36pSDW4fdwE-fLN9fcA_ZboIU2Q5VtgixBNo/edit?usp=sharing


그래서 언제 뭘 써야 하지...
-> 일단 공식 문서에 규칙이 있음 https://kotlinlang.org/docs/reference/coding-conventions.html

apply를 쓰고 싶을 땐
-> receiver에 정의된 함수는 안 쓰고 그냥 같은 receiver를 리턴하고 싶을때. 예를 들면 receiver로 넘어가는 object 의 property만 건드리고 다시 그 receiver object를 리턴할때 (== initialize할 때) 쓸 수 있음

<code>
val peter = Person().apply {
    // only access properties in apply block!
    name = "Peter"
    age = 18
}
</code>

<code>
val clark = Person()
clark.name = "Clark"
clark.age = 18
</code>
이거랑 같은것임 .. 파이썬 생각나는군 (대충 별로라는 뜻)


also를 쓰고 싶을 땐
receiver 안의 property를 변경하고 싶지 않고(변경하고 싶으면 also쓰면 안됨) 그대로 receiver를 리턴하고 싶을 경우. 모 예를 들면 .. property 값들을 확인하고 싶을 수가 있겠지 validation 이랄지
<code>
class Book(author: Person) {
    val author = author.also {
      requireNotNull(it.age)
      print(it.name)
    }
}
</code>

<code>
class Book(val author: Person) {
    init {
      requireNotNull(author.age)
      print(author.name)
    }
}
</code>
이거랑 같은 의미임 (apply와 also는 많이 다르다는 것을 알겠음)

이제 let 을 써야 하는 경우에 대해 알아볼까 ^~^?

receiver가 not null 일때만 뭔가를 하고 싶을 때
<code>
getNullablePerson()?.let {
    // only executed when not-null
        promote(it)
}

// let 을 안 쓴다면
val person: Person? = getPromotablePerson()
if (person != null) {
  promote(person)
}
</code>

nullable object -> nullable object로 바꾸고 싶을때
<code>
val driversLicence: Licence? = getNullablePerson()?.let {
    // convert nullable person to nullable driversLicence
        licenceService.getDriversLicence(it) 
}

// let 을 안 쓴다면

val driver: Person? = getDriver()
val driversLicence: Licence? = if (driver == null) null else
    licenceService.getDriversLicence(it)
</code> 

local variable의 scope를 제한하고 싶을때
<code>
val person: Person = getPerson()
getPersonDao().let { dao -> 
    // scope of dao variable is limited to this block
        dao.insert(person)
}

//let 을 안 쓴다면
val person: Person = getPerson()
val personDao: PersonDao = getPersonDao()
personDao.insert(person)
</code>

null check 때문에라도 let 쓰는게 코드를 예쁘게 쓸 수 있어서 좋은듯


with를 쓰고 싶을 때

not null이 확실할 때에만 쓰고, with 내에서 실행한 결과를 딱히 어디에 할당할 필요가 없을 때
<code>
val person: Person = getPerson()
with(person) {
    print(name)
        print(age)
}
</code>


run은? (run 잘 모름 일할때 안써본듯)
변수값들을 가지고 계산을 해야 한다든지, 변수들의 범위를 제한해야 할때, 그리고 explicit parameter를 implicit parameter 로 변환하고 싶을 때

<code>
val inserted: Boolean = run {
    val person: Person = getPerson()
    val personDao: PersonDao = getPersonDao()
    personDao.insert(person)
}
fun printAge(person: Person) = person.run {
    print(age)
}

// run을 안 쓴다면
val person: Person = getPerson()
val personDao: PersonDao = getPersonDao()
val inserted: Boolean = personDao.insert(person)
fun printAge(person: Person) = {
    print(person.age)
}
</code>

모 이건 다른 애들처럼 글케 많이는 쓸모 없는거같음

일케 각각 scoping functions 의 쓰임새들을 알아보았다
가독성도 높여주고 참 좋은 애들이다
근데 nested scoping functions 는 ...웬만하면 하지마라 헷갈리니까 
apply, run, with 는 섞지 말아라
let과 also를 섞고 싶을 때는 it 보단 명시적인 이름을 쓰도록 하자

그렇지만 call chain에 scoping function들을 섞으면 오히려 읽기 좋고 편하다
예시)
<code>
private fun insert(user: User) = SqlBuilder().apply {
  append("INSERT INTO user (email, name, age) VALUES ")
  append("(?", user.email)
  append(",?", user.name)
  append(",?)", user.age)
}.also {
  print("Executing SQL update: $it.")
}.run {
  jdbc.update(this) > 0
}
</code>
새 object를 database 에 넣는 dao function (dao란?Data Access Object로 database 추상 인터페이스를 제공한다. 그럼 orm 의 이전 세대에서 db를 이용한 개발에 활용된걸까?) 인데 SQL 준비, 실행 로깅, SQL 실행 을 각각 scope를 나누어서 실행하고 있다. 너무 좋으내 
