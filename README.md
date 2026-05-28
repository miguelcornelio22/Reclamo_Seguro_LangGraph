# Proyecto M3 – Reclamo de Seguro  
**LangGraph con CSV externo, Checkpoint, Replay, Fork y Loop controlado**

## 📌 Descripción
Este proyecto implementa un flujo de procesamiento de reclamos de seguro utilizando **LangGraph**.  
La lógica se basa en un grafo de estados que valida pólizas y coberturas desde un archivo CSV, estima montos según el tipo de cobertura, y decide automáticamente o con intervención humana.  
Se incluyen mecanismos de **checkpoint, replay y fork**, además de un **loop controlado** para evitar recursión infinita.

---

## 🔹 Lógica de los nodos

- **recibir_reclamo**  
  Registra el reclamo inicial y añade un evento de recepción.

- **validar_poliza**  
  Verifica en el CSV si el cliente tiene póliza registrada.  
  - Si no existe, genera error y decisión `"Sin póliza"`.  
  - Si existe, guarda el tipo de póliza.

- **validar_cobertura**  
  Comprueba cobertura, vencimiento y monto máximo de la póliza.  
  - Si la póliza está vencida o sin cobertura, genera error y decisión `"Sin cobertura"`.  
  - Si es válida, guarda cobertura, vencimiento y monto base.

- **estimar_monto**  
  Ajusta el monto según cobertura:  
  - Cobertura **total** → 90% del monto de cobertura.  
  - Cobertura **parcial** → 40% del monto de cobertura.  
  - Otro caso → monto sin ajuste.  
  Registra el cálculo en eventos.

- **tomar_decision**  
  Nodo central que evalúa monto, errores y decisión humana.  
  Reglas:  
  - Si hay errores → decisión final según error.  
  - Si monto ≤ 3000 → aprobado automático.  
  - Si monto > 3000 → requiere aprobación humana.  
  - Si hay decisión humana → aprobado o rechazado.  
  - Si no hay decisión humana tras 2 intentos → corta loop y marca `"Revisión"`.  
  Controla el loop con el campo `intentos`.

- **solicitar_aprobacion**  
  Nodo que espera intervención humana.  
  Regresa a `tomar_decision` para reevaluar con la decisión humana.

- **generar_respuesta**  
  Cierra el flujo con la respuesta final al cliente.  
  Incluye la decisión tomada en los eventos.

---

## 🔹 Lógica del grafo

1. Flujo lineal:  
   `recibir_reclamo → validar_poliza → validar_cobertura → estimar_monto → tomar_decision`  
2. Condicional en `tomar_decision`:  
   - `"Pendiente"` → va a `solicitar_aprobacion`.  
   - Cualquier otra decisión → va a `generar_respuesta`.  
3. Loop controlado:  
   `solicitar_aprobacion → tomar_decision`  
4. Finalización:  
   `generar_respuesta → END`

---

## 🔹 Mecanismos adicionales

- **Checkpoint**: permite guardar el estado en un punto intermedio (ej. antes de cobertura) y reanudar desde allí.  
- **Replay**: reejecuta el flujo desde un checkpoint para reproducir resultados.  
- **Fork**: modifica el estado en un checkpoint (ej. cambiar póliza) y continúa el flujo recalculando con los nuevos datos.  
- **Loop controlado**: el contador `intentos` evita recursión infinita en la aprobación humana.

---

## ✅ Conclusión
La lógica del grafo asegura que:  
- Los reclamos se validan contra datos externos (CSV).  
- El monto se ajusta según cobertura.  
- Las decisiones pueden ser automáticas o humanas.  
- Los errores se manejan con salidas claras (`Sin póliza`, `Sin cobertura`).  
- El loop de aprobación está controlado con un límite de intentos.  
- Se soportan **checkpoint, replay y fork** para trazabilidad y corrección de flujo.
