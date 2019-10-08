### El patrón Circuit Breaker

En un artículo anterior hablamos sobre como gestionar errores transitorios mediante la aplicación del [patrón Retry].

Recordemos que los errores transitorios son aquellos errores que se producen durante un breve lapso de tiempo y que se solucionan de manera automática.

El [patrón retry] funciona muy bien cuando los errores son de este tipo ya que sabemos (o creemos saber) que no van a repetirse en una llamada posterior. 
Sin embargo, se pueden dar situaciones en las que estos errores transitorios pasen a convertirse en fallos totales del servicio. En estos casos la estrategia del [patrón Retry] deja de tener sentido e incluso puede llegar a empeorar la situación consumiendo recursos críticos. En estas situaciones es preferible que la operación falle inmediatamente y pueda ser gestionado por el sistema. Además sería interesante que el servicio solo volviera a ser invocado si existieran posibilidades de que el resultado sea correcto. 

Aquí entra en juego el patrón Circuit Breaker.

El patrón Circuit Breaker evita que una aplicación intente de manera reiterada una operación que con probabilidad vaya a fallar, permitiendo que esta continúe con su ejecución sin malgastar recursos mientras el problema no se resuelva. Además este patrón puede detectar cuando se ha resuelto el problema permitiendo de esta manera volver a ejecutar la operación comprometida. Podemos entender este patrón como un proxy entre nuestra aplicación y el servicio remoto que se implementa como si fuera una máquina de estados que imita el comportamiento de un interruptor de un circuito eléctrico.

__Los estado son los siguientes__:

* **Closed**: El circuito está cerrado y el flujo fluye ininterrumpidamente. Este es el estado inicial, todo funciona bien, la aplicación funciona de la manera esperada y la llamada al recurso/servicio se realiza de manera normal. 

* **Open**: El circuito está abierto y el flujo interrumpido. En este estado todas las llamadas al recurso/servicio fallan inmediatamente, es decir no se realizan, devolviendo la última excepción conocida a la aplicación. 

* **Half-Open**:  El circuito está medio abierto (o medio cerrado) dando una oportunidad al al flujo para su restauración. En este estado la aplicación volverá a intentar realizar la petición al servicio/recurso que fallaba.  

__Como funcionan los cambios de estado__:

Como ya hemos comentado el estado inicial es **Closed**. El proxy mantiene un contador con el número de errores que se producen al realizar la llamada, si el número de errores excede el límite especificado por configuración el proxy establece el estado a **Open**. Además, esto punto es muy importante, 
al mismo tiempo se inicia un **temporizador** 

Mientras el estado sea **Open** las llamadas al servicio no se realizarán devolviendo de manera automática el último error conocido. El tiempo en que el proxy permanece en este estado lo marca la configuración del **temporizador**

Cuando el **temporizador** concluye su ciclo el estado pasa a ser **Half-Open**. En este estado la llamada al servicio vuelve a estar disponible al menos una vez y:

* Si la petición funciona correctamente se asume que el error se ha corregido, se establece a cero el contador de errores y se establece el estado del proxy a **Closed** de nuevo. Todo vuelve a funcionar correctamente. 

* Si por lo contrario se produce algún error en la petición se asume que el error continua, se establece de nuevo el estado a **Open** y se reinicia el **temporizador**. El servicio/recurso sigue siendo inaccesible.


<p align="center">
  <img src="https://raw.githubusercontent.com/ldavidgomez/circuit-breaker-pattern/master/circuit_breaker_flow.png">
</p>

Este sería un ejemplo de una implementación simple en Kotlin:

```kotlin
    @Throws(Exception::class)
    fun run(action: KFunction0<String>) {
        logger.info("state is $state")
        if (state == CircuitBreakerState.CLOSED) {
            try {
                action.invoke()
                resetCircuit()
                logger.info("Success calling external service")
            } catch (ex: Exception) {
                handleException(ex)
                throw Exception("Something was wrong")
            }

        } else {
            if (state == CircuitBreakerState.HALF_OPEN || isTimerExpired) {
                state = CircuitBreakerState.HALF_OPEN

                logger.info("Time to retry...")

                try {
                    action.invoke()
                    logger.info("Success when HALF_OPEN")
                    closeCircuit()
                } catch (ex: Exception) {
                    logger.info("Fails when HALF_OPEN")
                    openCircuit(ex)
                    throw Exception("Fails when HALF_OPEN")
                }

            } else {
                logger.info("Circuit is still opened. Retrying at ${lastFailure!!.plus(openTimeout)}")
                throw Exception("Circuit is still opened. Retrying at ${lastFailure!!.plus(openTimeout)}")
            }

        }
    }
```

Por supuesto las implementaciones pueden ser mucho más sofisticadas. Podemos, por ejemplo, realizar una implementación que incremente el tiempo del temporizador en función del número de veces que falle la petición con el estado Half-Open.

Al igual que en el resto de patrones es muy importante tener en cuenta que **la complejidad de la implementación debe responder a las necesidades reales de nuestra aplicación y a los requerimientos de negocio.**

Existen librerías que implementan de manera muy sencilla el patrón Circuit Breaker, como por ejemplo Spring, donde con unas simples anotaciones tenemos implementado el patrón.

```kotlin
    @CircuitBreaker(maxAttempts = 3, openTimeout = 5000L, resetTimeout = 20000L)
    fun run(): String {
        logger.info("Calling external service...")
        if (Math.random() > 0.5) {
            throw RemoteAccessException("Something was wrong...")
        }
        logger.info("Success calling external service")
        return "Success calling external service"
    }

    @Recover
    private fun fallback_run(): String {
        logger.error("Fallback for external service")
        return "Succes on fallback"
    }
```

Como podemos ver la implementación mediante esta librería es muy sencilla. En el ejemplo podemos observar que lo único que hay que hacer es añadir la anotación @CircuitBreaker en el método donde vamos a realizar la petición.

La configuramos con los parámetros:

* **maxAttempts**: número máximo de intentos fallidos antes de llamar al método de recuperación.
* **openTimeout**: periodo de tiempo durante el que debe producirse el número 
máximo de intentos fallidos.
* **resetTimeout**: temporizador para pasar del estado Open a Half-Open.

Ya solo nos queda añadir la anotación @Recover al método encargado de gestionar el error cuando nuestra aplicación se encuentra en estado Open.

Este patrón proporciona estabilidad y resistencia a nuestras aplicaciones, ayuda a evitar el consumo de recursos que impactan de manera directa en el rendimiento de nuestro sistema.

Es necesario indicar aquí también, al igual que hicimos al hablar del [patrón Retry], que es altamente recomendable guardar un registro de las operaciones fallidas ya que es una información de gran utilidad para ayudar a dimensionar correctamente las infraestructuras de un proyecto y a encontrar errores recurrentes y silenciados por la gestión de errores de la aplicación.

Podéis encontrar los ejemplos completos tratados en este artículo en nuestro [github].

### <a id="source-code"></a>

[github]: https://github.com/ldavidgomez/circuit-breaker-pattern


### <a id="source-code"></a>

[patrón Retry]: https://apiumhub.com/es/tech-blog-barcelona/el-patron-retry
