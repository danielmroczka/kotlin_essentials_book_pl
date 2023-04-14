## Klasy i interfejsy szczelne

Klasy i interfejsy w Kotlinie nie służą tylko do reprezentowania zestawu operacji lub danych; możemy również używać klas i dziedziczenia do reprezentowania hierarchii za pomocą polimorfizmu. Na przykład, powiedzmy, że wysyłasz żądanie sieciowe; w rezultacie albo otrzymujesz żądane dane, albo żądanie kończy się niepowodzeniem z informacjami o tym, co poszło nie tak. Te dwa rezultaty można przedstawić za pomocą dwóch klas implementujących interfejs:

```kotlin
interface Result
class Success(val data: String) : Result
class Failure(val exception: Throwable) : Result
```

Alternatywnie można użyć klasy abstrakcyjnej:

```kotlin
abstract class Result
class Success(val data: String) : Result()
class Failure(val exception: Throwable) : Result()
```

W przypadku każdego z nich wiemy, że gdy funkcja zwraca `Result`, może być to `Success` lub `Failure`.

```kotlin
val result: Result = getSomeData()
when (result) {
    is Success -> handleSuccess(result.data)
    is Failure -> handleFailure(result.exception)
}
```

Problem polega na tym, że przy użyciu zwykłego interfejsu lub klasy abstrakcyjnej nie ma gwarancji, że zdefiniowane podklasy są wszystkimi możliwymi podtypami tego interfejsu lub klasy abstrakcyjnej. Ktoś może zdefiniować inną klasę i sprawić, że zaimplementuje `Result`. Ktoś może także zaimplementować wyrażenie obiektu, które implementuje `Result`.

```kotlin
class FakeSuccess : Result

val res: Result = object : Result {}
```

Hierarchia, której podklasy nie są znane z góry, nazywana jest hierarchią nieograniczoną. Dla `Result` wolelibyśmy zdefiniować hierarchię ograniczoną, co robimy, używając modyfikatora `sealed` przed klasą lub interfejsem[^14_0].

```kotlin
sealed interface Result
class Success(val data: String) : Result
class Failure(val exception: Throwable) : Result

// lub

sealed class Result
class Success(val data: String) : Result()
class Failure(val exception: Throwable) : Result()
```

> Gdy używamy modyfikatora `sealed` przed klasą, sprawia to, że klasa staje się już abstrakcyjna, więc nie używamy modyfikatora `abstract`.

Wszystkie dzieci klasy szczelnej lub interfejsu szczelnego muszą spełniać kilka wymagań:
* muszą być zdefiniowane w tym samym pakiecie i module, co szczelna klasa lub interfejs,
* nie mogą być lokalne ani zdefiniowane za pomocą wyrażenia obiektu.

Oznacza to, że gdy używasz modyfikatora `sealed`, kontrolujesz, jakie podklasy ma klasa lub interfejs. Klienci Twojej biblioteki lub modułu nie mogą dodać własnych bezpośrednich podklas[^14_2]. Nikt nie może cicho dodać lokalnej klasy ani wyrażenia obiektu, które rozszerza szczelną klasę lub interfejs. Kotlin uczynił to niemożliwym. Hierarchia podklas jest ograniczona.

> Szczelne interfejsy zostały wprowadzone w nowszych wersjach Kotlina, aby umożliwić klasom implementowanie wielu szczelnych hierarchii. Relacja między szczelną klasą a szczelnym interfejsem jest podobna do relacji między klasą abstrakcyjną a interfejsem. Mocą klas jest to, że mogą przechowywać stan (właściwości nieabstrakcyjne) i kontrolować otwartość swoich elementów (mogą mieć metody i właściwości końcowe). Mocą interfejsów jest to, że klasa może dziedziczyć tylko z jednej klasy, ale może implementować wiele interfejsów.

### Szczelne klasy i wyrażenia `when`

Użycie `when` jako wyrażenia musi zwrócić jakąś wartość, więc musi być wyczerpujące. W większości przypadków jedynym sposobem na osiągnięcie tego jest określenie klauzuli `else`.

```kotlin
fun commentValue(value: String) = when {
    value.isEmpty() -> "Nie powinno być puste"
    value.length < 5 -> "Zbyt krótkie"
    else -> "Poprawne"
}

fun main() {
    println(commentValue("")) // Nie powinno być puste
    println(commentValue("ABC")) // Zbyt krótkie
    println(commentValue("ABCDEF")) // Poprawne
}
```

Jednak istnieją również przypadki, w których Kotlin wie, że określiliśmy wszystkie możliwe wartości. Na przykład, gdy używamy wyrażenia when z wartością enum i porównujemy tę wartość do wszystkich możliwych wartości enum.

```kotlin
enum class PaymentType {
    GOTÓWKA,
    KARTA,
    CZEK
}

fun commentDecision(type: PaymentType) = when (type) {
    PaymentType.GOTÓWKA -> "Zapłacę gotówką"
    PaymentType.KARTA -> "Zapłacę kartą"
    PaymentType.CZEK -> "Zapłacę czekiem"
}
```

Moc posiadania skończonego zestawu typów jako argument sprawia, że wyczerpujące `when` z gałęzią dla każdej możliwej wartości jest możliwe. W przypadku klas szczelnie zamkniętych lub interfejsów oznacza to posiadanie sprawdzeń `is` dla wszystkich możliwych podtypów.

```kotlin
sealed class Response<out V>
class Success<V>(val value: V) : Response<V>()
class Failure(val error: Throwable) : Response<Nothing>()

fun handle(response: Response<String>) {
    val text = when (response) {
        is Success -> "Sukces z ${response.value}"
        is Failure -> "Błąd"
        // else nie jest potrzebne tutaj
    }
    print(text)
}
```

Ponadto IntelliJ automatycznie sugeruje dodanie pozostałych gałęzi. To sprawia, że klasy szczelnie zamknięte są bardzo wygodne, gdy musimy pokryć wszystkie możliwe typy.

![](remaining_branches.png)

Zauważ, że gdy klauzula `else` nie jest używana, a my dodajemy kolejną podklasę tej klasy szczelnie zamkniętej, należy dostosować jej użycie, uwzględniając ten nowy typ. Jest to wygodne w lokalnym kodzie, ponieważ zmusza nas do obsługi tej nowej klasy w wyczerpujących wyrażeniach `when`. Niewygodne jest to, że gdy ta klasa szczelnie zamknięta jest częścią publicznego API biblioteki lub współdzielonego modułu, dodanie podtypu stanowi łamanie zgodności, ponieważ wszystkie moduły używające wyczerpującego `when` muszą obsłużyć jeden więcej możliwy typ.

### Szczelnie zamknięte vs enum

Klasy Enum służą do reprezentowania zestawu wartości. Klasy szczelnie zamknięte lub interfejsy reprezentują zestaw podtypów, które można utworzyć za pomocą klas lub deklaracji obiektów. To istotna różnica. Klasa to coś więcej niż wartość. Może mieć wiele instancji i może być nośnikiem danych. Pomyśl o `Response`: gdyby była klasą enum, nie mogłaby przechowywać `value` ani `error`. Podklasy szczelnie zamknięte mogą przechowywać różne dane, podczas gdy enum to tylko zestaw wartości.

### Przypadki użycia

Używamy klas szczelnie zamkniętych, gdy chcemy wyrazić, że istnieje konkretna liczba podklas danej klasy.

```kotlin
sealed class MathOperation
class Plus(val left: Int, val right: Int) : MathOperation()
class Minus(val left: Int, val right: Int) : MathOperation()
class Times(val left: Int, val right: Int) : MathOperation()
class Divide(val left: Int, val right: Int) : MathOperation()

sealed interface Tree
class Leaf(val value: Any?) : Tree
class Node(val left: Tree, val right: Tree) : Tree

sealed interface Either<out L, out R>
class Left<out L>(val value: L) : Either<L, Nothing>
class Right<out R>(val value: R) : Either<Nothing, R>

sealed interface AdView
object FacebookAd : AdView
object GoogleAd : AdView
class OwnAd(val text: String, val imgUrl: String) : AdView
```

Kluczową korzyścią jest to, że wyrażenie when może łatwo pokryć wszystkie możliwe typy w hierarchii, używając is-checks. Warunek when z uszczelnionym elementem jako wartość zapewnia, że kompilator wykonuje wyczerpujące sprawdzanie typów, a nasz program może reprezentować tylko prawidłowe stany.

```kotlin
fun BinaryTree.height(): Int = when (this) {
    is Leaf -> 1
    is Node -> maxOf(this.left.height(), this.right.height())
}
```

Jednak wyrażenie, że hierarchia jest ograniczona, poprawia czytelność. W końcu, gdy używamy modyfikatora `sealed`, możemy użyć refleksji, aby znaleźć wszystkie podklasy[^14_1]:

```kotlin
sealed interface Parent
class A : Parent
class B : Parent
class C : Parent

fun main() {
    println(Parent::class.sealedSubclasses)
    // [class A, class B, class C]
}
```

### Podsumowanie

Klasy szczelnie zamknięte oraz interfejsy powinny być używane do reprezentowania ograniczonych hierarchii. Wyrażenie when ułatwia obsługę każdego możliwego szczelnie zamkniętego podtypu i, w rezultacie, dodawanie nowych metod do uszczelnionych elementów za pomocą funkcji rozszerzenia. Klasy abstrakcyjne pozostawiają miejsce dla nowych klas, które dołączą do tej hierarchii. Jeśli chcemy kontrolować, jakimi są podklasy danej klasy, powinniśmy użyć modyfikatora `sealed`.

Następnie omówimy ostatni specjalny rodzaj klasy, który służy do dodawania dodatkowych informacji o elementach naszego kodu: adnotacje.

[^14_0]: Ograniczone hierarchie są używane do reprezentowania wartości, które mogą przyjmować kilka różnych, ale stałych typów. W innych językach, ograniczone hierarchie mogą być reprezentowane przez sumę typów, koprodukty lub unie oznakowane.
[^14_1]: Wymaga to zależności `kotlin-reflect`. Więcej o refleksji w *Zaawansowane Kotlin*.
[^14_2]: Nadal można deklarować klasę abstrakcyjną lub interfejs jako część uszczelnionej hierarchii, z której klient będzie mógł dziedziczyć w innym module.