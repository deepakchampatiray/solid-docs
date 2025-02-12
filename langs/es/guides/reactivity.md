---
title: Reactividad
description: Resumen completo de la reactividad de Solid.
sort: 1
---

# Reactividad

El manejo de datos de Solid está construido con un set de reactivos primitivos flexibles que son responsables de todas las actualizaciones. Toma un acercamiento muy similar al de MobX o Vue excepto que no sacrifica su granularidad por un DOM Virtual. Las dependencias son rastreadas automáticamente cuando accedes a tus valores reactivos en los Effects y código de Vistas JSX.

Los primitivos de Solid vienen en la forma de llamadas `create` que seguido retornan tuples, donde generalmente el primer elemento es un primitivo legible y el segundo es un setter. Es común referirse únicamente a la parte legible por el nombre del primitivo.

Aquí te mostramos un contador auto-incremental básico que se actualiza basado en establecer la señal `contador`.

```jsx
import { createSignal, onCleanup } from "solid-js";
import { render } from "solid-js/web";

const App = () => {
	const [contador, setContador] = createSignal(0),
		timer = setInterval(() => setContador(contador() + 1), 1000);
	onCleanup(() => clearInterval(timer));

	return <div>{contador()}</div>;
};

render(() => <App />, document.getElementById("app"));
```

## Conoce los Primitivos

Solid está hecho de 3 primitivos primarios, Signal, Memo y Effect. En su núcleo está el patron Observador donde Signals (y Memos) son rastreados envolviendo Memos y Effects.

Signals son los primitivos mas simples. Contienen un valor y obtienen y definen funciones para que podamos interceptarlos cuando estos son utilizados.

```js
const [contador, setContador] = createSignal(0);
```

Effects son funciones que envuelven lecturas de nuestras Signals y se re-ejecutan cuando cambia un valor de la misma. esto es util para crear side effects, como renderizado.

```js
createEffect(() => console.log("The latest count is", count()));
```

Finalmente, Memos, son valores derivados en caché. Comparten las propiedades de ambos, Signals y Effects. Rastrean sus propias Signals dependientes, re-ejecutando únicamente cuando estas cambien, además son propiamente, Signals rastreables.

```js
const nombreCompleto = createMemo(() => `${nombre()} ${apellido()}`);
```

## Cómo funciona

Signals, son son emisores de eventos que albergan una lista de suscripciones. Estas notifican a sus suscriptores cuando su valor cambia.

Donde las cosas se vuelven mas interesantes es en la forma en que estas suscripciones pasan. Solid usa rastreo automático de dependencias. Las actualizaciones suceden automáticamente cuando cambia el valor.

El truco es un stack global en tiempo de ejecución. Antes de que se ejecute un Memo o un Effect (o se re-ejecute) dicha función, se empuja a si misma al stack. Entonces, cualquier señal que es leída checa si hay algún suscriptor en el stack y si es así agrega el suscriptor a sus suscripciones.

Puedes pensarlo así:

```js
function createSignal(valor) {
	const suscriptores = new Set();

	const leer = () => {
		const suscriptor = getCurrentListener();
		if (suscriptor) suscriptores.add(suscriptor);
		return valor;
	};

	const escribir = (nextValor) => {
		valor = nextValor;
		for (const sub of suscriptores) sub.run();
	};

	return [leer, escribir];
}
```

Ahora cada que actualizamos la Signal sabemos que Effects re-ejecutar. Simple y aún así, efectivo. La verdadera implementación es mucho mas complicada pero básicamente eso es lo que sucede.

Para entender mas detalladamente como la Reactividad funciona, estos son unos artículos útiles:

[A Hands-on Introduction to Fine-Grained Reactivity](https://dev.to/ryansolid/a-hands-on-introduction-to-fine-grained-reactivity-3ndf)

[Building a Reactive Library from Scratch](https://dev.to/ryansolid/building-a-reactive-library-from-scratch-1i0p)

[SolidJS: Reactivity to Rendering](https://indepth.dev/posts/1289/solidjs-reactivity-to-rendering)

## Consideraciones

Este acercamiento a la reactividad es muy poderoso y dinámico. Puede manejar dependencias cambiando al vuelo ejecutando diferentes ramas del código condicional. También funciona mediante muchos niveles de indirection. Cualquier función ejecutada dentro de un ámbito de rastreo también será rastreada.

Sin embargo, hay algunos comportamientos clave y compensaciones de las que debemos ser conscientes.

1. Toda la reactividad está siendo rastreada por llamadas de función ya sea directamente o escondidas bajo el getter/proxy y disparada por el acceso a la propiedad. Esto significa que donde accedes a las propiedades de objetos reactivos es importante.

2. Los componentes y callbacks desde el control de flujo no son ambientes de rastreo y solo se ejecutan una vez. Esto significa que desestructurar o hacer lógica de nivel superior en tus componentes no se re-ejecutará. Debes acceder a estas Signals, Stores, y props desde dentro de otros primitivos reactivos o del JSX para que dicha parte del código se re-evalúe.

3. Este acercamiento solo rastrea sincrónicamente. Si tienes un setTimeout o usas una función asíncrona en tu Effect, el código que se ejecute de forma asíncrona después del hecho no será rastreada.
