# Técnicas

- [Técnicas](#t%c3%a9cnicas)
	- [Assertions en tiempo de compilación](#assertions-en-tiempo-de-compilaci%c3%b3n)
	- [Especialización de plantilla parcial](#especializaci%c3%b3n-de-plantilla-parcial)
	- [Clases locales](#clases-locales)
	- [Asignación de constantes integrales a tipos](#asignaci%c3%b3n-de-constantes-integrales-a-tipos)
	- [Mapeo de tipo a tipo](#mapeo-de-tipo-a-tipo)
	- [Selección de tipo](#selecci%c3%b3n-de-tipo)
	- [Detección de convertibilidad y herencia en tiempo de compilación](#detecci%c3%b3n-de-convertibilidad-y-herencia-en-tiempo-de-compilaci%c3%b3n)
	- [Un envoltorio alrededor type_info](#un-envoltorio-alrededor-typeinfo)
	- [NullType &amp; EmptyType](#nulltype-amp-emptytype)
	- [Características de tipo](#caracter%c3%adsticas-de-tipo)
		- [Implementación de características de puntero](#implementaci%c3%b3n-de-caracter%c3%adsticas-de-puntero)
		- [Detección de tipos fundamentales](#detecci%c3%b3n-de-tipos-fundamentales)
		- [Tipos de parámetros optimizados](#tipos-de-par%c3%a1metros-optimizados)
		- [Stripping Qualifiers](#stripping-qualifiers)
		- [Usando TypeTraits](#usando-typetraits)
		- [Terminando](#terminando)
	- [Resumen](#resumen)
	- [Table TypeTraits<T> Members](#table-typetraitst-members)

---

Este capítulo presenta una serie de técnicas de C++ que se utilizan en todo el libro. Debido a que son de ayuda en diversos contextos, las técnicas presentadas tienden a ser generales y reutilizables, por lo que puede encontrar aplicaciones para ellos en otros contextos. Algunas de las técnicas, como la especialización parcial de plantilla, son características del lenguaje. Otros, como las assertions en tiempo de compilación, vienen con algún código de soporte.

En este capítulo, se familiarizará con las siguientes técnicas y herramientas:

- Assertions en tiempo de compilación
- Especialización parcial de plantilla
- Clases locales
- Asignaciones entre tipos y valores (las plantillas de clase `Int2Type` y `Type2Type`)
- La plantilla de clase `Select`, una herramienta que elige un tipo en tiempo de compilación basado en una condición de valor booleano.
- Detección de convertibilidad y herencia en tiempo de compilación
- `TypeInfo`, un útil contenedor alrededor de `std::type_info`
- `Traits`, una colección de característica que se aplican a cualquier tipo de `C++`

Tomados de forma aislada, cada técnica y su código de soporte pueden parecer triviales; la norma es de cinco a diez líneas de código fácil de entender. Sin embargo, las técnicas tienen una propiedad importante: son "no terminales"; es decir, puedes combinarlos para generar modismos de alto nivel. Juntos, forman una base sólida de servicios que ayudan a construir estructuras arquitectónicas poderosas.

Las técnicas vienen con ejemplos, así que no esperes que la discusión sea seca. A medida que lea el resto del libro, es posible que desee volver a este capítulo como referencia.

## Assertions en tiempo de compilación

A medida que despegó la programación genérica en `C++`, surgió la necesidad de una mejor comprobación estática (y mejores mensajes de error más personalizables).

Supongamos, por ejemplo, que está desarrollando una función para la conversión segura. Desea transmitir de un tipo a otro, mientras se asegura de que se conserva la información; los tipos más grandes no se deben lanzar a tipos más pequeños.

```cpp
template <class To, class From>
To safe_reinterpret_cast(From from) {
	assert(sizeof(From) <= sizeof(To));
	return reinterpret_cast<To>(from);
}
```

Se llama a esta función con la misma sintaxis que las versiones nativas de `C++`:

```cpp
int i = ...;
char* p = safe_reinterpret_cast<char*>(i);
```

Usted especifica el argumento de la plantilla `To` explícitamente, y el compilador deduce el argumento de la plantilla `From` del tipo `i`. Al afirmar (assert) en la comparación de tamaño, se asegura de que el tipo de destino pueda contener todos los bits del tipo de origen. De esta manera, el código anterior produce una conversión supuestamente correcta o una afirmación (assert) en tiempo de ejecución.

> En la mayoría de las máquinas, es decir, con `reinterpret_cast`, nunca puedes estar seguro.

Obviamente, sería más deseable detectar dicho error durante la compilación. Por un lado, el error podría estar en una parte de su programa que rara vez se ejecuta. A medida que transfiere la aplicación a un nuevo compilador o plataforma, no puede recordar todas las partes potencialmente no portables y puede dejar el error inactivo hasta que bloquee el programa frente a su cliente.

Hay esperanza; la expresión que se evalúa es una constante de tiempo de compilación, lo que significa que puede tener el compilador, en lugar del código de tiempo de ejecución, verifíquelo. La idea es pasar al compilador una construcción de lenguaje que sea legal para una expresión distinta de cero e ilegal para una expresión que se evalúe a cero. De esta manera, si pasa una expresión que se evalúa a cero, el compilador señalará un error en tiempo de compilación.

La solución más simple para las afirmaciones (assert) en tiempo de compilación (Van Horn 1997), y una que funciona tanto en `C` como en `C++`, se basa en el hecho de que una matriz de longitud cero es ilegal.

```cpp
#define STATIC_CHECK(expr) { char unnamed[(expr) ? 1 : 0]; }
```

Ahora si escribes:

```cpp
template <class To, class From>
To safe_reinterpret_cast(From from) {
	STATIC_CHECK(sizeof(From) <= sizeof(To));
	return reinterpret_cast<To>(from);
}
...
void* somePointer = ...;
char c = safe_reinterpret_cast<char>(somePointer);
```

y si en el sistema los punteros son más grandes que los caracteres, el compilador se queja de que está intentando crear una matriz de longitud cero.

El problema con este enfoque es que el mensaje de error que recibe no es terriblemente informativo. "No se puede crear una matriz de tamaño cero" no sugiere "Tipo `char` es demasiado pequeño para contener un puntero". Es muy difícil proporcionar mensajes de error personalizados de forma portátil. Los mensajes de error no tienen reglas que deben obedecer; todo depende del compilador. Por ejemplo, si el error se refiere a una variable indefinida, el nombre de esa variable no aparece necesariamente en el mensaje de error.

Una mejor solución es confiar en una plantilla con un nombre informativo; con suerte, el compilador mencionará el nombre de esa plantilla en el mensaje de error.

```cpp
template<bool> struct CompileTimeError;
template<> struct CompileTimeError<true> {};
#define STATIC_CHECK(expr) (CompileTimeError<(expr) != 0>())
```

`CompileTimeError` es una plantilla que toma un parámetro sin tipo (una constante booleana). `Compile-TimeError` se define solo para el valor verdadero de la constante booleana. Si intenta crear una instancia de `CompileTimeError<false>`, el compilador emite un mensaje como "Especialización indefinida `CompileTimeError<false>`". Este mensaje es una pista ligeramente mejor de que el error es intencional y no un error del compilador o del programa.

Por supuesto, hay mucho margen de mejora. ¿Qué hay de personalizar ese mensaje de error? Una idea es pasar un parámetro adicional a `STATIC_CHECK` y de alguna manera hacer que ese parámetro aparezca en el mensaje de error. La desventaja que queda es que el mensaje de error personalizado que pasa debe ser un identificador legal de `C++` (sin espacios, no puede comenzar con un dígito, etc.). Esta línea de pensamiento conduce a un `CompileTimeError` mejorado, como se muestra en el siguiente código. En realidad, el nombre `CompileTimeError` ya no es sugerente en el nuevo contexto; como verá en un minuto, `CompileTimeChecker` tiene más sentido.

```cpp
template<bool> struct CompileTimeChecker {
	CompileTimeChecker(...);
};
template<> struct CompileTimeChecker<false> { };
#define STATIC_CHECK(expr, msg) {
	class ERROR_##msg {};
	(void)sizeof(CompileTimeChecker<(expr) != 0>((ERROR_##msg())));
}
```

Suponga que `sizeof(char) < sizeof(void *)`. (El estándar no garantiza que esto sea
necesariamente cierto.) Veamos qué sucede cuando escribe lo siguiente:

```cpp
template <class To, class From>
To safe_reinterpret_cast(From from) {
	STATIC_CHECK(sizeof(From) <= sizeof(To), Destination_Type_Too_Narrow);
	return reinterpret_cast<To>(from);
}
...
void* somePointer = ...;
char c = safe_reinterpret_cast<char>(somePointer);
```

Después del preprocesamiento de macros, el código de `safe_reinterpret_cast` se expande a lo siguiente:

```cpp
template <class To, class From>
To safe_reinterpret_cast(From from) {
	{
		class ERROR_Destination_Type_Too_Narrow {};
		(void)sizeof(CompileTimeChecker<(sizeof(From) <= sizeof(To))>(ERROR_Destination_Type_Too_Narrow()));
	}
	return reinterpret_cast<To>(from);
}
```

El código define una clase local llamada `ERROR_Destination_Type_Too_Narrow` que tiene un cuerpo vacío. Luego, crea un valor temporal de tipo `CompileTimeChecker<(sizeof(From) <= sizeof(To))>`, inicializado con un valor temporal de tipo `ERROR_estination_Type_Too_Narrow`. Finalmente, `sizeof` mide el tamaño de la variable temporal resultante.

Ahora aquí está el truco. La especialización `CompileTimeChecker<true>` tiene un constructor que acepta cualquier cosa; Es una función de puntos suspensivos. Esto significa que si la expresión de tiempo de compilación marcada se evalúa como verdadera, el programa resultante es válido. Si la comparación entre tamaños se evalúa como falsa, se produce un error en tiempo de compilación: el compilador no puede encontrar una conversión de un `ERROR_Destination_Type_Too_Narrow` a un `CompileTimeChecker<false>`. Y lo mejor de todo es que un compilador decente genera un mensaje de error como "Error: No se puede convertir `ERROR_Destination_Type_Too_Narrow` a `CompileTimeChecker<false>`".

## Especialización de plantilla parcial

La especialización de plantilla parcial le permite especializar una plantilla de clase para subconjuntos del posible conjunto de instancias de esa plantilla.

Primero recapitulemos la especialización explícita total de la plantilla. Si tiene una clase de plantilla `widget`,

```cpp
template <class Window, class Controller>
class Widget {
	... generic implementation ...
};
```

entonces puede especializar explícitamente la plantilla de clase `Widget` como se muestra:

```cpp
template <>
class Widget<ModalDialog, MyController> {
	... specialized implementation ...
};
```

`ModalDialog` y `MyController` son clases definidas por su aplicación.

Después de ver la definición de la especialización de `Widget`, el compilador usa la implementación especializada siempre que defina un objeto de tipo `Widget<ModalDialog, MyController>` y usa la implementación genérica si usa cualquier otra instancia de `Widget`.

A veces, sin embargo, es posible que desee especializar `Widget` para cualquier `Window` y `MyController`. Aquí es donde entra en juego la especialización parcial de la plantilla.

```cpp
// Partial specialization of Widget
template <class Window>
class Widget<Window, MyController> {
	... partially specialized implementation ...
};
```

Por lo general, en una especialización parcial de una plantilla de clase, usted especifica solo algunos de los argumentos de la plantilla y deja los otros genéricos. Cuando crea una instancia de la plantilla de clase en su programa, el compilador intenta encontrar la mejor coincidencia. El algoritmo de coincidencia es muy complejo y preciso, lo que le permite especializarse parcialmente en formas innovadoras. Por ejemplo, suponga que tiene una clase de plantilla `Button` que acepta un parámetro de plantilla. Luego, incluso si especializó `Widget` para cualquier `Window` y un `MyController` específico, puede especializar aún más los `Widgets` para todas las instancias de `Button` y `MyController`:

```cpp
template <class ButtonArg>
class Widget<Button<ButtonArg>, MyController> {
	... further specialized implementation ...
};
```

Como puede ver, las capacidades de especialización parcial de plantilla son bastante sorprendentes. Cuando crea una instancia de una plantilla, el compilador hace una coincidencia de patrones de especializaciones parciales y totales existentes para encontrar el mejor candidato; Esto te da una enorme flexibilidad.

Desafortunadamente, la especialización parcial de plantilla no se aplica a las funciones, ya sean miembros o no miembros, lo que reduce un poco la flexibilidad y la granularidad de lo que puede hacer.

- Aunque puede *especializar totalmente* las funciones miembro de una plantilla de clase, no puede *especializar parcialmente* las funciones miembro.
- No puede especializar parcialmente las funciones de plantilla de nivel de espacio de nombres (no miembro). Lo más parecido a la especialización parcial para funciones de plantilla de nivel de espacio de nombres es la sobrecarga. Para fines prácticos, esto significa que tiene habilidades de especialización de grano fino solo para los parámetros de la función, no para el valor de retorno o para los tipos utilizados internamente. Por ejemplo,

	```cpp
	template <class T, class U> T Fun(U obj); // primary template
	template <class U> void Fun<void, U>(U obj); // illegal partial
	// specialization
	template <class T> T Fun (Window obj); // legal (overloading)
	```

Esta falta de granularidad en la especialización parcial ciertamente facilita la vida de los escritores de compiladores, pero tiene efectos negativos para los desarrolladores. Algunas de las herramientas presentadas más adelante (como `Int2Type` y `Type2Type`) abordan específicamente esta limitación de la especialización parcial.

Este libro utiliza una especialización parcial de la plantilla copiosamente. Prácticamente toda la instalación de la lista de tipos se construye utilizando esta función.

## Clases locales

Las clases locales son una característica interesante y poco conocida de `C++`. Puede definir clases dentro de las funciones, de la siguiente manera:

```cpp
void Fun() {
	class Local {
	... member variables ...
	... member function definitions ...
	};
	... code using Local ...
}
```

Existen algunas limitaciones: las clases locales no pueden definir variables miembro estáticas y no pueden acceder a variables locales no estáticas. Lo que hace que las clases locales sean realmente interesantes es que puedes usarlas en funciones de plantilla. Las clases locales definidas dentro de las funciones de plantilla pueden usar los parámetros de plantilla de la función de cierre.

La función de plantilla `MakeAdapter` en el siguiente código adapta una interfaz a otra. `MakeAdapter` implementa una interfaz sobre la marcha con la ayuda de una clase local. La clase local almacena miembros de tipos genéricos.

```cpp
class Interface {
public:
	virtual void Fun() = 0;
	...
};

template <class T, class P>
Interface* MakeAdapter(const T& obj, const P& arg) {
	class Local : public Interface {
	public:
		Local(const T& obj, const P& arg) : obj_(obj), arg_(arg) {}
		virtual void Fun() {
			obj_.Call(arg_);
		}
	private:
		T obj_;
		P arg_;
	};
	return new Local(obj, arg);
}
```

Se puede demostrar fácilmente que cualquier idioma que use una clase local puede implementarse usando una clase de plantilla fuera de la función. En otras palabras, las clases locales no son una característica habilitadora de modismos. Por otro lado, las clases locales pueden simplificar las implementaciones y mejorar la localidad de los símbolos.

Sin embargo, las clases locales tienen una característica única: son *`final`*. Los usuarios externos no pueden derivar de una clase oculta en una función. Sin clases locales, tendría que agregar un espacio de nombres sin nombre en una unidad de traducción separada.

## Asignación de constantes integrales a tipos

Una plantilla simple, descrita inicialmente en Alexandrescu (2000b), puede ser muy útil para muchos modismos genéricos de programación. Aquí está una:

```cpp
template <int v>
struct Int2Type {
	enum { value = v };
};
```

`Int2Type` genera un tipo distinto para cada valor integral constante distinto pasado. Esto se debe a que las diferentes instancias de plantillas son tipos distintos; por lo tanto, `Int2Type<0>` es diferente de `Int2Type<1>`, y así sucesivamente. Además, el valor que genera el tipo se "guarda" en el valor del miembro `enum`.

Puede usar `Int2Type` siempre que necesite "tipificar" una constante integral rápidamente. De esta manera, puede seleccionar diferentes funciones, dependiendo del resultado de un cálculo en tiempo de compilación. Efectivamente, logra el despacho estático en un valor integral constante.

Por lo general, utiliza `Int2Type` cuando se cumplen estas dos condiciones:

- Debe llamar a una de varias funciones diferentes, dependiendo de una constante de tiempo de compilación.
- Debe hacer este envío en tiempo de compilación.

Para despachar en tiempo de ejecución, puede usar declaraciones simples `if-else` o la declaración `switch`. El costo de tiempo de ejecución es insignificante en la mayoría de los casos. Sin embargo, a menudo no puedes hacer esto. La instrucción `if-else` requiere que ambas ramas se compilen correctamente, incluso cuando la condición probada por `if` se conoce en el momento de la compilación. ¿Confuso? Sigue leyendo.

Considere esta situación: diseña un contenedor genérico `NiftyContainer`, que está diseñado según el tipo contenido:

```cpp
template <class T> class NiftyContainer {
	...
};
```

Digamos que `NiftyContainer` contiene punteros a objetos de tipo `T`. Para duplicar un objeto contenido en `NiftyContainer`, debe llamar a su constructor de copia (para tipos no polimórficos) o una función virtual `Clone()` (para tipos polimórficos). Obtiene esta información del usuario en forma de un parámetro de plantilla booleana.

```cpp
template <typename T, bool isPolymorphic>
class NiftyContainer {
	...
	void DoSomething() {
		T* pSomeObj = ...;
		if (isPolymorphic) {
			T* pNewObj = pSomeObj->Clone();
			... polymorphic algorithm ...
		} else {
			T* pNewObj = new T(*pSomeObj);
			... nonpolymorphic algorithm ...
		}
	}
};
```

El problema es que el compilador no le permitirá salirse con la suya con este código. Por ejemplo, debido a que el algoritmo polimórfico usa `pObj->Clone()`, `NiftyContainer::DoSomething` no compila ningún tipo que no defina una función miembro `Clone()`. Es cierto que es obvio en el momento de la compilación qué rama de la instrucción `if` se ejecuta. Sin embargo, eso no le importa al compilador: trata diligentemente de compilar ambas ramas, incluso si el optimizador luego eliminará el código muerto. Si intenta llamar a `DoSomething` para `NiftyContainer<int, false>`, el compilador se detiene en la llamada `pObj->Clone()` y dice: "¿Huh?"

También es posible que la rama no polimórfica del código no se pueda compilar. Si `T` es un tipo polimórfico y la rama de código no polimórfico intenta un nuevo `T(* pObj)`, el código puede fallar al compilarse. Esto podría suceder si `T` ha deshabilitado su constructor de copia (haciéndolo privado), como debería hacerlo una clase polimórfica de buen comportamiento.

Sería bueno si el compilador no se molestara en compilar el código que está muerto de todos modos, pero ese no es el caso. Entonces, ¿cuál sería una solución satisfactoria?

Como resultado, hay una serie de soluciones, y `Int2Type` ofrece una particularmente limpia. Puede transformar ad hoc el valor booleano `isPolymorphic` en dos tipos distintos correspondientes a los valores `true` y `false` de `isPolymorphic`. Entonces puede usar `Int2Type<isPolymorphic>` con sobrecarga simple, ¡y listo!

```cpp
template <typename T, bool isPolymorphic>
class NiftyContainer {
private:
	void DoSomething(T* pObj, Int2Type<true>) {
		T* pNewObj = pObj->Clone();
		... polymorphic algorithm ...
	}
	void DoSomething(T* pObj, Int2Type<false>) {
		T* pNewObj = new T(*pObj);
		... nonpolymorphic algorithm ...
	}
public:
	void DoSomething(T* pObj) {
		DoSomething(pObj, Int2Type<isPolymorphic>());
	}
};
```

`Int2Type` es muy útil como un medio para traducir un valor en un tipo. Luego pasa una variable temporal de ese tipo a una función sobrecargada. Las sobrecargas implementan los dos algoritmos necesarios.

El truco funciona porque el compilador no compila funciones de plantilla que nunca se usan, solo verifica su sintaxis. Y, por supuesto, generalmente realiza el envío en tiempo de compilación en código de plantilla.

Verá `Int2Type` en funcionamiento en varios lugares de Loki, especialmente en el Capítulo 11, Multimetodos. Allí, la clase de plantilla es un motor de doble despacho, y el parámetro de plantilla `bool` brinda la opción de admitir el despacho simétrico o no.

## Mapeo de tipo a tipo

Como se menciona en la Sección [Especialización de plantilla parcial](#especializaci%c3%b3n-de-plantilla-parcial), no existe una especialización parcial para funciones de plantilla. A veces, sin embargo, es posible que deba simular una funcionalidad similar. Considere la siguiente función.

```cpp
template <class T, class U>
T* Create(const U& arg) {
	return new T(arg);
}
```

`Create` crea un nuevo objeto, pasando un argumento a su constructor.

Ahora supongamos que hay una regla en su aplicación: los objetos de tipo `Widget` son códigos heredados intocables y deben tomar dos argumentos en la construcción, el segundo es un valor fijo como `-1`. Sus propias clases, derivadas de `Widget`, no tienen este problema.

¿Cómo puede especializarse `Create` para que trate a `Widget` de manera diferente a todos los demás tipos? Una solución obvia es crear una función `CreateWidget` separada que haga frente al caso particular. Desafortunadamente, ahora no tiene una interfaz uniforme para crear `Widgets` y objetos derivados de `Widget`. Esto hace que `Create` inutilizable en cualquier código genérico.

No se puede especializar parcialmente una función; es decir, no puedes escribir algo como esto:

```cpp
// Illegal code — don't try this at home
template <class U>
Widget* Create<Widget, U>(const U& arg) {
	return new Widget(arg, -1);
}
```

En ausencia de una especialización parcial de las funciones, la única herramienta disponible es, de nuevo, la sobrecarga. Una solución sería pasar un objeto ficticio de tipo `T` y confiar en la sobrecarga:

```cpp
template <class T, class U>
T* Create(const U& arg, T /* dummy */) {
	return new T(arg);
}

template <class U>
Widget* Create(const U& arg, Widget /* dummy */) {
	return new Widget(arg, -1);
}
```

Tal solución incurriría en la sobrecarga de construir un objeto complejo arbitrariamente que no se utiliza. Necesitamos un vehículo ligero para transportar la información de tipo sobre `T` para `Create`. Esta es la función de `Type2Type`: es el representante de un tipo, un identificador ligero que puede pasar a funciones sobrecargadas.

La definición de `Type2Type` es la siguiente:

```cpp
template <typename T>
struct Type2Type {
	typedef T OriginalType;
};
```

`Type2Type` carece de cualquier valor, pero distintos tipos conducen a instancias de `Type2Type` distintas, que es lo que necesitamos.

Ahora puedes escribir lo siguiente:

```cpp
// An implementation of Create relying on overloading
// and Type2Type
template <class T, class U>
T* Create(const U& arg, Type2Type<T>) {
	return new T(arg);
}

template <class U>
Widget* Create(const U& arg, Type2Type<Widget>) {
	return new Widget(arg, -1);
}

// Use Create()
String* pStr = Create("Hello", Type2Type<String>());
Widget* pW = Create(100, Type2Type<Widget>());
```

El segundo parámetro de `Create` sirve solo para seleccionar la sobrecarga adecuada. Ahora puede especializarse en `Create` para varias instancias de `Type2Type`, que asigna a varios tipos en su aplicación.

## Selección de tipo

A veces, el código genérico necesita seleccionar un tipo u otro, dependiendo de una constante booleana.

En el ejemplo de `NiftyContainer` discutido en la Sección [Asignación de constantes integrales a tipos](#asignaci%c3%b3n-de-constantes-integrales-a-tipos), es posible que desee utilizar un `std::vector` como su almacenamiento de fondo. Obviamente, no puede almacenar tipos polimórficos por valor, por lo que debe almacenar punteros. Por otro lado, es posible que desee almacenar tipos no polimórficos por valor, porque esto es más eficiente.

En tu plantilla de clase,

```cpp
template <typename T, bool isPolymorphic>
class NiftyContainer {
	...
};
```

debe almacenar un `vector<T *>` (si `isPolymorphic` es `true`) o un `vector<T>` (si `isPolymorphic` es `false`). En esencia, necesita un typedef `ValueType` que sea `T *` o `T`, dependiendo del valor de `isPolymorphic`.

Puede usar una plantilla de clase de características (Alexandrescu 2000a), de la siguiente manera.

```cpp
template <typename T, bool isPolymorphic>
struct NiftyContainerValueTraits {
	typedef T* ValueType;
};

template <typename T>
struct NiftyContainerValueTraits<T, false> {
	typedef T ValueType;
};

template <typename T, bool isPolymorphic>
class NiftyContainer {
	...
	typedef NiftyContainerValueTraits<T, isPolymorphic>
		Traits;
	typedef typename Traits::ValueType ValueType;
};
```

Esta forma de hacer las cosas es innecesariamente torpe. Además, no escala: para cada selección de tipo, debe definir una nueva plantilla de clase de características.

La plantilla de clase de biblioteca `Select` proporcionada por Loki hace que la selección de tipo esté disponible en el acto. Su definición utiliza la especialización parcial de plantilla.

```cpp
template <bool flag, typename T, typename U>
struct Select {
	typedef T Result;
};

template <typename T, typename U>
struct Select<false, T, U> {
	typedef U Result;
};
```

Así es como funciona. Si `flag` se evalúa como `true`, el compilador utiliza la primera definición (genérica) y, por lo tanto, el `Result` se evalúa como `T`. Si `flag` es `false`, la especialización entra en acción y, por lo tanto, el `Result` se evalúa como `U`.

Ahora puede definir `NiftyContainer::ValueType` mucho más fácilmente.

```cpp
template <typename T, bool isPolymorphic>
class NiftyContainer {
	...
	typedef typename Select<isPolymorphic, T*, T>::Result
	ValueType;
	...
};
```

## Detección de convertibilidad y herencia en tiempo de compilación

Cuando implementa funciones y clases de plantilla, surge una pregunta de vez en cuando: dados dos tipos arbitrarios `T` y `U` de los que no sabe nada, ¿cómo puede detectar si `U` hereda o no de `T`? Descubrir tales relaciones en tiempo de compilación es clave para implementar optimizaciones avanzadas en bibliotecas genéricas. En una función genérica, puede confiar en un algoritmo optimizado si una clase implementa una determinada interfaz. Descubrir esto en tiempo de compilación significa no tener que usar `dynamic_cast`, que es costoso en tiempo de ejecución.

La detección de la herencia se basa en un mecanismo más general, el de detectar la convertibilidad. El problema más general es, ¿cómo puede detectar si un tipo arbitrario `T` admite la conversión automática a un tipo arbitrario `U`?

Hay una solución a este problema, y se basa en `sizeof`. Hay una sorprendente cantidad de poder en `sizeof`: puede aplicar `sizeof` a cualquier expresión, sin importar cuán complejo sea, y `sizeof` devuelve su tamaño sin evaluar realmente esa expresión en tiempo de ejecución. Esto significa que `sizeof` es consciente de sobrecarga, creación de instancias de plantilla, reglas de conversión, todo lo que puede participar en una expresión de `C++`. De hecho, `sizeof` oculta una facilidad completa para deducir el tipo de expresión; eventualmente, `sizeof` desecha la expresión y devuelve solo el tamaño de su resultado.


> Hay una propuesta para agregar un operador `typeof` a `C++`, es decir, un operador que devuelve el tipo de una expresión. Tener `typeof` facilitaría mucho la escritura y la comprensión del código de la plantilla. `Gnu C++` ya implementa `typeof` como una extensión. Obviamente, `typeof` y `sizeof` comparten el mismo back-end, porque `sizeof` tiene que descubrir el tipo de todos modos.

La idea de la detección de conversión se basa en el uso de `sizeof` junto con funciones sobrecargadas. Proporcionamos dos sobrecargas de una función: una acepta el tipo de conversión a (`U`) y la otra acepta casi cualquier otra cosa. Llamamos a la función sobrecargada con un temporal de tipo `T`, el tipo cuya convertibilidad a `U` queremos determinar. Si se llama a la función que acepta una `U`, sabemos que `T` es convertible a `U`; si se llama a la función de reserva, entonces `T` no es convertible a `U`. Para detectar qué función se llama, organizamos las dos sobrecargas para devolver tipos de diferentes tamaños, y luego discriminamos con `sizeof`. Los tipos en sí mismos no importan, siempre que tengan diferentes tamaños.

Primero creemos dos tipos de diferentes tamaños. (Aparentemente, `char` y `long double` tienen diferentes tamaños, pero eso no está garantizado por el estándar). Un esquema infalible sería el siguiente:

```cpp
typedef char Small;
class Big { char dummy[2]; };
```

Por definición, `sizeof(Small)` es `1`. El tamaño de `Big` es desconocido, pero ciertamente es mayor que 1, que es la única garantía que necesitamos.

A continuación, necesitamos las dos sobrecargas. Uno acepta una `U` y devuelve, digamos, un `Small`:

```cpp
Small Test(U);
```

¿Cómo podemos escribir una función que acepte "cualquier otra cosa"? Una plantilla no es una solución porque la plantilla siempre calificaría como la mejor coincidencia, ocultando así la conversión. Necesitamos una coincidencia que sea "peor" que una conversión automática, es decir, una conversión que se activa si y solo si no hay conversión automática. Un vistazo rápido a las reglas de conversión aplicadas para una llamada a función produce la coincidencia de puntos suspensivos, que es lo peor de todo: el final de la lista. Eso es exactamente lo que recetó el médico.

```cpp
Big Test(...);
```

> Pasar un objeto `C++` a una función con puntos suspensivos tiene resultados indefinidos, pero esto no importa. En realidad, nada llama a la función. Ni siquiera está implementado. Recuerde que `sizeof` no evalúa su argumento.

Ahora necesitamos aplicar `sizeof` a la llamada de `Test`, pasándole un `T`:

```cpp
const bool convExists = sizeof(Test(T())) == sizeof(Small);
```

¡Eso es! La llamada de `Test` obtiene un objeto construido por defecto `T()` y luego `sizeof` extrae el tamaño del resultado de esa expresión. Puede ser `sizeof(Small)` o `sizeof(Big)`, dependiendo de si el compilador encontró o no una conversión.

Hay un pequeño problema. Si `T` hace que su constructor predeterminado sea privado, la expresión `T()` no se compila y también lo hace todo nuestro andamiaje. Afortunadamente, hay una solución simple: simplemente use una función de hombre de paja que devuelva un `T`. (Recuerde, estamos en el tamaño del país de las maravillas donde no se evalúa realmente ninguna expresión). En este caso, el compilador está contento y nosotros también.

```cpp
T MakeT(); // not implemented
const bool convExists = sizeof(Test(MakeT())) == sizeof(Small);
```

> Por cierto, ¿no es ingenioso cuánto puedes hacer con funciones, como `MakeT` y `Test`, que no solo no hacen nada sino que ni siquiera existen en absoluto?

Ahora que lo tenemos funcionando, empaquetemos todo en una plantilla de clase que oculte todos los detalles de la deducción de tipos y exponga solo el resultado.

```cpp
template <class T, class U>
class Conversion {
	typedef char Small;
	class Big { char dummy[2]; };
	static Small Test(U);+
	static Big Test(...);
	static T MakeT();
public:
	enum { exists =
	sizeof(Test(MakeT())) == sizeof(Small) };
};
```

Ahora puede probar la plantilla de clase de conversión escribiendo:

```cpp
int main() {
	using namespace std;
	cout
		<< Conversion<double, int>::exists << ' '
		<< Conversion<char, char*>::exists << ' '
		<< Conversion<size_t, vector<int> >::exists << ' ';
}
```

Este pequeño programa imprime "1 0 0". Tenga en cuenta que aunque `std::vector` implementa un constructor que toma un `size_t`, la prueba de conversión devuelve 0 porque ese constructor es explícito.

Podemos implementar una constante más dentro de `Conversion::sameType`, que es cierto si `T` y `U` representan el mismo tipo:

```cpp
template <class T, class U>
class Conversion {
	... as above ...
	enum { sameType = false };
};
```

Implementamos `sameType` a través de una especialización parcial de `Conversion`:

```cpp
template <class T>
class Conversion<T, T> {
public:
	enum { exists = 1, sameType = 1 };
};
```

Finalmente, estamos de vuelta en casa. Con la ayuda de `Conversion`, ahora es muy fácil determinar la herencia:

```cpp
#define SUPERSUBCLASS(T, U) \
(Conversion<const U*, const T*>::exists && \
!Conversion<const T*, const void*>::sameType)
```

`SUPERSUBCLASS(T, U)` se evalúa como `true` si `U` hereda de `T` públicamente, o si `T` y `U` son realmente del mismo tipo. `SUPERSUBCLASS` hace su trabajo evaluando la convertibilidad de una `const U *` a una `const T *`. Solo hay tres casos en los que `const U *` se convierte implícitamente en `const T *`:

1. `T` es del mismo tipo que `U`.
1. `T` es una base pública inequívoca de `U`.
1. `T` es `void`.

El último caso es eliminado por la segunda prueba. En la práctica, es útil aceptar el primer caso (`T` es lo mismo que `U`) como un caso degenerado de "is-a" porque, a efectos prácticos, a menudo se puede considerar que una clase es su propia superclase. Si necesita una prueba más estricta, puede escribirla de esta manera:

```cpp
#define SUPERSUBCLASS_STRICT(T, U) \
	(SUPERSUBCLASS(T, U) && \
	!Conversion<const T, const U>::sameType)
```

¿Por qué el código agrega todos esos modificadores `const`? La razón es que no queremos que la prueba de conversión falle debido a problemas `const`. Si el código de plantilla aplica `const` dos veces (a un tipo que ya es `const`), la segunda `const` se ignora. En pocas palabras, al usar `const` en `SUPERSUBCLASS`, siempre estamos seguros.

¿Por qué usar `SUPERSUBCLASS` y no `BASE_OF` o `INHERITS`? Por una razón muy práctica. Inicialmente, Loki usó `INHERITS`. Pero con los `INHERITS(T, U)` fue una lucha constante decir de qué manera funcionaba la prueba: ¿sabía si `T` heredó `U` o viceversa? Podría decirse que `SUPERSUBCLASS(T, U)` aclara cuál es el primero y cuál es el segundo.

## Un envoltorio alrededor `type_info`

El Standard `C++` proporciona la clase `std::type_info`, que le brinda la capacidad de investigar tipos de objetos en tiempo de ejecución. Por lo general, usa `type_info` junto con el operador `typeid`. El operador `typeid` devuelve una referencia a un objeto `type_info`:

```cpp
void Fun(Base* pObj) {
	// Compare the two type_info objects corresponding
	// to the type of *pObj and Derived
	if (typeid(*pObj) == typeid(Derived)) {
	... aha, pObj actually points to a Derived object ...
	}
	...
}
```

El operador `typeid` devuelve una referencia a un objeto de tipo `type_info`. Además de admitir los operadores de comparación `operator==` y `operator!=`, `type_info` proporciona dos funciones más:

- La función miembro `name` devuelve una representación textual de un tipo, en forma de `const char *`. No hay una forma estandarizada de asignar nombres de clases a cadenas, por lo que no debe esperar que `typeid(Widget)` devuelva "`Widget`". Una implementación conforme (pero no necesariamente premiada) puede hacer que `type_info::name` devuelva la cadena vacía para todos los tipos.
- La función miembro `before` introduce una relación de orden para los objetos `type_info`. Usando `type_info::before`, puede realizar indexación en objetos `type_info`.

Desafortunadamente, las capacidades útiles de `type_info` están empaquetadas de una manera que las hace innecesariamente difíciles de explotar. La clase `type_info` deshabilita el constructor de copia y el operador de asignación, lo que hace imposible almacenar objetos `type_info`. Sin embargo, puede almacenar punteros para escribir objetos `type_info`. Los objetos devueltos por `typeid` tienen almacenamiento estático, por lo que no tiene que preocuparse por problemas de por vida. Sin embargo, debe preocuparse por la identidad del puntero.

El estándar no garantiza que cada invocación de, digamos, `typeid(int)` devuelva una referencia al mismo objeto `type_info`. En consecuencia, no puede comparar punteros con objetos `type_info`. Lo que debe hacer es almacenar punteros en objetos `type_info` y compararlos aplicando `type_info::operator==` a los punteros desreferenciados.

Si desea ordenar objetos `type_info`, nuevamente debe almacenar los punteros a `type_info`, y esta vez debe usar la función de miembro `before`. En consecuencia, si desea utilizar los contenedores ordenados de `STL` con `type_info`, debe escribir un pequeño `functor` y tratar con punteros.

Todo esto es lo suficientemente torpe como para ordenar una clase de contenedor alrededor de `type_info` que almacena un puntero a un objeto `type_info` y proporciona:

- Todas las funciones miembro de `type_info`
- Semántica de valor (constructor de copia pública y operador de asignación)
- Comparaciones perfectas definiendo el `operator<` y el `operator==`

Loki define la clase de contenedor `TypeInfo` que implementa un contenedor tan práctico alrededor de `type_info`. La sinopsis de `TypeInfo` sigue.

```cpp
class TypeInfo {
public:
	// Constructors/destructors
	TypeInfo(); // needed for containers
	TypeInfo(const std::type_info&);
	TypeInfo(const TypeInfo&);
	TypeInfo& operator=(const TypeInfo&);
	// Compatibility functions
	bool before(const TypeInfo&) const;
	const char* name() const;
private:
	const std::type_info* pInfo_;
};
// Comparison operators
bool operator==(const TypeInfo&, const TypeInfo&);
bool operator!=(const TypeInfo&, const TypeInfo&);
bool operator<(const TypeInfo&, const TypeInfo&);
bool operator<=(const TypeInfo&, const TypeInfo&);
bool operator>(const TypeInfo&, const TypeInfo&);
bool operator>=(const TypeInfo&, const TypeInfo&);
```

Debido al constructor de conversión que acepta `std::type_info` como parámetro, puede comparar directamente objetos de tipo `TypeInfo` y `std::type_info`, como se muestra:

```cpp
void Fun(Base* pObj) {
	TypeInfo info = typeid(Derived);
	...
	if (typeid(*pObj) == info) {
		... pBase actually points to a Derived object ...
	}
	...
}
```

La capacidad de copiar y comparar objetos `TypeInfo` es importante en muchas situaciones. La fábrica de clonación en el Capítulo 8 y un motor de doble despacho en el Capítulo 11 le dieron buen uso a `TypeInfo`.

## NullType & EmptyType

Loki define dos tipos muy simples: `NullType` y `EmptyType`. Puede usarlos en cálculos de tipo para marcar ciertos casos de borde.

`NullType` es una clase que sirve como marcador null para los tipos:

```cpp
class NullType {};
```

Por lo general, no crea objetos de tipo `NullType`; su único uso es indicar "No soy un tipo interesante". La Sección 2.10 usa `NullType` para los casos en que un tipo debe estar allí sintácticamente pero no tiene un sentido semántico. (Por ejemplo: "¿A qué tipo apunta un `int`?") Además, la función de lista de tipos en el Capítulo 3 usa `NullType` para marcar el final de una lista de tipos y devolver información de "tipo no encontrado".

El segundo tipo de ayuda es `EmptyType`. Como era de esperar, la definición de `EmptyType` es:

```cpp
struct EmptyType {};
```

`EmptyType` es un tipo legal para heredar, y puede pasar valores de tipo `EmptyType`. Puede usar este tipo insípido como tipo predeterminado ("no me importa") para una plantilla. La función de lista de tipos en el Capítulo 3 usa `EmptyType` de tal manera.

## Características de tipo

Los traits son una técnica de programación genérica que permite tomar decisiones en tiempo de compilación en función de los tipos, de la misma manera que tomaría decisiones en tiempo de ejecución basadas en valores (Alexandrescu 2000a). Al agregar el proverbial "nivel adicional de indirección" que resuelve muchos problemas de ingeniería de software, los traits le permiten tomar decisiones relacionadas con el tipo de letra fuera del contexto inmediato en el que se toman. Esto hace que el código resultante sea más limpio, más legible y más fácil de mantener.

Por lo general, escribirá sus propias plantillas y clases de traits cuando su código genérico las necesite. Ciertos traits, sin embargo, son aplicables a cualquier tipo. Pueden ayudar a los programadores genéricos a adaptar mejor el código de la plantilla a las capacidades de un tipo.

Supongamos, por ejemplo, que implementa un algoritmo de copia:

```cpp
template <typename InIt, typename OutIt>
OutIt Copy(InIt first, InIt last, OutIt result) {
	for (; first != last; ++first, ++result)
		*result = *first;
}
```

En teoría, no debería tener que implementar dicho algoritmo, ya que duplica la funcionalidad de `std::copy`. Pero es posible que deba especializar su rutina de copia para tipos específicos.

Supongamos que desarrolla código para una máquina multiprocesador que tiene una función primitiva `BitBlast` muy rápida, y le gustaría aprovechar esa primitiva siempre que sea posible.

```cpp
// Prototype of BitBlast in "SIMD_Primitives.h"
void BitBlast(const void* src, void* dest, size_t bytes);
```

`BitBlast`, por supuesto, funciona solo para copiar tipos primitivos y estructuras de datos antiguas simples. No puede usar `BitBlast` con tipos que tienen un constructor de copia no trivial. Entonces, le gustaría implementar `Copy` para aprovechar `BitBlast` siempre que sea posible y recurrir a un algoritmo más general y conservador para tipos elaborados. De esta manera, las operaciones de `Copy` en rangos de tipos primitivos se ejecutarán  automáticamente" más rápido.

Lo que necesitas aquí son dos pruebas:

- ¿Son punteros regulares `InIt` y `OutIt` (a diferencia de los tipos de iterador más sofisticados)?
- ¿El tipo al que apuntan `InIt` y `OutIt` se puede copiar con copia bit a bit?

Si puede encontrar respuestas a estas preguntas en el momento de la compilación y si la respuesta a ambas preguntas es sí, puede usar `BitBlast`. De lo contrario, debe confiar en el bucle genérico `for`.

Los traits de tipo ayudan a resolver tales problemas. Los traits de tipo en este capítulo le deben mucho a la implementación de traits de tipo que se encuentra en la biblioteca `Boost C++`.

### Implementación de características de puntero

Loki define una plantilla de clase `TypeTraits` que recopila una gran cantidad de características de tipo genérico. `TypeTraits` utiliza la especialización de plantilla internamente y expone los resultados.

La implementación de la mayoría de las características de tipo se basa en la especialización de plantilla total o parcial (Sección [Especialización de plantilla parcial](#especializaci%c3%b3n-de-plantilla-parcial)). Por ejemplo, el siguiente código determina si un tipo `T` es un puntero:

```cpp
template <typename T>
class TypeTraits {
private:
	template <class U> struct PointerTraits {
		enum { result = false };
		typedef NullType PointeeType;
	};

	template <class U> struct PointerTraits<U*> {
		enum { result = true };
		typedef U PointeeType;
	};

public:
	enum { isPointer = PointerTraits<T>::result };
	typedef PointerTraits<T>::PointeeType PointeeType;
	...
};
```

La primera definición introduce la plantilla de clase `PointerTraits`, que dice: "`T` no es un puntero y no se aplica un tipo de puntero". Recuerde de la Sección [NullType &amp; EmptyType](#nulltype-amp-emptytype) que `NullType` es un tipo de marcador de posición para casos "no aplica".

La segunda definición (la línea en negrita) introduce una especialización parcial de `PointerTraits`, una especialización que coincide con cualquier tipo de puntero. Para los punteros a cualquier cosa, la especialización en negrita califica como una mejor coincidencia que la plantilla genérica para cualquier tipo de puntero. En consecuencia, la especialización entra en acción para un puntero, por lo que el resultado se evalúa como verdadero. Además, `PointeeType` se define adecuadamente.

Ahora puede obtener una idea de la implementación de `std::vector::iterator` ¿es un puntero simple o un tipo elaborado?

```cpp
int main() {
	const bool iterIsPtr = TypeTraits<vector<int>::iterator>::isPointer;
	cout << "vector<int>::iterator is " << iterIsPtr ? "fast" : "smart" << '\n';
}
```

Del mismo modo, `TypeTraits` implementa una constante `isReference` y una definición de tipo `ReferencedType`. Para un tipo de referencia `T`, `ReferencedType` es el tipo al que se refiere `T`; si `T` es un tipo directo, `ReferencedType` es `T` en sí.

La detección de punteros a miembros (consulte el Capítulo 5 para obtener una descripción de los punteros a miembros) es un poco diferente. La especialización necesaria es la siguiente:

```cpp
template <typename T>
class TypeTraits {
private:
	template <class U> struct PToMTraits {
		enum { result = false };
	};

	template <class U, class V>
	struct PToMTraits<U V::*> {
		enum { result = true };
	};
public:
	enum { isMemberPointer = PToMTraits<T>::result };
	...
};
```

### Detección de tipos fundamentales

`TypeTraits<T>` implementa una constante de tiempo de compilación `isStdFundamental` que dice si `T` es o no un tipo fundamental estándar. Los tipos fundamentales estándar consisten en el tipo `void` y todos los tipos numéricos (que a su vez son de coma flotante y tipos integrales). `TypeTraits` define constantes que revelan las categorías a las que pertenece un tipo determinado.

Al precio de anticipar un poco, debe decirse que la magia de las *typelist* (listas de tipos) (Capítulo 3) hace que sea fácil detectar si un tipo pertenece a un conjunto conocido de tipos. Por ahora, todo lo que debes saber es que la expresión:

```cpp
TL::IndexOf<T, TYPELIST_nn(comma-separated list of types)>::value
```

(donde *`nn`* es el número de tipos en la lista de tipos) devuelve la posición de `T` basada en cero en la lista, o –1 si `T` no figura en la lista. Por ejemplo, la expresión:

```cpp
TL::IndexOf<T, TYPELIST_4(signed char, short int, int, long int)>::value
```

es mayor o igual que cero si y solo si `T` es un tipo integral con signo.

La siguiente es la definición de la parte de `TypeTraits` dedicada a los tipos primitivos.

```cpp
template <typename T>
class TypeTraits {
	... as above ...
public:
	typedef TYPELIST_4(unsigned char, unsigned short int, unsigned int, unsigned long int) UnsignedInts;
	typedef TYPELIST_4(signed char, short int, int, long int) SignedInts;
	typedef TYPELIST_3(bool, char, wchar_t) OtherInts;
	typedef TYPELIST_3(float, double, long double) Floats;
	enum { isStdUnsignedInt = TL::IndexOf<T, UnsignedInts>::value >= 0 };
	enum { isStdSignedInt = TL::IndexOf<T, SignedInts>::value >= 0 };
	enum { isStdIntegral = isStdUnsignedInt || isStdSignedInt ||
	TL::IndexOf <T, OtherInts>::value >= 0 };
	enum { isStdFloat = TL::IndexOf<T, Floats>::value >= 0 };
	enum { isStdArith = isStdIntegral || isStdFloat };
	enum { isStdFundamental = isStdArith || isStdFloat || Conversion<T, void>::sameType };
	...
};
```

El uso de listas de tipos y `TL::IndexOf` le brinda la capacidad de inferir información rápidamente sobre los tipos, sin tener que especializarse en una plantilla muchas veces. Si no puede resistir la tentación de profundizar en los detalles de las listas de tipos y `TL::IndexOf`, eche un vistazo al Capítulo 3, pero no olvide volver aquí.

La implementación real de la detección de tipos fundamentales es más sofisticada, lo que permite tipos de extensión específicos del proveedor (como int64 o long long).

### Tipos de parámetros optimizados

En el código de plantilla, a veces necesita responder la siguiente pregunta: Dado un tipo arbitrario `T`, ¿cuál es la forma más eficiente de pasar y aceptar objetos de tipo `T` como argumentos para las funciones? En general, la forma más eficiente es pasar tipos elaborados por referencia y tipos escalares por valor. (Los tipos escalares consisten en los tipos aritméticos descritos anteriormente, así como las `enums`, punteros y punteros a los miembros). Para los tipos elaborados, evita la sobrecarga de un temporal adicional (llamadas de constructor más destructor), y para los tipos escalares, evita los sobrecarga de la indirección resultante de la referencia.

Un detalle que debe manejarse con cuidado es que `C++` no permite referencias a referencias. Por lo tanto, si `T` *ya es* una referencia, no debe agregarle una referencia más.

Un poco de análisis sobre el tipo de parámetro óptimo para una llamada de función genera el siguiente algoritmo. Llamemos al tipo de parámetro que buscamos `ParameterType`.

- Si `T` es una referencia a algún tipo, `ParameterType` es lo mismo que `T` (sin cambios). *Motivo*: No se permiten referencias a referencias.
- De lo contrario:
- Si `T` es un tipo escalar (`int`, `float`, etc.), `ParameterType` es `T`. *Motivo*: los tipos primitivos se pasan mejor por valor.
- De lo contrario `ParameterType` es `const T &`. Motivo: en general, los tipos no primitivos se pasan mejor por referencia.

Un logro importante de este algoritmo es que evita el error de referencia a referencia, que podría aparecer si combina las funciones de biblioteca estándar `bind2nd` con `mem_fun`.

Es fácil implementar `TypeTraits::ParameterType` utilizando las técnicas que ya tenemos a mano y las características definidas anteriormente: `ReferencedType` y `isPrimitive`.

```cpp
template <typename T>
class TypeTraits {
	... as above ...
public:
	typedef Select<isStdArith || isPointer || isMemberPointer, T, ReferencedType&>::Result ParameterType;
};
```

Desafortunadamente, este esquema no pasa los tipos enumerados (`enum`) por valor porque no hay una forma conocida de determinar si un tipo es un `enum` o no.

La plantilla de clase `Functor` definida en el Capítulo 5 usa `TypeTraits::ParameterType`.

### Stripping Qualifiers

Dado un tipo `T`, puede llegar fácilmente a su hermano constante simplemente escribiendo `const T`. Sin embargo, hacer lo contrario (quitar la `const` de un tipo) es un poco más difícil. Del mismo modo, a veces es posible que desee deshacerse del calificador `volatile` de un tipo.

Considere, por ejemplo, construir una clase de puntero inteligente `SmartPtr` (el Capítulo 7 trata los punteros inteligentes en detalle). Aunque desea permitir a los usuarios crear punteros inteligentes para objetos constantes, como en `SmartPtr<const Widget>`, aún necesita modificar el puntero a `Widget` que tiene internamente. En este caso, dentro de `SmartPtr` necesita obtener `Widget` de `const Widget`.

Implementar un "`const` stripper" es fácil, de nuevo mediante el uso de especialización parcial de plantilla:

```cpp
template <typename T>
class TypeTraits {
	... as above ...
private:
	template <class U> struct UnConst {
		typedef U Result;
	};
	template <class U> struct UnConst<const U> {
		typedef U Result;
	};
public:
	typedef UnConst<T>::Result NonConstType;
};
```

### Usando `TypeTraits`

`TypeTraits` puede ayudarlo a hacer muchas cosas interesantes. Por un lado, ahora puede implementar la rutina `Copy` para usar `BitBlast` (el problema mencionado en la Sección [Características de tipo](#caracter%c3%adsticas-de-tipo)) simplemente ensamblando las técnicas presentadas en este capítulo. Puede usar `TypeTraits` para averiguar la información de tipo sobre los dos iteradores y la plantilla `Int2Type` para enviar la llamada a `BitBlast` o a una rutina de copia clásica.

```cpp
enum CopyAlgoSelector { Conservative, Fast };

// Conservative routine-works for any type
template <typename InIt, typename OutIt>
OutIt CopyImpl(InIt first, InIt last, OutIt result, Int2Type<Conservative>) {
	for (; first != last; ++first, ++result)
		*result = *first;
	return result;
}

// Fast routine-works only for pointers to raw data
template <typename InIt, typename OutIt>
OutIt CopyImpl(InIt first, InIt last, OutIt result, Int2Type<Fast>) {
	const size_t n = last-first;
	BitBlast(first, result, n * sizeof(*first));
	return result + n;
}

template <typename InIt, typename OutIt>
OutIt Copy(InIt first, InIt last, OutIt result) {
	typedef TypeTraits<InIt>::PointeeType SrcPointee;
	typedef TypeTraits<OutIt>::PointeeType DestPointee;
	enum { copyAlgo =
		TypeTraits<InIt>::isPointer &&
	TypeTraits<OutIt>::isPointer &&
	TypeTraits<SrcPointee>::isStdFundamental &&
	TypeTraits<DestPointee>::isStdFundamental &&
	sizeof(SrcPointee) == sizeof(DestPointee) ? Fast :
		Conservative };
	return CopyImpl(first, last, result, Int2Type<copyAlgo>);
}
```

Aunque `Copy` en sí mismo no hace mucho, la parte interesante está ahí. El valor de `enum` `copyAlgo` selecciona una implementación u otra. La lógica es la siguiente: use `BitBlast` si los dos iteradores son punteros, si ambos tipos apuntados son fundamentales y si los tipos apuntados son del mismo tamaño. La última condición es un giro interesante. Si haces esto:

```cpp
int* p1 = ...;
int* p2 = ...;
unsigned int* p3 = ...;
Copy(p1, p2, p3);
```

luego `Copy` llama a la rutina rápida, como debería, aunque los tipos de origen y destino son diferentes.

El inconveniente de `Copy` es que no acelera todo lo que podría acelerarse. Por ejemplo, es posible que tenga una `struct` simple tipo `C` que no contenga nada más que datos primitivos: una estructura llamada *datos antiguos simples* o POD. El estándar permite la copia a nivel de bits de estructuras POD, pero `Copy` no puede detectar "PODness", por lo que llamará a la rutina lenta. Aquí tiene que confiar, nuevamente, en características clásicos además de `TypeTraits`. Por ejemplo:

```cpp
template <typename T> struct SupportsBitwiseCopy {
	enum { result = TypeTraits<T>::isStdFundamental };
};

template <typename InIt, typename OutIt>
OutIt Copy(InIt first, InIt last, OutIt result, Int2Type<true>) {
	typedef TypeTraits<InIt>::PointeeType SrcPointee;
	typedef TypeTraits<OutIt>::PointeeType DestPointee;
	enum { useBitBlast =
		TypeTraits<InIt>::isPointer &&
		TypeTraits<OutIt>::isPointer &&
		SupportsBitwiseCopy<SrcPointee>::result &&
		SupportsBitwiseCopy<DestPointee>::result &&
		sizeof(SrcPointee) == sizeof(DestPointee) };
	return CopyImpl(first, last, Int2Type<useBitBlast>);
}
```

Ahora, para liberar `BitBlast` para sus tipos de interés de POD, solo necesita especializarse en `SupportsBitwiseCopy` y poner un `true` allí:

```cpp
template<> struct SupportsBitwiseCopy<MyType> {
	enum { result = true };
};
```

### Terminando

La Tabla [Table TypeTraits<T> Members](#table-typetraitst-members) define el conjunto completo de rasgos implementados por Loki.

## Resumen

Una serie de técnicas de los componentes básicos de los componentes presentes en este libro. La mayoría de las técnicas están relacionadas con el código de plantilla.

- *Las aserciones en tiempo de compilación* (Sección [Assertions en tiempo de compilación](#assertions-en-tiempo-de-compilaci%c3%b3n)) ayudan a las bibliotecas a generar mensajes de error significativos en código con plantilla.
- *La especialización parcial de plantilla* (Sección [Especialización de plantilla parcial](#especializaci%c3%b3n-de-plantilla-parcial)) le permite especializar una plantilla, no para un conjunto específico de parámetros fijos, sino para una familia de parámetros que coinciden con un patrón.
- *Las clases locales* (Sección [Clases locales](#clases-locales)) le permiten hacer cosas interesantes, especialmente dentro de las funciones de plantilla.
- *La asignación de constantes integrales a tipos* (Sección [Asignación de constantes integrales a tipos](#asignaci%c3%b3n-de-constantes-integrales-a-tipos)) facilita el despacho en tiempo de compilación en función de valores numéricos (especialmente condiciones booleanas).
- *El mapeo de tipo a tipo* (Sección [Mapeo de tipo a tipo](#mapeo-de-tipo-a-tipo)) le permite sustituir la sobrecarga de funciones por la especialización parcial de plantilla de funciones, una característica que falta en C++.
- *La selección de tipo* (Sección [Selección de tipo](#selecci%c3%b3n-de-tipo)) le permite seleccionar tipos basados ​​en condiciones booleanas.
- *La detección de la convertibilidad y la herencia en tiempo de compilación* (Sección [Detección de convertibilidad y herencia en tiempo de compilación](#detecci%c3%b3n-de-convertibilidad-y-herencia-en-tiempo-de-compilaci%c3%b3n)) le brinda la capacidad de determinar si dos tipos arbitrarios son convertibles entre sí, son alias del mismo tipo o heredan uno del otro.
- `TypeInfo` (Sección [Un envoltorio alrededor type_info](#un-envoltorio-alrededor-typeinfo)) implementa un contenedor alrededor de `std::type_info`, con semántica de valores y comparaciones de pedidos.
- Las clases `NullType` y `EmptyType` (Sección [NullType &amp; EmptyType](#nulltype-amp-emptytype)) funcionan como tipos de marcador de posición en la metaprogramación de plantillas.
- La plantilla `TypeTraits` (Sección [Características de tipo](#caracter%c3%adsticas-de-tipo)) ofrece una serie de rasgos de propósito general que puede usar para adaptar el código a categorías específicas de tipos.

## Table TypeTraits<T> Members

| Nombre             | Tipo              | Comentarios                                                                                                                             |
| ------------------ | ----------------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `isPointer`        | Boolean, constant | True if `T` is a pointer.                                                                                                               |
| `PointeeType`      | Type              | Evaluates to the type to which `T` points, if `T` is a pointer. Otherwise, evaluates to `NullType`.                                     |
| `isReference`      | Boolean, constant | True if `T` is a reference.                                                                                                             |
| `ReferencedType`   | Type              | If `T` is a reference, evaluates to the type to which `T` refers. Otherwise, evaluates to the type `T` itself.                          |
| ParameterType      | Type              | The type that's most appropriate as a parameter of a nonmutable function. Can be either T or const T&.                                  |
| `isConst`          | Boolean, constant | True if `T` is a `const`-qualified type.                                                                                                |
| `NonConstType`     | Type              | Removes the `const` qualifier, if any, from type `T`.                                                                                   |
| `isVolatile`       | Boolean, constant | True if `T` is a `volatile`-qualified type.                                                                                             |
| `NonVolatileType`  | Type              | Removes the `volatile` qualifier, if any, from type `T`.                                                                                |
| `NonQualifiedType` | Type              | Removes both the `const` and `volatile` qualifiers, if any, from type `T`.                                                              |
| `isStdUnsignedInt` | Boolean, constant | True if `T` is one of the four unsigned integral types (`unsigned char`, `unsigned short int`, `unsigned int`, or `unsigned long int`). |
| `isStdSignedInt`   | Boolean, constant | True if `T` is one of the four signed integral types (`signed char`, `short int`, `int`, or `long int`).                                |
| `isStdIntegral`    | Boolean, constant | True if `T` is a standard integral type.                                                                                                |
| `isStdFloat`       | Boolean, constant | True if `T` is a standard floating-point type (`float`, `double`, or `long double`).                                                    |
| `isStdArith`       | Boolean, constant | True if T is a standard arithmetic type (integral or floating point).                                                                   |
| `isStdFundamental` | Boolean, constant | True if `T` is a fundamental type (arithmetic or `void`).                                                                               |
