## Adnotacje

Innym specjalnym rodzajem klasy w Kotlinie są adnotacje, które używamy do dostarczania dodatkowych informacji o elementach naszego kody (takich jak klasy, funkcje, właściwości). Oto przykład klasy, której elementy używają adnotacji `JvmField`, `JvmStatic` oraz `Throws`[^15_2].

```kotlin
import java.math.BigDecimal
import java.math.MathContext

class Money(
    val amount: BigDecimal,
    val currency: String,
) {
    @Throws(IllegalArgumentException::class)
    operator fun plus(other: Money): Money {
        require(currency == other.currency)
        return Money(amount + other.amount, currency)
    }

    companion object {
        @JvmField
        val MATH = MathContext(2)

        @JvmStatic
        fun eur(amount: Double) =
            Money(amount.toBigDecimal(MATH), "EUR")

        @JvmStatic
        fun usd(amount: Double) =
            Money(amount.toBigDecimal(MATH), "USD")

        @JvmStatic
        fun pln(amount: Double) =
            Money(amount.toBigDecimal(MATH), "PLN")
    }
}
```

Możesz również definiować własne adnotacje. Oto przykład deklaracji i użycia adnotacji:

```kotlin
annotation class Factory
annotation class FactoryFunction(val name: String)

@Factory
class CarFactory {

    @FactoryFunction(name = "toyota")
    fun makeToyota(): Car = Toyota()

    @FactoryFunction(name = "skoda")
    fun makeSkoda(): Car = Skoda()
}

abstract class Car
class Toyota : Car()
class Skoda : Car()
```

Możesz się zastanawiać, co robią te adnotacje. Odpowiedź jest zaskakująco prosta: absolutnie nic. Adnotacje same w sobie nie zmieniają sposobu działania naszego kodu. Przechowują jedynie informacje. Jednak wiele bibliotek uzależnia swoje działanie od użytych adnotacji. Służą więc one do określania, jak odpowiednie biblioteki mają się zachować.

Wiele ważnych bibliotek korzysta z mechanizmu zwanego *przetwarzaniem adnotacji* (ang. *annotation processing*). Działa to prosto: istnieją klasy zwane *procesorami adnotacji*, które są uruchamiane podczas budowy naszego kodu. Analizują nasz kod i generują dodatkowy kod. Ogólnie rzecz biorąc, są one silnie związane z adnotacjami. Powstały nowy kod nie jest częścią źródeł naszego projektu, nie możemy go więc sami zmieniać, ale możemy z niego korzystać w innych częściach naszego kodu, jak również mogą z niego korzystać używane biblioteki. Tak właśnie biblioteki używają przetwarzania adnotacji. Spójrz na poniższą klasę, używającą biblioteki `Mockito` z procesorem adnotacji:

```kotlin
class DoctorServiceTest {

    @Mock
    lateinit var doctorRepository: DoctorRepository

    lateinit var doctorService: DoctorService

    @Before
    fun init() {
        MockitoAnnotations.initMocks(this)
        doctorService = DoctorService(doctorRepository)
    }

    // ...
}
```

Właściwość `doctorRepository` jest oznaczona jako `Mock`, co sprawia, że procesor biblioteki Mockito w specjalnie wygenerowanej klasie generuje kod, który ustawia wartość właściwości `doctorRepository` na nowo stworzony obiekty typu mock. Oczywiście, ta wygenerowana klasa nie będzie działać sama z siebie, ponieważ musi być uruchomiona. Właśnie po to jest `MockitoAnnotations.initMocks(this)`: używa refleksji, aby wywołać tę wygenerowaną klasę.

Przetwarzanie adnotacji jest lepiej opisane w *Zaawansowane Kotlin*, w rozdziale *Przetwarzanie adnotacji* oraz *Kotlin Symbol Processing* gdzie pokazuję jak pisać różnego rodzaju procesory adnotacji. 

### Meta-adnotacje

Adnotacje, które służą do oznaczania innych adnotacji, są znane jako meta-adnotacje. W bibliotece standardowej Kotlina są cztery kluczowe meta-adnotacje:
* `Target` wskazuje rodzaje elementów kodu, które są możliwymi celami adnotacji. Jako argumenty przyjmuje wartości wyliczenia `AnnotationTarget`, które obejmują wartości takie jak `CLASS`, `PROPERTY`, `FUNCTION`, itp.
* `Retention` określa, czy adnotacja jest przechowywana w binarnym wyniku kompilacji i jest widoczna dla refleksji. Domyślnie obie wartości są prawdziwe.
* `Repeatable` określa, że adnotacja może być stosowana dwa lub więcej razy dla pojedynczego elementu kodu.
* `MustBeDocumented` określa, że adnotacja jest częścią publicznego API i dlatego powinna być uwzględniona w wygenerowanej dokumentacji dla elementu, do którego adnotacja jest stosowana.

Oto przykłady użycia niektórych z tych adnotacji:

```kotlin
@MustBeDocumented
@Target(AnnotationTarget.CLASS)
annotation class Factory

@Repeatable
@Target(AnnotationTarget.FUNCTION)
annotation class FactoryFunction(val name: String)
```

### Adnotowanie konstruktora głównego

Aby oznaczyć konstruktor główny adnotacją, należy użyć słowa kluczowego `constructor` jako części jego definicji, przed nawiasami tego konstruktora.

```kotlin
// JvmOverloads oznacza konstruktor główny
class User @JvmOverloads constructor(
    val name: String,
    val surname: String,
    val age: Int = -1,
)
```

### Listy literalne

Gdy określamy adnotację z wartością tablicy, możemy użyć specjalnej składni zwanej "literałem tablicy". Oznacza to, że zamiast używać `arrayOf`, możemy zadeklarować tablicę za pomocą nawiasów kwadratowych.

```kotlin
annotation class AnnotationWithList(
    val elements: Array<String>
)

@AnnotationWithList(["A", "B", "C"])
val str1 = "ABC"

@AnnotationWithList(elements = ["D", "E"])
val str2 = "ABC"

@AnnotationWithList(arrayOf("F", "G"))
val str3 = "ABC"
```

Ten zapis jest dozwolony tylko dla adnotacji i aktualnie nie działa przy definiowaniu tablic w żadnym innym kontekście w naszym kodzie.

### Podsumowanie

Adnotacje służą do opisywania naszego kodu. Mogą być interpretowane przez procesory adnotacji lub przez klasy używające refleksji w czasie wykonywania. Narzędzia i biblioteki wykorzystują to do automatyzacji niektórych działań dla nas. Adnotacje same w sobie są prostą funkcjonalności, ale w połączeniu z procesorami adnotacji, dają one niesamowite możliwości[^15_3].

Przejdźmy teraz do słynnej funkcjonalności Kotlin, która daje nam możliwość rozszerzenia dowolnego typu o metody lub właściwości: rozszerzenia.

[^15_2]: Adnotacje `JvmField`, `JvmStatic` i `Throws` są opisane w książce *Zaawansowany Kotlin* i służą do dostosowywania sposobu, w jaki elementy Kotlin mogą być używane w kodzie Java.
[^15_3]: W książce *Zaawansowany Kotlin* zobaczymy, jak można zaimplementować różne procesory adnotacji i co można z nimi zrobić.
