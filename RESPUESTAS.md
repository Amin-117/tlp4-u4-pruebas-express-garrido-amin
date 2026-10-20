# Laboratorio de pruebas automatizadas en Express.js

**Taller de Lenguajes de Programación IV · Unidad 4 · Octubre de 2026**

## 10. Registro de resultados

| Comprobación | Resultado | Observación |
| --- | --- | --- |
| Respuesta de `GET /healthy` | `{"estado":"ok"}` | Verificado en `http://localhost:3000/healthy`. El cálculo manual con `Invoke-RestMethod` devolvió `{"total":900}`. |
| Primera ejecución de `npm test` | 4 pasan, 1 falla | Falla `rechaza descuentos superiores al 100 %`: la función devuelve -500 en vez de lanzar `RangeError`. |
| Corrección del servicio | 5 de 5 pasan | Se agregaron validaciones de precio (finito, ≥ 0) y descuento (finito, entre 0 y 100) en `src/services/precios.ts`, conservando la fórmula. |
| Nuevas pruebas unitarias | 9 de 9 pasan | Casos: descuento 100 % → 0; precio -100 → `RangeError`; descuento -10 → `RangeError`; desafío: precio `NaN` → `RangeError`. |
| Prueba de integración HTTP 400 | 10 de 10 pasan | `POST /api/precios/calcular` con descuento 150 responde 400 y el cuerpo tiene la propiedad `error`. |
| Verificación `npm run typecheck` | Sin errores | `tsc --noEmit` no reportó problemas de tipos. |

## 2. Conocer la aplicación

**¿Qué archivo contiene la regla de cálculo?**
`src/services/precios.ts`, en la función `calcularPrecioFinal(precio, descuento)`.

**¿Qué archivo define la ruta HTTP?**
`src/app.ts`, que define `GET /healthy` y `POST /api/precios/calcular`.

**¿En qué se diferencian sus responsabilidades?**
`precios.ts` contiene la lógica de negocio: cómo se calcula un precio y qué valores son válidos. No sabe nada de HTTP. `app.ts` se ocupa de la comunicación: recibe la petición, extrae `precio` y `descuento` del JSON, llama al servicio y arma la respuesta (200 con el total, o 400 con un `error` si el servicio lanza `RangeError` o `TypeError`). Separarlos permite probar la regla de cálculo de forma aislada y reutilizarla desde otros lugares.

## 4. Interpretar la falla

**1. ¿Qué total devuelve actualmente un descuento del 150 % sobre $1.000?**
-500, porque `1000 - (1000 × 150) / 100 = 1000 - 1500 = -500`.

**2. ¿Por qué ese resultado no cumple la regla de negocio?**
Un descuento no puede superar el 100 % del precio: no tiene sentido un precio final negativo (sería pagarle al cliente por llevarse el producto). La regla exige rechazar ese descuento lanzando un `RangeError`.

**3. ¿Por qué la prueba falla si la función terminó de ejecutarse?**
Porque la prueba no comprueba que la función termine, sino que lance un `RangeError`. Al terminar normalmente y devolver -500, no se produjo la excepción esperada, así que la aserción `toThrow(RangeError)` no se cumple. Que el código se ejecute sin errores no significa que se comporte como pide el requisito.

## 7. Primera prueba de integración HTTP

**¿Por qué esta comprobación es de integración y no solamente una prueba unitaria?**
Porque no llama a `calcularPrecioFinal` directamente: envía una petición HTTP real a la aplicación Express mediante Supertest. Para que la prueba pase deben colaborar varias partes: Express tiene que interpretar el JSON, la ruta tiene que llamar al servicio, el servicio tiene que lanzar `RangeError` y la ruta tiene que convertir ese error en un status 400 con la propiedad `error`. Si cualquiera de esos eslabones fallara (por ejemplo, si la ruta devolviera 500 o un 200 con el error), la prueba unitaria del servicio seguiría pasando, pero esta no.

## 8. Diferenciar pruebas y verificación de tipos

**¿Por qué conviene utilizar tanto `typecheck` como `npm test`?**
Porque detectan problemas distintos. `npm test` ejecuta el código y verifica comportamientos: que con ciertas entradas se obtengan ciertos resultados. `npm run typecheck` (`tsc --noEmit`) analiza el código sin ejecutarlo y detecta incompatibilidades de tipos en todo el proyecto, incluso en partes que ninguna prueba recorre. Además, Node ejecuta los `.ts` eliminando los tipos sin verificarlos, así que un error de tipos puede pasar inadvertido al correr la app o las pruebas. Usar ambos da una verificación más completa.

## 11. Preguntas de cierre

**1. ¿Qué diferencia hay entre comprobar manualmente una respuesta y ejecutar una prueba automatizada?**
En la comprobación manual una persona ejecuta la acción (abrir el navegador, enviar el `Invoke-RestMethod`) y evalúa a simple vista si el resultado es correcto; es útil para explorar, pero depende de la atención humana y hay que repetirla a mano cada vez. En la prueba automatizada, la entrada y el resultado esperado están escritos en código, se ejecuta en segundos con `npm test` y siempre verifica lo mismo, de forma repetible, en la terminal o en la CI.

**2. ¿Cuál fue la causa de la primera prueba fallida?**
`calcularPrecioFinal` no validaba los valores de entrada. Con un descuento del 150 % hacía el cálculo igual y devolvía -500, cuando la regla de negocio exigía lanzar un `RangeError`. El defecto estaba en la implementación, no en la prueba.

**3. ¿Por qué es útil incluir casos inválidos además de los casos habituales?**
Porque los errores suelen aparecer en los bordes y en las entradas inesperadas, no en el caso típico. La función original calculaba bien el 10 %, pero aceptaba 150 %, precios negativos o `NaN`. Los casos inválidos verifican que el sistema rechace lo que debe rechazar y documentan esas reglas; los casos límite (0 % y 100 %) confirman que los valores extremos permitidos siguen funcionando.

**4. ¿Qué parte del sistema comprueba `tests/precios.test.ts` y qué partes comprueba `tests/api.test.ts`?**
`tests/precios.test.ts` comprueba solo el servicio de negocio (`calcularPrecioFinal` en `src/services/precios.ts`), de forma aislada: son pruebas unitarias. `tests/api.test.ts` comprueba la aplicación Express de `src/app.ts` junto con ese servicio: rutas, lectura del JSON, códigos de estado (200, 400) y forma del cuerpo de la respuesta. Son pruebas de integración.

**5. ¿Qué ventajas ofrece ejecutar las pruebas después de cada modificación?**
Permite detectar regresiones enseguida, es decir, fallas introducidas al cambiar algo que antes funcionaba. Como el cambio es reciente y pequeño, es fácil identificar la causa. También da confianza para modificar o refactorizar el código y, en el laboratorio, sirvió para confirmar paso a paso que la corrección arreglaba la falla sin romper los otros casos.

## Preguntas de repaso (material teórico)

**1. ¿Por qué una API que devuelve una respuesta HTTP 200 puede contener errores de negocio?**
Porque el 200 solo indica que la solicitud se procesó sin una falla técnica, no que el resultado respete las reglas del negocio. Antes de la corrección, un descuento del 150 % producía un total de -500 con una respuesta normal. Por eso hay que verificar el contenido de la respuesta, no solo el código de estado.

**2. ¿Qué diferencia hay entre una prueba unitaria y una de integración?**
La unitaria comprueba una unidad pequeña de forma aislada, como `calcularPrecioFinal` llamada directamente. La de integración comprueba la colaboración entre varias partes: con Supertest, la petición pasa por la ruta de Express, el servicio de cálculo y el manejo de errores que convierte un `RangeError` en un 400.

**3. ¿Por qué `toThrow` recibe una función en lugar del resultado de ejecutarla?**
Si se escribe `expect(calcularPrecioFinal(1000, 150))`, la función se ejecuta antes de llegar a `expect` y, si lanza el error, este interrumpe la prueba antes de que `toThrow` pueda inspeccionarlo. Al pasar `() => calcularPrecioFinal(1000, 150)`, Vitest ejecuta la función de forma controlada y puede verificar si lanzó el error y de qué tipo.

**4. ¿Qué diferencia existe entre `npm test` y `npm run typecheck`?**
`npm test` ejecuta `vitest run`: corre las pruebas y verifica comportamientos. `npm run typecheck` ejecuta `tsc --noEmit`: analiza los tipos sin ejecutar el código ni generar archivos JavaScript. Uno detecta resultados incorrectos; el otro, incoherencias de tipos.

**5. ¿Qué aporta GitHub Actions al trabajo colaborativo?**
Automatiza las verificaciones (instalar dependencias con `npm ci`, comprobar tipos y correr las pruebas) en cada `push`, en un entorno limpio y reproducible. Así cualquier integrante sabe enseguida si un cambio rompió algo, antes de integrarlo, y el equipo comparte un mismo criterio de "en verde". No reemplaza la revisión humana ni garantiza ausencia de defectos.
