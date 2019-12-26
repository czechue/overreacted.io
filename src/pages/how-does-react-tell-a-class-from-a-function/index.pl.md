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

Ten blog jest przeznaczony dla ciekawskich czytelników, którzy chcą wiedzieć *dlaczego* React działa w określony sposób. Czy jesteś tą osobą? W takim razie rozpocznijmy podróż.

























