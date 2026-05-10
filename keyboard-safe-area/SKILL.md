---
name: keyboard-safe-area
description: Reglas y patrones probados para usar SafeAreaView, KeyboardAvoidingView y ScrollView en React Native + Expo, garantizando que el teclado funcione correctamente en iOS y Android sin offsets hardcodeados ni hacks por dispositivo. Aplica cuando una screen tiene inputs, botones fijos, headers personalizados, steppers o cuando reportan que "el botón se corta", "el teclado tapa el input", "no se puede hacer scroll" o "se ve diferente entre iOS y Android".
license: MIT
metadata:
  tags: react-native, expo, keyboard, safe-area, scrollview, ios, android
---

# Keyboard, SafeAreaView and ScrollView — Production Rules

## Overview

Documenta los patrones correctos para manejar teclado en React Native + Expo, junto con los **bugs sutiles** que aparecen al combinar `SafeAreaView`, `KeyboardAvoidingView`, `ScrollView`, headers de navegación, animation wrappers, `Pressable` y formularios multi-step.

La regla más importante: **el teclado no se "arregla" con `keyboardVerticalOffset`** — se arregla diseñando la jerarquía correcta y aceptando que iOS y Android necesitan `behavior` distintos.

---

## Quick Rules (TL;DR)

1. **Importar `SafeAreaView` desde `react-native-safe-area-context`** — nunca desde `react-native`.
2. **Jerarquía base:** `SafeAreaView > KeyboardAvoidingView > ScrollView/FlatList > Content`. El KAV NO va dentro del ScrollView.
3. **`behavior` por plataforma:** `Platform.OS === "ios" ? "padding" : undefined`. **NUNCA `"height"` en Android** — choca con `adjustResize` nativo.
4. **NO uses el header del Drawer/Stack/Tabs de Expo Router cuando la screen tiene inputs.** El KAV no lo "ve" -> cálculos incorrectos -> contenido cortado. Usa `headerShown: false` y mete un header custom DENTRO del SafeAreaView.
5. **NO uses `keyboardVerticalOffset` con valores fijos.** Si por alguna razón debes mantener el header del navigator, usa `useHeaderHeight()` de React Navigation. Pero la solución preferida es la regla 4.
6. **Configura Android:** `softwareKeyboardLayoutMode: "resize"` en `app.config.js` — es el default en Expo.
7. **`ScrollView`:** `style={{ flex: 1 }}` + `contentContainerStyle={{ flexGrow: 1 }}` (NO `flex: 1` adentro). `keyboardShouldPersistTaps="handled"` siempre.
8. **No envuelvas el ScrollView en `Pressable onPress={Keyboard.dismiss}`** — bloquea el scroll. Usa `keyboardDismissMode="on-drag"` en el ScrollView.
9. **No anides `KeyboardAvoidingView`s** — solo uno por screen, lo más arriba posible.
10. **Edges del `SafeAreaView`:** si la screen tiene un header custom interno que ya maneja `edges={["top"]}`, usa `edges={["left","right","bottom"]}` en el padre para evitar duplicar el inset top.
11. **`SafeAreaView` con `edges={["bottom"]}` + `paddingBottom` da espacio variable según teclado** — el bottom inset se vuelve 0 con teclado abierto. Para padding fijo siempre, usa un `View` simple.

---

## Decision Tree

```text
¿La screen tiene inputs (TextInput)?
├── SÍ
│   ├── ¿Tiene un botón fijo abajo?
│   │   ├── SÍ -> Patrón A (KAV con ScrollView + View footer)
│   │   └── NO -> Patrón B (KAV con ScrollView, botón dentro)
│   └── ¿Tiene una FlatList grande?
│       └── SÍ -> Patrón C (KAV con FlatList directa)
└── NO
    ├── ¿Solo muestra contenido + botón?
    │   └── Patrón D (sin KAV, solo SafeAreaView + ScrollView)
    └── ¿Es un screen de Drawer/Stack con header?
        └── Patrón E (header dentro de SafeAreaView, NO usar header del navigator)
```

---

## Patrones canónicos

### Patrón A — Form con botón fijo abajo

```tsx
import { KeyboardAvoidingView, Platform, ScrollView, View, StyleSheet } from "react-native";
import { SafeAreaView } from "react-native-safe-area-context";

export const FormScreen = () => (
  <SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
    <KeyboardAvoidingView
      style={{ flex: 1 }}
      behavior={Platform.OS === "ios" ? "padding" : undefined}
    >
      <ScrollView
        style={{ flex: 1 }}
        contentContainerStyle={{ flexGrow: 1, padding: 16 }}
        keyboardShouldPersistTaps="handled"
        keyboardDismissMode="on-drag"
        showsVerticalScrollIndicator={false}
      >
        {/* inputs */}
      </ScrollView>

      <View style={styles.footer}>
        {/* botón "Save" / "Next" */}
      </View>
    </KeyboardAvoidingView>
  </SafeAreaView>
);

const styles = StyleSheet.create({
  screen: { flex: 1, backgroundColor: "#fff" },
  footer: {
    paddingHorizontal: 16,
    paddingTop: 8,
    paddingBottom: 8,
  },
});
```

**Notas críticas:**
- `behavior={undefined}` en Android es lo correcto; deja que el sistema (`adjustResize`) maneje el teclado. NO uses `"height"` ahí.
- El `View` del footer NO es un `SafeAreaView edges={["bottom"]}` — el bottom inset ya lo aplica el SafeAreaView padre.
- Un `paddingBottom` fijo (aprox. 8pt) asegura espacio mínimo entre el botón y el teclado/borde sin depender de safe insets.

### Patrón B — Form con botón dentro del scroll

Cuando el contenido es corto y el botón naturalmente queda al final.

```tsx
<SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
  <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
    <ScrollView
      style={{ flex: 1 }}
      contentContainerStyle={{ flexGrow: 1, padding: 16, paddingBottom: 24 }}
      keyboardShouldPersistTaps="handled"
    >
      {/* inputs */}
      {/* botón inline al final */}
    </ScrollView>
  </KeyboardAvoidingView>
</SafeAreaView>
```

### Patrón C — FlatList con KAV

Para chats (input + lista de mensajes) o listas largas con un botón fijo.

```tsx
<SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
  <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
    <FlatList
      data={items}
      keyExtractor={(item) => item.id}
      renderItem={({ item }) => <Item {...item} />}
      style={{ flex: 1 }}
      contentContainerStyle={{ padding: 16 }}
      keyboardShouldPersistTaps="handled"
      keyboardDismissMode="on-drag"
    />
    <View style={styles.footer}>
      {/* botón fijo o input chat */}
    </View>
  </KeyboardAvoidingView>
</SafeAreaView>
```

### Patrón D — Sin KAV (no inputs)

Si la screen no tiene `TextInput`, **no agregues `KeyboardAvoidingView`** — es ruido.

```tsx
<SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
  <ScrollView style={{ flex: 1 }} contentContainerStyle={{ flexGrow: 1, padding: 16 }}>
    {/* contenido */}
  </ScrollView>
</SafeAreaView>
```

### Patrón E — Drawer/Stack screen con header custom interno

**Crítico:** si la screen está dentro de un Drawer/Stack con header, NO uses el header del navigator. Mete el header DENTRO del SafeAreaView. Esto evita el bug donde `KeyboardAvoidingView` no compensa el header height y el botón queda tapado.

```tsx
// CustomHeader es un componente que maneja edges={["top"]} internamente
// (back button, título, etc.)

export const Screen = () => (
  <>
    <Drawer.Screen options={{ headerShown: false }} />
    <SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
      <CustomHeader title="Title" />

      <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
        <ScrollView style={{ flex: 1 }} contentContainerStyle={{ flexGrow: 1, padding: 16 }} keyboardShouldPersistTaps="handled">
          {/* contenido */}
        </ScrollView>
        <View style={styles.footer}>{/* botón */}</View>
      </KeyboardAvoidingView>
    </SafeAreaView>
  </>
);
```

**Por qué funciona:**
- El header vive dentro del SafeAreaView, así que el KAV mide su posición empezando justo debajo del header.
- No se necesita `keyboardVerticalOffset` ni `useHeaderHeight()`.
- `edges={["left","right","bottom"]}` evita duplicar el inset top (el header custom ya tiene `edges={["top"]}` internamente).

#### Ejemplo de un CustomHeader reutilizable

```tsx
import { SafeAreaView } from "react-native-safe-area-context";
import { Pressable, Text, View, StyleSheet } from "react-native";
import { useRouter } from "expo-router";

export const CustomHeader = ({ title }: { title: string }) => {
  const router = useRouter();
  return (
    <SafeAreaView edges={["top"]} style={styles.safeArea}>
      <View style={styles.row}>
        <Pressable onPress={() => router.back()} style={styles.backBtn}>
          {/* icon back */}
        </Pressable>
        <Text style={styles.title}>{title}</Text>
      </View>
    </SafeAreaView>
  );
};

const styles = StyleSheet.create({
  safeArea: { backgroundColor: "#fff" },
  row: { flexDirection: "row", alignItems: "center", paddingHorizontal: 16, paddingVertical: 8 },
  backBtn: { padding: 8 },
  title: { fontSize: 16, fontWeight: "600", flex: 1 },
});
```

---

## Common Bugs y troubleshooting

### Bug 1: "El botón se corta cuando abro el teclado" (header de Expo Router)

**El bug más común y el más difícil de diagnosticar.** Si estás usando el header que ofrece Expo Router (Drawer header, Stack header, Tabs header) en una screen con inputs, casi siempre vas a tener problemas con el teclado.

**Síntomas:**
- El botón fijo abajo queda cortado o tapado al abrir el teclado.
- El padding entre el botón y el teclado desaparece.
- El contenido se ve "comprimido" o "saltado" cuando el teclado abre.
- Funciona perfecto en una screen sin header, falla cuando hay header.
- El bug es peor en Android.

**Por qué pasa:**
- El header del Drawer/Stack/Tabs se renderiza FUERA del árbol de tu screen — es parte del navigator, no de tu componente.
- Tu `KeyboardAvoidingView` mide su posición desde el top absoluto de la window, sin saber que el header le quita 50-110pt arriba.
- Cuando el teclado abre, el KAV calcula `screen height - keyboard height = espacio disponible`, pero el espacio REAL disponible es `screen height - header height - keyboard height`. Falta restar el header.
- Resultado: el KAV cree que tiene más espacio del que realmente tiene -> no sube el contenido lo suficiente -> el botón queda tapado.

**Fix correcto (preferido):** Patrón E — mover el header DENTRO del SafeAreaView de tu screen.

```tsx
// ❌ MAL — header del navigator
<Drawer.Screen options={{ headerShown: true, title: "Edit" }} />
<SafeAreaView>
  <KeyboardAvoidingView>
    <ScrollView>...</ScrollView>
    <View style={styles.footer}><Button /></View>  {/* se corta con keyboard */}
  </KeyboardAvoidingView>
</SafeAreaView>

// ✅ BIEN — header inside
<Drawer.Screen options={{ headerShown: false }} />
<SafeAreaView edges={["left", "right", "bottom"]}>
  <CustomHeader title="Edit" />  {/* dentro del SafeAreaView */}
  <KeyboardAvoidingView>
    <ScrollView>...</ScrollView>
    <View style={styles.footer}><Button /></View>  {/* funciona perfecto */}
  </KeyboardAvoidingView>
</SafeAreaView>
```

Ahora el KAV mide su posición empezando justo debajo del header (porque el header es parte del mismo árbol), y el cálculo es correcto sin necesidad de offset.

**Fix alternativo (peor, pero válido):** `keyboardVerticalOffset={useHeaderHeight()}` de `@react-navigation/elements`. Es dinámico (no hardcoded) y compensa el header. Pero suma complejidad y NO ayuda en Android porque allí el conflicto es con `adjustResize`, no con el offset.

**NUNCA hagas:** `keyboardVerticalOffset={Platform.OS === "ios" ? 80 : 0}`. Es un valor mágico que falla en otros dispositivos (notch grande/chico, large title, header con subtítulo, status bar grande).

### Bug 2: "El padding del footer se 'come' cuando abro el teclado"

**Diagnóstico:**
- Tienes `<SafeAreaView edges={["bottom"]} style={styles.footer}>` envolviendo el botón.
- Sin teclado: el bottom safe inset (aprox. 34pt en iPhones modernos) + `paddingBottom` da mucho espacio.
- Con teclado: el bottom safe inset se vuelve 0 (iOS lo reporta así porque el teclado ocupa esa zona).
- Resultado: el espacio visible debajo del botón se reduce drásticamente.

**Fix:** mover el `bottom` al `SafeAreaView` padre (`edges={["left","right","bottom"]}`) y usar un `<View>` simple para el footer con un `paddingBottom` fijo. Así el padding es constante.

### Bug 3: "En iOS funciona pero en Android el botón se rompe"

**Diagnóstico:** estás usando `behavior="height"` en Android.

**Por qué pasa:**
- Android tiene `softwareKeyboardLayoutMode: "resize"` (default Expo) — el sistema YA redimensiona la window al abrir el teclado.
- `behavior="height"` hace que el KAV TAMBIÉN reduzca su altura. Doble cálculo -> contenido comprimido.

**Fix:** `behavior={Platform.OS === "ios" ? "padding" : undefined}`. Con `undefined`, el KAV se vuelve un `<View>` pasivo en Android y deja que el sistema maneje todo.

### Bug 4: "No puedo hacer scroll dentro del form"

**Diagnóstico:** envolviste el ScrollView en `<Pressable onPress={Keyboard.dismiss}>` o `<TouchableWithoutFeedback>`.

**Por qué pasa:** estos componentes registran un gesture handler que captura los toques verticales antes de que lleguen al ScrollView.

**Fix:**
1. Quitar el `Pressable`/`TouchableWithoutFeedback`.
2. Agregar `keyboardDismissMode="on-drag"` al ScrollView. Cuando el usuario hace swipe vertical, el teclado se cierra solo.

### Bug 5: "Doble safe area top — hay un gap arriba"

**Diagnóstico:** tienes `<SafeAreaView>` (default edges = top/right/bottom/left) envolviendo otro componente que ya aplica `edges={["top"]}` internamente.

**Fix:** especificar edges en el padre sin `"top"`:
```tsx
<SafeAreaView edges={["left", "right", "bottom"]}>
  <CustomHeader />  {/* ya maneja top */}
</SafeAreaView>
```

### Bug 6: "El KAV no funciona porque está anidado"

**Diagnóstico:** tienes un KAV global en el padre y otro KAV en cada step/sub-componente.

**Por qué pasa:** los KAVs internos miden desde su propia posición (que ya está afectada por el KAV padre). Cálculos incorrectos.

**Fix:** UN SOLO KAV por screen, lo más arriba posible. Si la screen es un stepper, el KAV va en el padre, no en cada step.

### Bug 7: "Animation wrappers (Reanimated, FadeIn) rompen el ScrollView"

**Diagnóstico:** el ScrollView está dentro de `<Animated.View>` o un wrapper de animación que aplica transforms.

**Fix:** usar el animation wrapper en HIJOS del ScrollView, no como wrapper. O usar Reanimated v3+ con `Animated.ScrollView` directamente.

---

## Patrón especial — Stepper con header global (back + progress bar)

Para flujos multi-step (onboarding, creación de recurso paso-a-paso).

```tsx
const TOTAL_STEPS = 5;
const progressAnim = useRef(new Animated.Value(0)).current;

useEffect(() => {
  Animated.timing(progressAnim, {
    toValue: Math.min((step + 1) / TOTAL_STEPS, 1),
    duration: 250,
    easing: Easing.out(Easing.ease),
    useNativeDriver: false,
  }).start();
}, [step]);

return (
  <SafeAreaView style={styles.screen} edges={["top", "left", "right", "bottom"]}>
    <Drawer.Screen options={{ headerShown: false }} />

    {/* Header con back + progress bar */}
    {step < TOTAL_STEPS && (
      <View style={styles.headerRow}>
        <View style={styles.headerSide}>
          <TouchableOpacity onPress={step > 0 ? goBack : goOut} style={styles.backButton}>
            {/* icon back */}
          </TouchableOpacity>
        </View>
        <View style={styles.headerCenter}>
          <View style={styles.progressTrack}>
            <Animated.View
              style={[styles.progressFill, {
                width: progressAnim.interpolate({ inputRange: [0, 1], outputRange: ["0%", "100%"] }),
              }]}
            />
          </View>
        </View>
        <View style={styles.headerSide} />
      </View>
    )}

    {/* KAV global, NUNCA en cada step */}
    <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
      {/* ScrollView global, NO uno por step */}
      <ScrollView
        style={{ flex: 1 }}
        contentContainerStyle={{ flexGrow: 1 }}
        keyboardShouldPersistTaps="handled"
        keyboardDismissMode="on-drag"
      >
        {step === 0 && <Step1 />}
        {step === 1 && <Step2 />}
        {/* etc — cada Step solo retorna un <View>, sin su propio ScrollView ni KAV */}
      </ScrollView>

      <View style={styles.footer}>
        {/* botón "Next" */}
      </View>
    </KeyboardAvoidingView>
  </SafeAreaView>
);
```

**Reglas del stepper:**
1. UN solo KAV en el padre. Los steps son `<View>` plain.
2. UN solo ScrollView en el padre. Los steps no tienen su propio scroll.
3. Render condicional `{step === N && <StepX />}` para mostrar el step activo.
4. NO uses `<Pressable onPress={Keyboard.dismiss}>` envolviendo los steps. Usa `keyboardDismissMode="on-drag"` en el ScrollView.

---

## Platform behavior reference

| Comportamiento iOS | Comportamiento Android |
|--------------------|------------------------|
| Sistema NO maneja el teclado nativamente. | `adjustResize` redimensiona la window al abrir el teclado. |
| KAV con `behavior="padding"` agrega padding-bottom = altura del teclado. | KAV con `behavior=undefined` no hace nada; deja al sistema. |
| `bottom` safe inset se vuelve 0 con teclado abierto. | No hay safe inset bottom (excepto navigation bar). |
| `useHeaderHeight()` reporta el alto del header del navigator. | Igual, pero menos crítico porque `adjustResize` lo compensa. |

### Configuración Android necesaria (app.config.js)

```ts
{
  expo: {
    android: {
      softwareKeyboardLayoutMode: "resize",  // default, asegúrate de no cambiarlo a "pan"
    },
  },
}
```

---

## Anti-patterns (NO hacer)

```tsx
// ❌ SafeAreaView de react-native (deprecado, no respeta safe insets dinámicos)
import { SafeAreaView } from "react-native";

// ❌ behavior="height" en Android (conflicto con adjustResize)
behavior={Platform.OS === "ios" ? "padding" : "height"}

// ❌ keyboardVerticalOffset hardcoded
keyboardVerticalOffset={Platform.OS === "ios" ? 64 : 0}

// ❌ ScrollView envolviendo KAV
<ScrollView>
  <KeyboardAvoidingView>...</KeyboardAvoidingView>
</ScrollView>

// ❌ KAV anidados (uno en parent, otro en child)
<KeyboardAvoidingView>
  <SomeChildScreen>
    <KeyboardAvoidingView>...</KeyboardAvoidingView>
  </SomeChildScreen>
</KeyboardAvoidingView>

// ❌ Pressable con onPress envolviendo el ScrollView (rompe scroll)
<Pressable onPress={Keyboard.dismiss}>
  <ScrollView>...</ScrollView>
</Pressable>

// ❌ contentContainerStyle con flex: 1 (no permite scroll)
<ScrollView contentContainerStyle={{ flex: 1 }}>

// ❌ height: '100%' dentro de contenido scrollable
<View style={{ height: '100%' }}>

// ❌ SafeAreaView edges duplicados con un header interno que ya maneja top
<SafeAreaView>  {/* default = top/right/bottom/left */}
  <CustomHeader />  {/* ya tiene edges=["top"] internamente */}
</SafeAreaView>

// ❌ Drawer.Screen / Stack.Screen con headerShown: true cuando hay inputs en la screen
// (causa que el botón se corte con el teclado)
<Drawer.Screen options={{ headerShown: true, title: "..." }} />
```

---

## Checklist al crear/refactorizar una screen con teclado

- [ ] `SafeAreaView` viene de `react-native-safe-area-context`.
- [ ] Jerarquía: `SafeAreaView > KAV > ScrollView/FlatList > content`.
- [ ] `behavior={Platform.OS === "ios" ? "padding" : undefined}` (NUNCA `"height"`).
- [ ] No hay `keyboardVerticalOffset` con valor numérico fijo.
- [ ] Si hay header: usa un header custom DENTRO del SafeAreaView, con `headerShown: false` en el navigator.
- [ ] `ScrollView` con `style={{flex:1}}` + `contentContainerStyle={{flexGrow:1}}` + `keyboardShouldPersistTaps="handled"`.
- [ ] Si quieres dismiss del teclado al hacer scroll: `keyboardDismissMode="on-drag"`.
- [ ] No hay `Pressable`/`TouchableWithoutFeedback` envolviendo el ScrollView.
- [ ] Solo UN `KeyboardAvoidingView` por screen, lo más arriba posible.
- [ ] Si la screen es un stepper: scroll y KAV viven en el parent, no en cada step.
- [ ] Edges del SafeAreaView ajustados al contexto (`["left","right","bottom"]` cuando el header interno maneja `top`).
- [ ] Footer con botón fijo: `<View>` plain con `paddingHorizontal`, `paddingTop`, `paddingBottom` fijos. No SafeAreaView interno.
- [ ] Probado en iPhone con home indicator (espacio bottom respetado) y Android (botón no tapado por teclado).

---

## Quick Reference: footer style

Patrón estándar para un botón fijo abajo:

```ts
footer: {
  paddingHorizontal: 16,
  paddingTop: 8,
  paddingBottom: 8,
}
```

Si el `SafeAreaView` padre tiene `edges={["bottom"]}`, el padding bottom se suma al inset (más espacio sin teclado, menos con teclado). Si el padre NO tiene `bottom` en sus edges, el `paddingBottom` es constante (recomendado para evitar el bug 2).
