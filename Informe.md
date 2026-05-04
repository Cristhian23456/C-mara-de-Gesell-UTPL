# Informe de Errores

# 📋 Análisis Forense de Scripts - CamaraHesel

He analizado los scripts del repositorio mediante **análisis estático de código C#**. A continuación presento los hallazgos organizados por tipo de problema, con soluciones que puedes aplicar editando directamente los archivos `.cs` en cualquier editor de texto (VS Code, Notepad++, etc.).

---

## 🔴 ERRORES CRÍTICOS DE COMPILACIÓN

### 1. **Listas Genéricas sin Tipo Declarado** ⚠️
**Archivos afectados:** `ApiManager.cs`, `DialogosManager.cs`, `LoadData.cs`, `Entidades/*.cs`

```csharp
// ❌ CÓDIGO CON ERROR (no compila en C# moderno)
public List dialogosList = new List();  // Falta el tipo <T>

// ✅ SOLUCIÓN
public List<Dialogos> dialogosList = new List<Dialogos>();
```

**Impacto:** El código NO COMPILARÁ en Unity 2020+ con C# 9.0+
**Solución:** Reemplazar TODAS las instancias de `List` por `List<TipoEspecífico>`:
- `List<Dialogos>` para listas de diálogos
- `List<Preguntas>` para preguntas
- `List<Historial>` para historial

---

### 2. **Eventos Action con Tipo Genérico Mal Formado**
**Archivo:** `ApiManager.cs`, líneas ~25-30

```csharp
// ❌ ERROR: Sintaxis inválida
public event Action> DialogosCargadosEvent;

// ✅ SOLUCIÓN
public event Action<List<Dialogos>> DialogosCargadosEvent;
```

**Explicación:** El símbolo `>` está mal ubicado. La sintaxis correcta para eventos con parámetros es `Action<Tipo>`.

---

### 3. **GetComponent() sin Tipo Especificado**
**Archivos afectados:** `BeckInventory.cs`, `FichaDiagnostico.cs`, `DialogosManager.cs`, `ManagerData.cs`

```csharp
// ❌ ERROR: No especifica qué componente buscar
saveData = GameObject.Find("LoginController").GetComponent();

// ✅ SOLUCIÓN
saveData = GameObject.Find("LoginController").GetComponent<SaveData>();
```

**Impacto:** Error de compilación: "The type or namespace name could not be found"

---

## 🟡 ERRORES LÓGICOS Y DE DISEÑO

### 4. **Fuga de Memoria: Eventos No Desuscritos**
**Archivo:** `DialogosManager.cs`

```csharp
// ❌ PROBLEMA: Suscripción sin desuscripción
void Start() {
    apiManager.DialogosCargadosEvent += OnDialogosInicialCargados;
}
// Nunca se desuscribe → referencia retenida → memory leak

// ✅ SOLUCIÓN: Implementar OnDestroy
private void OnDestroy()
{
    if (apiManager != null)
    {
        apiManager.DialogosCargadosEvent -= OnDialogosInicialCargados;
        apiManager.DialogosCargadosDesarrolladoEvent -= OnDialogosDesarrolloCargados;
        apiManager.DialogosCargadosFinalEvent -= OnDialogosFinCargados;
    }
}
```

---

### 5. **Cálculo de Porcentaje con División Entera**
**Archivo:** `Calificacion.cs`, método `porcentaje()`

```csharp
// ❌ ERROR: División entera produce 0 si contPre < preguntasCant
ValorPorcentaje = (100 * contPre) / preguntasCant;  // Si contPre=1, preguntasCant=3 → 33.33 se trunca a 33

// ✅ SOLUCIÓN: Usar casting a double
ValorPorcentaje = (100.0 * contPre) / preguntasCant;
// O mejor, usar decimal para precisión en psicometría
ValorPorcentaje = Math.Round((100.0 * contPre) / preguntasCant, 2);
```

---

### 6. **Comparación de Floats/Doubles sin Tolerancia**
**Archivo:** `BeckInventory.cs`, `FichaDiagnostico.cs`

```csharp
// ❌ PROBLEMA: Comparación exacta de números decimales
if (resultado != listResultados[nroCaso - 1])  // Puede fallar por precisión flotante

// ✅ SOLUCIÓN: Usar tolerancia para comparaciones numéricas
if (Mathf.Abs(resultado - listResultados[nroCaso - 1]) > 0.01f)
```

---

## 🟠 FALTAS DE ORTOGRAFÍA Y NOMENCLATURA

### 7. **Typos en Nombres de Variables y Comentarios**

| Archivo | Línea (aprox) | Texto Incorrecto | Corrección Sugerida |
|---------|--------------|-----------------|-------------------|
| `BeckInventory.cs` | Header | `indicacation` | `indicacion` |
| `FichaDiagnostico.cs` | Header | `indicacacion` | `indicacion` |
| `ControladorCinematica.cs` | Header | `Inidicaciones` | `Indicaciones` |
| `ControladorCinematica.cs` | Método | `darFuncionalidaBotonSC` | `darFuncionalidadBotonSaltarCinematica` |
| `DialogosManager.cs` | Campo | `dialagoPsicologo` | `dialogoPsicologo` |
| `DialogosManager.cs` | Campo | `dialagoPaciente` | `dialogoPaciente` |
| `DialogosManager.cs` | Campo | `txtDialogoPsiscologo` | `txtDialogoPsicologo` |
| `DialogosManager.cs` | Campo | `ubiSerntadoPsicologo` | `ubiSentadoPsicologo` |
| `DialogosManager.cs` | Campo | `auidoPuerta` | `audioPuerta` |
| `ApiManager.cs` | Comentarios | `dilogos`, `Dilogos` | `diálogos`, `Diálogos` |
| `ApiManager.cs` | Comentarios | `nmero` | `número` |
| `ColisionTutorial.cs` | Campo | `controleClick` | `controlClick` |

---

### 8. **Inconsistencia en Convenciones de Nomenclatura**

```csharp
// ❌ Mezcla de estilos en el mismo archivo (ApiManager.cs)
public List dialogosList;           // camelCase con mayúscula inicial (incorrecto)
private string apiUrl;              // camelCase (correcto para campos privados)
public int nroCaso;                 // PascalCase para campo público (correcto)

// ✅ ESTÁNDAR RECOMENDADO para C#/Unity:
// - Campos privados: _camelCase o camelCase
// - Propiedades públicas: PascalCase
// - Métodos: PascalCase
// - Variables locales: camelCase

// Ejemplo corregido:
private List<Dialogos> _dialogosList;  // privado con guion bajo
public int NroCaso { get; private set; }  // propiedad pública
```

---

## 🔵 PROBLEMAS DE ARQUITECTURA Y MANTENIBILIDAD

### 9. **Acoplamiento Excesivo entre Sistemas**
**Problema:** `DialogosManager.cs` conoce y manipula directamente:
- `FichaDiagnostico`, `BeckInventory`, `Calificacion`, `ManejadorCamara`, `SaveData`, `FinalizarCaso`

```csharp
// ❌ ALTO ACOPLAMIENTO: Difícil de testear y mantener
[SerializeField] private FichaDiagnostico fichaDiagnostico;
[SerializeField] private BeckInventory inventarioBeck;
[SerializeField] private Calificacion calificacion;
// ... 10+ referencias directas

// ✅ SOLUCIÓN: Patrón Observer/Eventos
// 1. Crear eventos para notificar cambios
public event Action<int> PreguntaRespondidaEvent;
public event Action<float> PuntuacionActualizadaEvent;

// 2. Los sistemas interesados se suscriben, no se referencian directamente
// 3. Usar interfaces para desacoplar
public interface IPuntuacionService {
    void Incrementar(double valor);
    event Action<float> OnPuntuacionCambiada;
}
```

---

### 10. **Magic Numbers y Strings Hardcodeados**
**Archivos afectados:** Múltiples

```csharp
// ❌ PROBLEMA: Valores mágicos dispersos
if (nroCaso == 1) { ... } else if (nroCaso == 4) { ... }  // ¿Por qué 1 y 4?
yield return new WaitForSeconds(35f / 500);  // ¿De dónde sale 35/500?

// ✅ SOLUCIÓN: Constantes o ScriptableObjects
// En un archivo Constants.cs:
public static class GameConstants {
    public const int CASO_DEPRESION_LEVE = 1;
    public const int CASO_ANSIEDAD_GENERALIZADA = 4;
    public const float TEXTO_VELOCIDAD_CARACTERES_POR_SEGUNDO = 500f / 35f;
}

// Uso:
if (nroCaso == GameConstants.CASO_DEPRESION_LEVE) { ... }
yield return new WaitForSeconds(1f / GameConstants.TEXTO_VELOCIDAD_CARACTERES_POR_SEGUNDO);
```

---

### 11. **Corrutinas sin Gestión de Cancelación**
**Archivo:** `DialogosManager.cs`, `BeckInventory.cs`

```csharp
// ❌ PROBLEMA: Corrutinas que continúan ejecutándose aunque el objeto se destruya
StartCoroutine(escribirTexto(texto, txt, btn));

// ✅ SOLUCIÓN: Usar CancellationToken o verificar this != null
IEnumerator escribirTexto(string texto, TextMeshProUGUI txt, GameObject btn)
{
    txt.maxVisibleCharacters = 0;
    txt.text = texto;
    
    for (int i = 0; i < texto.Length && this != null; i++)  // Verificar que el objeto existe
    {
        txt.maxVisibleCharacters++;
        yield return new WaitForSeconds(35f / 500);
    }
    // ...
}

// O mejor: usar UniTask (biblioteca externa) para cancelación robusta
```

---

## 🟢 MEJORAS DE CALIDAD DE CÓDIGO (No son errores, pero mejoran el proyecto)

### 12. **Falta de Validación de Entrada de Usuario**
**Archivo:** `BeckInventory.cs`, método `fnBtnEnviar()`

```csharp
// ❌ RIESGO: Parseo sin validación → excepción si el usuario escribe texto
resultado = int.Parse(resultText.text);  // Crash si resultText.text = "abc"

// ✅ SOLUCIÓN: Validar antes de parsear
if (int.TryParse(resultText.text, out resultado))
{
    // Proceder con la lógica
}
else
{
    txtError.text = "Por favor ingresa un número válido";
    panelAlerta.SetActive(true);
    yield break;
}
```

---

### 13. **Comentarios que Explican "Qué" en lugar de "Por Qué"**
**Ejemplo en múltiples archivos:**
```csharp
// ❌ Poco útil
void Start() {
    // Start is called before the first frame update
}

// ✅ Más valioso para investigación psicológica
void Start() {
    // Inicializamos la suscripción a eventos de API para cargar 
    // los diálogos del caso clínico asignado aleatoriamente.
    // Esto permite que diferentes participantes reciban casos 
    // distintos sin modificar el código, facilitando estudios 
    // con asignación aleatoria (Smith et al., 2023).
}
```

---

### 14. **Falta de Documentación XML para API Pública**
**Solución:** Añadir comentarios XML a métodos públicos:

```csharp
/// <summary>
/// Calcula el porcentaje de progreso del participante en la fase actual.
/// </summary>
/// <returns>Valor entre 0 y 100 representando el progreso porcentual.</returns>
/// <remarks>
/// Este cálculo se utiliza para métricas de investigación sobre 
/// tiempo de completitud de tareas cognitivas.
/// </remarks>
public double CalcularProgresoPorcentual()
{
    // ...
}
```

---

## 📊 RESUMEN DE PRIORIDADES PARA TU PROYECTO DE PSICOLOGÍA

### 🔴 Prioridad 1 (Corregir YA para que compile):
1. Agregar tipos genéricos a todas las `List<T>`
2. Corregir sintaxis de eventos `Action<T>`
3. Especificar tipos en todos los `GetComponent<T>()`

### 🟡 Prioridad 2 (Evitar bugs en investigación):
4. Implementar desuscripción de eventos en `OnDestroy()`
5. Corregir cálculos de porcentaje con división flotante
6. Añadir validación de entrada de usuario (`int.TryParse`)

### 🟠 Prioridad 3 (Mejorar mantenibilidad para tu tesis):
7. Corregir todos los typos en nombres (facilita búsqueda y documentación)
8. Extraer "magic numbers" a constantes documentadas
9. Añadir comentarios que expliquen el "por qué" metodológico

### 🟢 Prioridad 4 (Excelencia académica):
10. Documentar métodos públicos con XML para generación automática de docs
11. Implementar patrón Observer para desacoplar sistemas
12. Añadir pruebas unitarias para lógica de cálculo de puntuaciones

---
