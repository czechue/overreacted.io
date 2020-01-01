---
title: Jak React odróżnia klasę od funkcji?
date: '2018-12-02'
spoiler: Porozmawiamy o klasach, operatorach new i instanceof, łańcuchu prototypów, oraz architekturze API.
---

Rozważ ten przykładowy komponent `Greeting`, który jest zdefiniowany jako funkcja:

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
// Class or function — whatever.
<Greeting />
```

Jednak *React* rozróżnia tę kwestię.

Jeśli `Greeting` jest funkcją, React musi wywołać ją:

```jsx
// Your code
function Greeting() {
  return <p>Hello</p>;
}

// Wewnątrz React
const result = Greeting(props); // <p>Hello</p>
```

Lecz gdy `Greeting` jest klasą, React musi zinstancjonować go z wykorzystaniem operatora `new` i *wówczas* wywołać metodę `render` na nowopowstałej instancji:

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

W obydwu przypadkach zadaniem Reacta jest otrzymać wyrenderowany node (w tym przykładzie, `<p>Hello</p>`). Jednak dokładne kroki zależą od tego jak `Greeting` jest zdefiniowany.

**Więc skąd React wie czy coś jest klasą czy funkcją?**

Tak jak i w moim [poprzednim poście](/why-do-we-write-super-props/), **nie *musisz* tego wiedzieć aby być produktywnym w React.** Ja nie wiedziałem tego przez lata. Proszę nie zamieniaj tego w pytanie rekrutacyjne. Prawdę mówiąc, ten post jest bardziej o JavaScripcie niż o React.

Ten blog jest przeznaczony dla ciekawskich czytelników, którzy chcą wiedzieć *dlaczego* React działa w określony sposób. Czy jesteś tą osobą? W takim razie zgłębmy temat.

**To jest długa podróż. Zapnij pasy. Ten post nie posiada wielu informacji o samym React, ale przyjrzymy się niektórym aspektom `new`, `this`, `class`, funkcji strzałkowych, `prototype`, `__proto__`, `instanceof` i jak te rzeczy współpracują ze sobą w JavaScript. Na twoje szczęście zbyt często nie musisz o nich myśleć gdy *korzystasz* z React. Jeśli jednak implementujesz React...**

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

**Składnia klas pozwala nam powiedzieć: "To nie jest tylko funkcja - to klasa i ma konstruktor".** Jeśli zapomnisz `new` podczas wywołania, JavaScript zwróci błąd:

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

Zanim zobaczymy, jak React rozwiązuje tę kwestię, ważne jest aby pamiętać, że większość osób korzystających z React używa kompilatorów (np. Babel), aby skompilować nowoczesne funkcjonalności, takie jak klasy do starszych przeglądarek. Musimy więc wziąć pod uwagę kompilatory w naszych rozważaniach.

We wczesnej wersji Babela, klasy mogły być wywoływane bez `new`. Jednak zostało to naprawione - poprzez wygenerowanie dodatkowego kodu:

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



