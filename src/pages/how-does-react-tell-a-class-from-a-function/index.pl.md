---
title: Jak React odróżnia klasę od funkcji?
date: '2018-12-02'
spoiler: Porozmawiamy o klasach, operatorach new i instanceof, łańcuchu prototypów, oraz architekturze API.
---

Rozważmy ten przykładowy komponent `Greeting`, który jest zdefiniowany jako funkcja:

```jsx
function Greeting() {
  return <p>Hello</p>;
}
```

React również umożliwia zdefiniowanie go jako klasę:

```jsx
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}
```

(Do [niedawna](https://reactjs.org/docs/hooks-intro.html), był to jedyny sposób aby móc skorzystać z takich funkcjonalności jak stan.)

Kiedy chcesz wyrenderować `<Greeting />`, nie przejmujesz się jak jest on zdefiniowany:

```jsx
// Klasa czy funkcja - bez znaczenia.
<Greeting />
```

Jednak *React* rozróżnia tę kwestię.

Jeśli `Greeting` jest funkcją, React musi ją wywołać:

```jsx
// Your code
function Greeting() {
  return <p>Hello</p>;
}

// Wewnątrz React
const result = Greeting(props); // <p>Hello</p>
```

Lecz gdy `Greeting` jest klasą, React musi zinstancjonować ją z wykorzystaniem operatora `new` i *wówczas* wywołać metodę `render` na nowopowstałej instancji:

```jsx
// Twój kod
class Greeting extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

// Wewnątrz React
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

W obydwu przypadkach zadaniem React jest otrzymać wyrenderowany node (w tym przykładzie, `<p>Hello</p>`). Jednak dokładne kroki zależą od tego jak `Greeting` jest zdefiniowany.

**Skąd więc React wie czy coś jest klasą czy funkcją?**

Tak jak i w moim [poprzednim poście](/why-do-we-write-super-props/), **nie *musisz* tego wiedzieć aby być produktywnym w React.** Ja nie znałem odpowiedzi na to pytanie przez lata. Proszę nie zamieniaj go również w pytanie rekrutacyjne. Prawdę mówiąc, ten post jest bardziej o JavaScript niż o React.

Blog jest przeznaczony dla ciekawskich czytelników, którzy chcą wiedzieć *dlaczego* React działa w określony sposób. Jeśli jesteś taką osobą, to zapraszam do wspólnego zgłębienia tematu.

**Rozpoczynamy długą podróż. Zapnij pasy. Ten post nie posiada wielu informacji o samym React, ale przyjrzymy się niektórym aspektom `new`, `this`, `class`, funkcjom strzałkowym, `prototype`, `__proto__`, `instanceof` i jak te rzeczy współgrają ze sobą w JavaScript. Na twoje szczęście nie musisz o nich zbyt często myśleć gdy *korzystasz* z React. Jeśli jednak implementujesz React...**

(Chcesz tylko znać odpowiedzi na pytania? Zescrolluj do samego końca.)

----

Po pierwsze, musimy zrozumieć dlaczego ważne jest aby inaczej traktować funkcje i klasy. Zwróć uwagę, jak używamy operatora `new` podczas wywoływania klasy:

```jsx{5}
// Jeśli Greeting jest funkcją
const result = Greeting(props); // <p>Hello</p>

// Jeśli Greeting jest klasą
const instance = new Greeting(props); // Greeting {}
const result = instance.render(); // <p>Hello</p>
```

Zobaczmy, jaką rolę pełni operator `new` w JavaScript.

---

Dawniej JavaScript nie miał klas. Można jednak wyrazić wzór podobny do klas za pomocą zwykłych funkcji. **Konkretnie, możesz użyć *dowolnej* funkcji w roli podobnej do konstruktora klasy, dodając `new` przed jej wywołaniem.**

```jsx
// Zwykła funkcja
function Person(name) {
  this.name = name;
}

var fred = new Person('Fred'); // ✅ Person {name: 'Fred'}
var george = Person('George'); // 🔴 Nie zadziała
```

Wciąż możesz dzisiaj pisać kod w ten sposób! Sprawdź w DevTools.

Jeśli `Person('Fred')` zostałby wywołany **bez** `new`, `this` wewnątrz wskazywałoby na coś globalnego i bezużytecznego (na przykład na `window` lub `undefined`). Więc nasz kod zawiesiłby się lub zrobiłby coś głupiego, jak przypisanie `window.name`.

Przez dodanie `new` przed wywołaniem, mówimy: "Hej JavaScript, wiem, że `Person` to tylko funkcja, ale udawajmy, że jest ona czymś w rodzaju konstruktora klasy. **Utwórz obiekt `{}` i przyczep `this` wewnątrz funkcji `Person` do tego obiektu, aby móc przypisać rzeczy takie jak `this.name`. Następnie zwróc mi ten obiekt.**"

To właśnie robi operator `new`.

```jsx
var fred = new Person('Fred'); // Ten sam obiekt, co `this` wewnątrz `Person`
```

Operator `new` sprawia również, że wszystko co umieszczamy w `Person.prototype` jest dostępne w obiekcie `fred`.

```jsx{4-6,9}
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name);
}

var fred = new Person('Fred');
fred.sayHi();
```

W ten sposób ludzie emulowali klasy, zanim JavaScript dodał je bezpośrednio.

---

Tak więc `new` jest w JavaScript od jakiegoś czasu. Jednakże, klasy są nieco nowsze. Pozwalają nam zapisać powyższy kod, tak aby lepiej pasował do naszych intencji:

```jsx
class Person {
  constructor(name) {
    this.name = name;
  }
  sayHi() {
    alert('Hi, I am ' + this.name);
  }
}

let fred = new Person('Fred');
fred.sayHi();
```

*Poprawne zinterpretowanie zamiarów programisty* jest ważne w projektowaniu języka i API.

Jeśli napiszesz funkcję, JavaScript nie jest w stanie odgadnąć czy ma ona być wywołana jak `alert()` lub czy ma służyć jako konstruktor, przykładowo `new Person()`.  Pominięcie `new` dla funkcji takiej jak `Person` prowadziłoby do nieprzewidzianego zachowania.

**Składnia klas pozwala nam powiedzieć: "To nie jest tylko funkcja - to klasa i ma konstruktor".** Jeśli zapomnisz więc `new` podczas wywołania, JavaScript zwróci błąd:

```jsx
let fred = new Person('Fred');
// ✅  Jeśli Person jest funkcją, wszystko ok
// ✅  Jeśli Person jest klasą, wszystko ok również

let george = Person('George'); // Zapomnieliśmy `new`
// 😳 Jeśli Person jest funkcją zbudowaną jak konstruktor: nieprzewidziane zachowanie
// 🔴 Jeśli Person jest klasą: natychmiast zwraca błąd
```

Mechanizm ten pomaga nam odpowiednio wcześnie wychwycić błędy, z których w późniejszym etapie mogłyby powstać brzydkie bugi jak `this.name` traktowany jako `window.name` zamiast `george.name`.

Oznacza to jednak, że React musi wstawić `new` przed wywołaniem dowolnej klasy. Nie może jej wywołać jak zwykłej funkcji, ponieważ JavaScript potraktuje to jako błąd!

```jsx
class Counter extends React.Component {
  render() {
    return <p>Hello</p>;
  }
}

// 🔴 React nie może zrobić tego:
const instance = Counter(props);
```

To oznacza kłopoty.

---

Zanim przyjmrzymy się, jak React rozwiązuje tę kwestię, ważne jest aby pamiętać, że większość osób korzystających z React używa kompilatorów (np. Babel), aby skompilować nowoczesne funkcjonalności, takie jak klasy do starszych przeglądarek. Musimy więc wziąć pod uwagę kompilatory w naszych rozważaniach.

We wczesnej wersji Babel, klasy mogły być wywoływane bez `new`. Jednak zostało to naprawione - poprzez wygenerowanie dodatkowego kodu:

```jsx
function Person(name) {
  // Nieco uproszczony kod wynikowy z Bebel:
  if (!(this instanceof Person)) {
    throw new TypeError("Cannot call a class as a function");
  }
  // Our code:
  this.name = name;
}

new Person('Fred'); // ✅ Okay
Person('George');   // 🔴 Nie można wywołać klasy jako funkcji
``` 

Być może taki kod pojawił się w twoim bundlu. Tak właśnie działają wszystkie te funkcje `_classCallCheck`. (Możesz zmniejszyć rozmiar bundla, wybierając „tryb luźny” bez sprawdzania, ale może to skomplikować ostateczne przejście do prawdziwych klas natywnych).

---

Do tej pory powinieneś już z grubsza rozumieć różnicę między wywoływaniem czegoś za pomocą `new` lub bez `new`:

|  | `new Person()` | `Person()` |
|---|---|---|
| `klasa` | ✅ `this` jest instancją `Person` | 🔴 `TypeError`
| `funkcja` | ✅ `this` jest instancją `Person`  | 😳 `this` to `window` albo `undefined` |

Dlatego ważne jest, aby React poprawnie wywoływał twój komponent. **Jeśli twój komponent jest zdefiniowany jako klasa, React musi używać `new` podczas wywoływania go.**

Tak więc, czy React może po prostu sprawdzić, czy coś jest klasą, czy nie?

To nie takie proste. Nawet gdybyśmy mogli [odróżnić klasę od funkcji w JavaScript](https://stackoverflow.com/questions/29093396/how-do-you-check-the-difference-between-an-ecmascript-6-class-and-function), to nadal nie działałoby to w przypadku klas przetwarzanych przez narzędzia takie jak Babel. Dla przeglądarki są to zwykłe funkcje. Pechowo dla React.

---

Okej, więc może React mógłby po prostu używać „new” przy każdym wywołaniu? Niestety, to również nie zawsze działa.

Przy zwykłych funkcjach, wywoływanie ich za pomocą `new` dałoby im instancję obiektu jako `this`. Jest to pożądane dla funkcji napisanych jako konstruktor (takich jak nasza `Person` powyżej), ale byłoby mylące dla komponentów funkcyjnych:

```jsx
function Greeting() {
  // Nie spodziewalibyśmy się, że `this` będzie tu jakąkolwiek instancją
  return <p>Hello</p>;
}
```

Mogłoby to być akceptowalne. Istnieją jednak dwa *inne* powody, które dyskwalifikują ten sposób.

---

Pierwszym powodem, dla którego użycie `new` za kzdym razem nie zadziała, jest to, że w przypadku natywnych funkcji strzałek (nie tych skompilowanych przez Babel), wywołanie z `new` powoduje błąd:

```jsx
const Greeting = () => <p>Hello</p>;
new Greeting(); // 🔴 Greeting nie jest konstruktorem
```

To zachowanie jest zamierzone i wynika ze sposobu w jaki funkcje strzałkowe są zaimplementowane. Jedną z głównych zalet funkcji strzałkowych jest to, że *nie* mają własnej wartości `this` - zamiast tego `this` wskazuje na najbliższą regularną funkcję:

```jsx{2,6,7}
class Friends extends React.Component {
  render() {
    const friends = this.props.friends;
    return friends.map(friend =>
      <Friend
        // `this` is resolved from the `render` method
        size={this.props.size}
        name={friend.name}
        key={friend.id}
      />
    );
  }
}
```

Okej, więc **funkcje strzałkowe nie mają własnego `this`.** To oznacza, że byłyby całkowicie bezużyteczne jako konstruktory!

```jsx
const Person = (name) => {
  // 🔴 To nie miałoby sensu!
  this.name = name;
}
```

Dlatego **JavaScript nie pozwala na wywołanie funkcji strzałki za pomocą `new`.** Jeśli to zrobisz, prawdopodobnie i tak popełnisz błąd więc najlepiej jest o tym wiedzieć wcześniej. Działa tu podobny mechanizm do tego, w jaki sposób JavaScript nie pozwala na wywołanie klasy *bez* `new`.

Przydatna funkcjonalność, jednak udaremnia nasz plan. React niestety nie może wywoływać „new” na wszystkim, ponieważ spowodowałoby to uszkodzenie funkcji strzałkowych! Moglibyśmy spróbować wykrywać funkcje strzałkowe korzystając z braku `prototype` a nie `new`:

```jsx
(() => {}).prototype // undefined
(function() {}).prototype // {constructor: f}
```

Ale to [nie zadziała](https://github.com/facebook/react/issues/4599#issuecomment-136562930) dla funkcji skompilowanych przez Babel. Może to się nie wydawać wielką sprawą, jednak jest to kolejny powód, który dowodzi że takie podejście jest ślepym zaułkiem.

---

Innym powodem, dla którego nie zawsze możemy korzystać z `new`, jest fakt, że uniemożliwiałoby to Reactowi obsługę komponentów zwracających stringi lub inne typy prymitywne.

```jsx
function Greeting() {
  return 'Hello';
}

Greeting(); // ✅ 'Hello'
new Greeting(); // 😳 Greeting {}
```

To, znów ma związek z osobliwymi właściwościami [operatora `new`] (https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/new). Jak widzieliśmy wcześniej, `new` mówi silnikowi JavaScript, aby ten utworzył obiekt, wskazał `this` na ten obiekt wewnątrz funkcji, a później zwrócił nam ten obiekt w wyniku `new`.

Jednakże, JavaScript pozwala również funkcji wywoływanej przez `new` aby *nadpisać* wartość zwracaną przez `new`, zwracając jakiś inny obiekt. Przypuszczalnie uznano to za przydatne w przypadku wzorców takich jak pula obiektów, w których chcemy ponownie wykorzystać instancje:

```jsx{1-2,7-8,17-18}
// Stworzony leniwie
var zeroVector = null;

function Vector(x, y) {
  if (x === 0 && y === 0) {
    if (zeroVector !== null) {
      // Wykorzystuje tę samą instancję
      return zeroVector;
    }
    zeroVector = this;
  }
  this.x = x;
  this.y = y;
}

var a = new Vector(1, 1);
var b = new Vector(0, 0);
var c = new Vector(0, 0); // 😲 b === c
```

Jednak, `new` również *całkowicie ignoruje* wartość zwracaną przez funkcję, jeśli *nie* jest to obiekt. Jeśli zwrócisz string lub liczbę, to tak, jakby w ogóle nie było `return`.

```jsx
function Answer() {
  return 42;
}

Answer(); // ✅ 42
new Answer(); // 😳 Answer {}
```

Po prostu nie ma sposobu, aby odczytać zwracaną prymitywną wartość (np. liczbę lub ciąg) z funkcji podczas wywoływania jej za pomocą `new`. Więc jeśli React zawsze korzystałby z `new`, nie byłoby możliwe dodanie komponentów pomocniczych, które zwracają stringi!

To niedopuszczalne, więc musimy iść na kompromis.

---

Czego się nauczyliśmy do tej pory? React musi wywoływać klasy (włączając kod wynikowy Babel) *z* `new` ale musi też wywoływać regularne funkcje lub funkcje strzałkowe (również skompilowane przez Babel) *bez* `new`. I nie ma niezawodnego sposobu na ich rozróżnienie.

**Jeśli nie możemy rozwiązać ogólnego problemu, czy możemy rozwiązać bardziej konkretny?**

Kiedy zdefiniujesz komponent jako klasę, prawdopodobnie będziesz chciał dziedziczyć po `React.Component` dla wbudowanych metod takich jak `this.setState()`. **Zamiast próbować wykryć wszystkie klasy, czy możemy wykryć tylko potomków `React.Component`?**

Spoiler: dokładnie tak działa React.

---

Być może idiomatycznym sposobem sprawdzenia, czy `Greeting` jest klasą komponentu React, jest przetestowanie czy `Greeting.prototype instanceof React.Component`:

```jsx
class A {}
class B extends A {}

console.log(B.prototype instanceof A); // true
```

Wiem, o czym myślisz. Co się tu właśnie stało?! Aby odpowiedzieć na to pytanie, musimy zrozumieć prototypy JavaScript.

Być może słyszałeś już o „łańcuchu prototypów”. Każdy obiekt w JavaScript może mieć „prototyp”. Kiedy piszemy `fred.sayHi()`, ale obiekt `fred` nie ma właściwości `sayHi`, szukamy właściwości `sayHi` w prototypie `fred`. Jeśli go tam nie znajdziemy, patrzymy na następny prototyp w łańcuchu - prototyp prototypu „fred”. I tak dalej.

**Mylącym może być fakt, że właściwość `prototype` klasy lub funkcji _nie_ wskazuje na prototyp tej wartości.** Nie żartuję.

```jsx
function Person() {}

console.log(Person.prototype); // 🤪 Nie jest to prototyp obiektu Person 
console.log(Person.__proto__); // 😳 Prototyp obiektu Person
```

Zatem „łańcuch prototypów” bardziej przypomina `__proto __.__ proto __.__ proto__` niż `prototype.prototype.prototype`. Zrozumienie tego zajęło mi lata.

Czym jest zatem właściwość `prototype` funkcji lub klasy? **Jest to `__proto__` nadane wszystkim obiektom, które powstały poprzez wywołanie `new` z tą klasą lub funkcją!**

```jsx{8}
function Person(name) {
  this.name = name;
}
Person.prototype.sayHi = function() {
  alert('Hi, I am ' + this.name);
}

var fred = new Person('Fred'); // Ustawia `fred.__proto__` to `Person.prototype`
```

Łańcuch `__proto__` jest sposobem, w jaki JavaScript wyszukuje właściwości:

```jsx
fred.sayHi();
// 1. Czy fred ma właściwość sayHi? Nie.
// 2. Czy fred.__proto__ ma właściwość sayHi? Tak. Wywołaj ją!

fred.toString();
// 1. Czy fred ma właściwość toString? Nie.
// 2. Czy fred.__proto__ ma właściwość toString? Nie.
// 3. Czy fred.__proto__.__proto__ ma właściwość toString? Tak. Wywołaj ją!
```

W praktyce prawie nigdy nie powinieneś dotykać bezpośrednio `__proto__` w kodzie, chyba że debugujesz coś związanego z łańcuchem prototypów. Jeśli chcesz udostępnić rzeczy w `fred .__ proto__`, powinieneś dodać je do `Person.prototype`. Przynajmniej tak to zostało pierwotnie zaprojektowane.

Początkowo właściwość `__proto__` nie miała być ujawniana w przeglądarkach, ponieważ łańcuch prototypów był uważany za koncepcję wewnętrzną. Ale niektóre przeglądarki dodały `__proto__` i ostatecznie koncepcja została niechętnie ustandaryzowana (ale deprecated na korzyść `Object.getPrototypeOf()`).

**A jednak nadal uważam za bardzo mylące, że właściwość o nazwie `prototype` nie daje prototypu wartości** (na przykład `fred.prototype` jest undefined, ponieważ `fred` nie jest funkcją). Osobiście uważam, że jest to największy powód, dla którego nawet doświadczeni programiści źle rozumieją prototypy w JavaScript.

---

