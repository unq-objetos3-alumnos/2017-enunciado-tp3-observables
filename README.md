# TP3 - Deprecando el Observer
# Parte 1

> The Observer is dead, long live the observer

Se desea crear una librería que permita crear y manipular `Observables`,
capaces de emitir elementos (eventos), de manera que aquellos que se 
subscriban a estos observables puedan reaccionar a dichas secuencias 
de elementos.

```scala
def saludador(names: List[String]) = {
  Observable.from(names).subscribe { n =>
    println(s"Hola $n!")
  }
}
```

Si bien en el contexto de este trabajo vamos a tratar con ejemplos 
sincrónicos, la idea es poder extrapolar lo que implementemos a
casos asincrónicos (concurrentes).

## Creación de observables

### Create

Queremos implementar la función `create`, que recibe como argumento
una función de `Subscriber => Unit`, donde `Subscriber` tiene la interfaz:

```scala
trait Subscriber[-T] {
  /**
    * Recibe un elemento nuevo
    *
    * @param t el elemento emitido
    */
  def onNext(t: T): Unit

  /**
    * Termina con error.
    * 
    * @param t el throwable emitido
    */
  def onError(t: Throwable): Unit

  /**
    * Termina con success.
    * No se enviarán más eventos
    */
  def onComplete(): Unit
}
```

Por ejemplo, si quisiéramos crear un Observable que emitiese cierto rango de números:

```scala
val o = Observable.create { subscriber =>

    try {
      Range(1,5).foreach { num =>
        // se emite el elemento
        subscriber.onNext(num)
      }
      // se completa exitosamente
      subscriber.onCompleted() 
    } catch (e:Exception) {
      //si hubo un fallo se emite el error
      subscriber.onError(e)
    }
}

o.subscribe { n =>
  println(n)
}

// Output:
// 1
// 2
// 3
// 4
```

Se debe tener en cuenta que puede haber muchas subscripciones al mismo 
Observable. Los observables no comienzan a emitir elementos hasta no recibir 
su primer `subscribe`.

La firma del método subscribe debe poder incluir parámetros opcionales que reciban
callbacks para los eventos de `onCompleted` y `onError`.


### From

Observable que emite una secuencia de elementos

```scala
val strings:Observable[String] = Observable.from("a", "b", "c")

val list = List(1,2,3)
val integers:Observable[Int] = Observable.from(list)
```

### Just

Observable que emite un único elemento

```scala
val oneObject:Observable[String] = Observable.just("one object")
```

### Empty

Observable que no emite elementos y termina normalmente.

```scala
val o = Observable.empty()
```

### Never

Observable que no emite elementos y nunca termina.


```scala
val o = Observable.never()
```

### Error

Observable que no emite ítems y termina con error.

```scala
val o = Observable.error(throwable:Throwable)
```

### Repeat

Dado un observable, queremos poder repetir sus elementos N veces o 
infinitas veces si no se especifica.


```scala
val listaObservable = Observable.from(1,2,3)
val listaRepetible = observableList.repeat(3)
listaRepetible.subscribe { n =>
  println(s"Hola $n!")
}

// Output:
// 1
// 2
// 3
// 1
// 2
// 3
// 1
// 2
// 3
```

## Observable y Subscriber al mismo tiempo: Subjects

Para facilitar la simulación de eventos asincrónicos, vamos a definir entidades que
pueden ser al mismo tiempo Observables y Subscribers:

```scala

val subject = Subject.create[Int]
val subscriber = subject.subscriber
val observable = subject.observable

observable.subscribe({ n =>
                       println(s"Recibí $n")
                     },
                     {
                       println(s"Completado sin errores")
                     }) 

subscriber.onNext(1) //Recibí 1
subscriber.onNext(9) //Recibí 9
subscriber.onComplete() //Completado sin errores

```


## Transformar Observables mediante operadores

Queremos poder aplicarle transformaciones a los elementos emitidos por 
los observables.

### Map

Dado un elemento emitido, aplicarle una función y emitir su resultado:

```scala
val listaObservable = Observable.from(1,2,3)
val listaIncrementada = listaObservable.map { n => n + 1}
listaIncrementada.subscribe { n =>
  println(n)
}

// Output:
// 2
// 3
// 4
```

### Scan

Dado un elemento emitido, tomar el elemento anterior, aplicarles una función, y emitir
su resultado (si no hubo elementos anteriores, se emite el primero):

```scala
val listaObservable = Observable.from(1,2,3,4)
val listaSumando = listaObservable.scan { (x,y) => x + y}
listaSumando.subscribe(println(_))

// Output:
// 1
// 3
// 6
// 10
```

### Reduce

Similar a `scan` pero espera al `onComplete` antes de emitir un elemento,
y sólo emite el resultado final.

```scala
val listaObservable = Observable.from(1,2,3,4)
val listaSumando = listaObservable.reduce { (x,y) => x + y}
listaSumando.subscribe(println(_))

// Output:
// 10
```

## Filtrar elementos emitidos por Observables

### Distinct

Omite valores repetidos

```scala
val listaObservable = Observable.from(1,1,7,2,3,2,4,1,5)
val listaSumando = listaObservable.distinct()
listaSumando.subscribe(println(_))

// Output:
// 1
// 7
// 2
// 3
// 4
// 5
```

### First, Last, Skip, Take, Filter

 - `first` emite el primer elemento y termina.
 - `last` emite el último elemento (antes del `onComplete`)
 - `skip(n)` omite los primeros n elementos
 - `take(n)` únicamente emite los primeros n elementos 
 - `filter` filtra según un predicado


## Combinando Observables

### Merge

Dados dos observables, retornar un observable que emita los eventos de ambos.

```scala
val s1 = Subject.create[Int]
val s2 = Subject.create[Int]
val merged = Observable.merge(s1, s2) // también puede ser s1.mergeWith(s2)
merged.subscribe(println(_))
s1.onNext(1) // 1
s2.onNext(2) // 2

```

### Join

Dados dos observables, cuando uno termina de emitir, comenzar a emitir los eventos del otro.

### Zip

Dados dos observables, tomar el último elemento de cada uno y emitir una tupla:

```scala
val s1 = Subject.create[Int]
val s2 = Subject.create[String]
val zipped = Observable.zip(s1, s2)
zipped.subscribe(println(_))

s1.onNext(1) 
s1.onNext(2) 
s2.onNext("A") // (1, "A")
s1.onNext(3)
s2.onNext("B") // (2, "B")
```

# Parte 2

### Refactorizando: mouse, clicks y colores

Refactorizar el siguiente código utilizando la librería 
creada en la parte 1 del TP. Prestar especial atención a la creación
de los Observables necesarios.

```scala
import javafx.application.Application
import javafx.scene.canvas.Canvas
import javafx.scene.input.{KeyCode, KeyEvent}
import javafx.scene.paint.Color
import javafx.scene.{Group, Scene}
import javafx.stage.Stage

object MouseApp {
  def main(args: Array[String]) {
    Application.launch(classOf[MouseApp], args: _*)
  }
}

class MouseApp extends Application {

  def start(primaryStage: Stage): Unit = {
    val root = new Group()
    var fillColor1 = Color.rgb(0xF0, 0x50, 0x60)
    var fillColor2 = Color.rgb(0x60, 0xF0, 0x50)
    var fillColor = fillColor2
    var diameter = 20
    primaryStage.setScene(new Scene(root, 800, 600))

    val canvas = new Canvas(800, 600)
    val gc = canvas.getGraphicsContext2D

    gc.setLineWidth(5)

    primaryStage.addEventHandler(KeyEvent.KEY_PRESSED, { keyEvent: KeyEvent =>
      if (keyEvent.getCode eq KeyCode.SPACE) {
        fillColor = fillColor1
        keyEvent.consume()
      } else if ((keyEvent.getCode eq KeyCode.Z) && diameter > 10) {
        diameter -= 2
      } else if ((keyEvent.getCode eq KeyCode.X) && diameter < 80) {
        diameter += 2
      }
    })

    primaryStage.addEventHandler(KeyEvent.KEY_RELEASED, { keyEvent: KeyEvent =>
      if (keyEvent.getCode eq KeyCode.SPACE) {
        fillColor = fillColor2
        keyEvent.consume()
      }
    })

    root.getChildren.add(canvas)

    canvas.setOnMouseDragged { event =>
      gc.setFill(fillColor)
      gc.fillOval(event.getSceneX, event.getSceneY, diameter, diameter)
    }

    primaryStage.show()
  }
}
```

## Preguntas

 - ¿Qué observable deberíamos crear para poder variar el color
según va pasando el tiempo? 
 - ¿Cómo podríamos implementar un deshacer (undo)?
 - Si movemos el mouse muy rápido no tenemos un trazo contínuo, 
 ¿Cómo implementaría una interpolación que muestre el trazo contínuo?
 - ¿Qué deberíamos agregar para poder comenzar a dibujar presionando 
 `Shift` en lugar de haciendo click (permitiendo ambas cosas convivir)?

Bonus: implementar alguno de los puntos anteriores. 
 
--- 

### Guías
 
  - [Documentación de JavaFX](http://docs.oracle.com/javase/8/javase-clienttechnologies.htm)
  - [Introducción a Reactive Programming](https://gist.github.com/staltz/868e7e9bc2a7b8c1f754)
  - Lectura adicional: [Your Mouse is a Database, by Erik Meijer](http://queue.acm.org/detail.cfm?id=2169076)
  - Lectura adicional: [Mastering observables](https://developer.couchbase.com/documentation/server/3.x/developer/java-2.0/observables.html)

  
