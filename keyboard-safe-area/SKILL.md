---
name: keyboard-safe-area
description: Proven rules and patterns for using SafeAreaView, KeyboardAvoidingView, and ScrollView in React Native + Expo so the keyboard behaves correctly on iOS and Android without hardcoded offsets or device-specific hacks. Use this skill when a screen has inputs, fixed buttons, custom headers, steppers, or reports like "the button gets cut off", "the keyboard covers the input", "scroll does not work", or "it looks different on iOS and Android".
license: MIT
metadata:
  tags: react-native, expo, keyboard, safe-area, scrollview, ios, android
---

# Keyboard, SafeAreaView and ScrollView — Production Rules

## Overview

This skill documents reliable patterns for handling the keyboard in React Native + Expo, including the subtle bugs that appear when combining `SafeAreaView`, `KeyboardAvoidingView`, `ScrollView`, navigation headers, animation wrappers, `Pressable`, and multi-step forms.

The most important rule: **the keyboard is not "fixed" with `keyboardVerticalOffset`**. It is fixed by designing the right hierarchy and accepting that iOS and Android need different `behavior` values.

---

## Quick Rules (TL;DR)

1. **Import `SafeAreaView` from `react-native-safe-area-context`** — never from `react-native`.
2. **Base hierarchy:** `SafeAreaView > KeyboardAvoidingView > ScrollView/FlatList > Content`. The KAV does NOT go inside the `ScrollView`.
3. **Platform-specific `behavior`:** `Platform.OS === "ios" ? "padding" : undefined`. **NEVER use `"height"` on Android** because it conflicts with native `adjustResize`.
4. **Do not use the Expo Router Drawer/Stack/Tabs header when the screen has inputs.** The KAV cannot "see" that header, which causes incorrect calculations and clipped content. Use `headerShown: false` and place a custom header INSIDE the `SafeAreaView`.
5. **Do not use `keyboardVerticalOffset` with fixed values.** If you must keep the navigator header, use React Navigation's `useHeaderHeight()`. The preferred solution is rule 4.
6. **Configure Android:** `softwareKeyboardLayoutMode: "resize"` in `app.config.js`. This is the Expo default.
7. **`ScrollView`:** `style={{ flex: 1 }}` + `contentContainerStyle={{ flexGrow: 1 }}`. Do NOT use `flex: 1` inside `contentContainerStyle`. Always set `keyboardShouldPersistTaps="handled"`.
8. **Do not wrap `ScrollView` with `Pressable onPress={Keyboard.dismiss}`** because it can block scrolling. If the product explicitly wants drag-to-dismiss behavior, add `keyboardDismissMode="on-drag"` to the `ScrollView`; otherwise omit it.
9. **Do not nest `KeyboardAvoidingView`s.** Use one per screen, as high in the screen tree as possible.
10. **Safe-area `edges`:** if the screen has an internal custom header that already handles `edges={["top"]}`, use `edges={["left","right","bottom"]}` on the parent to avoid duplicating the top inset.
11. **`SafeAreaView` with `edges={["bottom"]}` + `paddingBottom` creates variable spacing when the keyboard opens** because the bottom inset becomes 0. For fixed padding, use a plain `View`.

---

## Decision Tree

```text
Does the screen have inputs (TextInput)?
├── YES
│   ├── Does it have a fixed bottom button?
│   │   ├── YES -> Pattern A (KAV with ScrollView + View footer)
│   │   └── NO -> Pattern B (KAV with ScrollView, button inside)
│   └── Does it have a large FlatList?
│       └── YES -> Pattern C (KAV with direct FlatList)
└── NO
    ├── Does it only show content + a button?
    │   └── Pattern D (no KAV, only SafeAreaView + ScrollView)
    └── Is it a Drawer/Stack screen with a header?
        └── Pattern E (header inside SafeAreaView, do NOT use navigator header)
```

---

## Canonical Patterns

### Pattern A — Form with Fixed Bottom Button

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
        showsVerticalScrollIndicator={false}
      >
        {/* inputs */}
      </ScrollView>

      <View style={styles.footer}>
        {/* "Save" / "Next" button */}
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

**Critical notes:**
- `behavior={undefined}` is correct on Android because it lets the system (`adjustResize`) handle the keyboard. Do NOT use `"height"` there.
- The footer `View` is NOT a `SafeAreaView edges={["bottom"]}`. The bottom inset is already applied by the parent `SafeAreaView`.
- A fixed `paddingBottom` (about 8pt) guarantees minimum space between the button and the keyboard/edge without depending on safe-area insets.

### Pattern B — Form with Button Inside the Scroll

Use this when the content is short and the button naturally belongs at the end.

```tsx
<SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
  <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
    <ScrollView
      style={{ flex: 1 }}
      contentContainerStyle={{ flexGrow: 1, padding: 16, paddingBottom: 24 }}
      keyboardShouldPersistTaps="handled"
    >
      {/* inputs */}
      {/* inline button at the end */}
    </ScrollView>
  </KeyboardAvoidingView>
</SafeAreaView>
```

### Pattern C — FlatList with KAV

Use this for chats (input + message list) or long lists with a fixed button.

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
    />
    <View style={styles.footer}>
      {/* fixed button or chat input */}
    </View>
  </KeyboardAvoidingView>
</SafeAreaView>
```

### Pattern D — No KAV (No Inputs)

If the screen has no `TextInput`, **do not add `KeyboardAvoidingView`**. It is noise.

```tsx
<SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
  <ScrollView style={{ flex: 1 }} contentContainerStyle={{ flexGrow: 1, padding: 16 }}>
    {/* content */}
  </ScrollView>
</SafeAreaView>
```

### Pattern E — Drawer/Stack Screen with Internal Custom Header

**Critical:** if the screen is inside a Drawer/Stack with a header, do NOT use the navigator header. Put the header INSIDE the `SafeAreaView`. This avoids the bug where `KeyboardAvoidingView` does not compensate for header height and the button gets covered.

```tsx
// CustomHeader is a component that handles edges={["top"]} internally
// (back button, title, etc.)

export const Screen = () => (
  <>
    <Drawer.Screen options={{ headerShown: false }} />
    <SafeAreaView style={styles.screen} edges={["left", "right", "bottom"]}>
      <CustomHeader title="Title" />

      <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
        <ScrollView style={{ flex: 1 }} contentContainerStyle={{ flexGrow: 1, padding: 16 }} keyboardShouldPersistTaps="handled">
          {/* content */}
        </ScrollView>
        <View style={styles.footer}>{/* button */}</View>
      </KeyboardAvoidingView>
    </SafeAreaView>
  </>
);
```

**Why this works:**
- The header lives inside the `SafeAreaView`, so the KAV measures its position starting right below the header.
- No `keyboardVerticalOffset` or `useHeaderHeight()` is needed.
- `edges={["left","right","bottom"]}` avoids duplicating the top inset because the custom header already handles `edges={["top"]}` internally.

#### Reusable CustomHeader Example

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
          {/* back icon */}
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

## Common Bugs and Troubleshooting

### Bug 1: "The button gets cut off when I open the keyboard" (Expo Router Header)

**This is the most common bug and the hardest one to diagnose.** If you are using the header provided by Expo Router (Drawer header, Stack header, Tabs header) on a screen with inputs, you will almost always have keyboard issues.

**Symptoms:**
- The fixed bottom button gets clipped or covered when the keyboard opens.
- The spacing between the button and keyboard disappears.
- The content looks compressed or jumps when the keyboard opens.
- It works perfectly on a screen without a header, then fails when a header is present.
- The bug is worse on Android.

**Why it happens:**
- The Drawer/Stack/Tabs header renders OUTSIDE your screen tree. It belongs to the navigator, not your component.
- Your `KeyboardAvoidingView` measures its position from the absolute top of the window, without knowing that the header consumes 50-110pt above it.
- When the keyboard opens, KAV calculates `screen height - keyboard height = available space`, but the REAL available space is `screen height - header height - keyboard height`. The header is missing from the calculation.
- Result: KAV thinks it has more space than it really has, so it does not move content enough and the button gets covered.

**Correct fix (preferred):** Pattern E — move the header INSIDE the screen's `SafeAreaView`.

```tsx
// ❌ BAD — navigator header
<Drawer.Screen options={{ headerShown: true, title: "Edit" }} />
<SafeAreaView>
  <KeyboardAvoidingView>
    <ScrollView>...</ScrollView>
    <View style={styles.footer}><Button /></View>  {/* gets cut off with keyboard */}
  </KeyboardAvoidingView>
</SafeAreaView>

// ✅ GOOD — header inside
<Drawer.Screen options={{ headerShown: false }} />
<SafeAreaView edges={["left", "right", "bottom"]}>
  <CustomHeader title="Edit" />  {/* inside SafeAreaView */}
  <KeyboardAvoidingView>
    <ScrollView>...</ScrollView>
    <View style={styles.footer}><Button /></View>  {/* works correctly */}
  </KeyboardAvoidingView>
</SafeAreaView>
```

Now the KAV measures its position starting right below the header because the header is part of the same tree, and the calculation is correct without an offset.

**Alternative fix (worse, but valid):** `keyboardVerticalOffset={useHeaderHeight()}` from `@react-navigation/elements`. It is dynamic, not hardcoded, and compensates for the header. But it adds complexity and does NOT help much on Android because the conflict there is with `adjustResize`, not with the offset.

**NEVER do this:** `keyboardVerticalOffset={Platform.OS === "ios" ? 80 : 0}`. It is a magic number that fails on other devices (large/small notch, large title, header with subtitle, tall status bar).

### Bug 2: "Footer padding disappears when I open the keyboard"

**Diagnosis:**
- You have `<SafeAreaView edges={["bottom"]} style={styles.footer}>` wrapping the button.
- Without the keyboard: bottom safe inset (about 34pt on modern iPhones) + `paddingBottom` creates generous spacing.
- With the keyboard: bottom safe inset becomes 0 because iOS reports that the keyboard occupies that area.
- Result: the visible space below the button shrinks dramatically.

**Fix:** move the `bottom` edge to the parent `SafeAreaView` (`edges={["left","right","bottom"]}`) and use a plain `<View>` for the footer with fixed `paddingBottom`. This keeps padding constant.

### Bug 3: "It works on iOS but breaks the button on Android"

**Diagnosis:** you are using `behavior="height"` on Android.

**Why it happens:**
- Android has `softwareKeyboardLayoutMode: "resize"` (Expo default), so the system ALREADY resizes the window when the keyboard opens.
- `behavior="height"` makes KAV ALSO reduce its height. Double calculation -> compressed content.

**Fix:** `behavior={Platform.OS === "ios" ? "padding" : undefined}`. With `undefined`, KAV becomes a passive `<View>` on Android and lets the system handle everything.

### Bug 4: "I cannot scroll inside the form"

**Diagnosis:** you wrapped the `ScrollView` in `<Pressable onPress={Keyboard.dismiss}>` or `<TouchableWithoutFeedback>`.

**Why it happens:** these components register a gesture handler that captures vertical touches before they reach the `ScrollView`.

**Fix:**
1. Remove the `Pressable`/`TouchableWithoutFeedback`.
2. Keep the `ScrollView` in charge of vertical gestures.
3. Add `keyboardDismissMode="on-drag"` only if the intended UX is to dismiss the keyboard while scrolling.

### Bug 5: "Double top safe area — there is a gap at the top"

**Diagnosis:** you have `<SafeAreaView>` (default edges = top/right/bottom/left) wrapping another component that already applies `edges={["top"]}` internally.

**Fix:** specify parent edges without `"top"`:
```tsx
<SafeAreaView edges={["left", "right", "bottom"]}>
  <CustomHeader />  {/* already handles top */}
</SafeAreaView>
```

### Bug 6: "KAV does not work because it is nested"

**Diagnosis:** you have a global KAV in the parent and another KAV in each step/subcomponent.

**Why it happens:** inner KAVs measure from their own position, which is already affected by the parent KAV. This produces incorrect calculations.

**Fix:** use ONE KAV per screen, as high as possible. If the screen is a stepper, the KAV belongs in the parent, not in each step.

### Bug 7: "Animation wrappers (Reanimated, FadeIn) break the ScrollView"

**Diagnosis:** the `ScrollView` is inside an `<Animated.View>` or animation wrapper that applies transforms.

**Fix:** use animation wrappers on CHILDREN of the `ScrollView`, not around the `ScrollView`. Or use Reanimated v3+ with `Animated.ScrollView` directly.

---

## Special Pattern — Stepper with Global Header (Back + Progress Bar)

Use this for multi-step flows such as onboarding or step-by-step resource creation.

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

    {/* Header with back button + progress bar */}
    {step < TOTAL_STEPS && (
      <View style={styles.headerRow}>
        <View style={styles.headerSide}>
          <TouchableOpacity onPress={step > 0 ? goBack : goOut} style={styles.backButton}>
            {/* back icon */}
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

    {/* Global KAV, NEVER one per step */}
    <KeyboardAvoidingView style={{ flex: 1 }} behavior={Platform.OS === "ios" ? "padding" : undefined}>
      {/* Global ScrollView, NOT one per step */}
      <ScrollView
        style={{ flex: 1 }}
        contentContainerStyle={{ flexGrow: 1 }}
        keyboardShouldPersistTaps="handled"
      >
        {step === 0 && <Step1 />}
        {step === 1 && <Step2 />}
        {/* etc — each Step only returns a <View>, with no local ScrollView or KAV */}
      </ScrollView>

      <View style={styles.footer}>
        {/* "Next" button */}
      </View>
    </KeyboardAvoidingView>
  </SafeAreaView>
);
```

**Stepper rules:**
1. ONE KAV in the parent. Steps are plain `<View>` components.
2. ONE `ScrollView` in the parent. Steps do not have their own scroll container.
3. Use conditional rendering (`{step === N && <StepX />}`) to show the active step.
4. Do NOT wrap steps with `<Pressable onPress={Keyboard.dismiss}>`. Add `keyboardDismissMode="on-drag"` only when drag-to-dismiss is desired.

---

## Platform Behavior Reference

| iOS behavior | Android behavior |
|--------------|------------------|
| The system does NOT handle the keyboard natively. | `adjustResize` resizes the window when the keyboard opens. |
| KAV with `behavior="padding"` adds bottom padding equal to keyboard height. | KAV with `behavior=undefined` does nothing and lets the system work. |
| Bottom safe inset becomes 0 when the keyboard is open. | There is usually no bottom safe inset except the navigation bar. |
| `useHeaderHeight()` reports the navigator header height. | Same, but less critical because `adjustResize` compensates more. |

### Required Android Config (app.config.js)

```ts
{
  expo: {
    android: {
      softwareKeyboardLayoutMode: "resize",  // default; make sure it was not changed to "pan"
    },
  },
}
```

---

## Anti-Patterns (Do Not Do This)

```tsx
// ❌ SafeAreaView from react-native (deprecated, does not respect dynamic safe insets)
import { SafeAreaView } from "react-native";

// ❌ behavior="height" on Android (conflicts with adjustResize)
behavior={Platform.OS === "ios" ? "padding" : "height"}

// ❌ hardcoded keyboardVerticalOffset
keyboardVerticalOffset={Platform.OS === "ios" ? 64 : 0}

// ❌ ScrollView wrapping KAV
<ScrollView>
  <KeyboardAvoidingView>...</KeyboardAvoidingView>
</ScrollView>

// ❌ Nested KAVs (one in parent, another in child)
<KeyboardAvoidingView>
  <SomeChildScreen>
    <KeyboardAvoidingView>...</KeyboardAvoidingView>
  </SomeChildScreen>
</KeyboardAvoidingView>

// ❌ Pressable with onPress around ScrollView (breaks scroll)
<Pressable onPress={Keyboard.dismiss}>
  <ScrollView>...</ScrollView>
</Pressable>

// ❌ contentContainerStyle with flex: 1 (prevents scrolling)
<ScrollView contentContainerStyle={{ flex: 1 }}>

// ❌ height: '100%' inside scrollable content
<View style={{ height: '100%' }}>

// ❌ Duplicated SafeAreaView edges with an internal header that already handles top
<SafeAreaView>  {/* default = top/right/bottom/left */}
  <CustomHeader />  {/* already has edges=["top"] internally */}
</SafeAreaView>

// ❌ Drawer.Screen / Stack.Screen with headerShown: true when the screen has inputs
// (causes the button to get cut off by the keyboard)
<Drawer.Screen options={{ headerShown: true, title: "..." }} />
```

---

## Checklist for Creating or Refactoring a Keyboard Screen

- [ ] `SafeAreaView` comes from `react-native-safe-area-context`.
- [ ] Hierarchy: `SafeAreaView > KAV > ScrollView/FlatList > content`.
- [ ] `behavior={Platform.OS === "ios" ? "padding" : undefined}`. NEVER `"height"`.
- [ ] There is no `keyboardVerticalOffset` with a fixed numeric value.
- [ ] If there is a header: use a custom header INSIDE the `SafeAreaView`, with `headerShown: false` on the navigator.
- [ ] `ScrollView` has `style={{flex:1}}` + `contentContainerStyle={{flexGrow:1}}` + `keyboardShouldPersistTaps="handled"`.
- [ ] If keyboard dismissal on scroll is desired: `keyboardDismissMode="on-drag"`.
- [ ] No `Pressable`/`TouchableWithoutFeedback` wraps the `ScrollView`.
- [ ] There is only ONE `KeyboardAvoidingView` per screen, as high as possible.
- [ ] If the screen is a stepper: scroll and KAV live in the parent, not inside each step.
- [ ] SafeAreaView edges match the context (`["left","right","bottom"]` when an internal header handles `top`).
- [ ] Fixed-button footer: plain `<View>` with fixed `paddingHorizontal`, `paddingTop`, and `paddingBottom`. No internal `SafeAreaView`.
- [ ] Tested on an iPhone with a home indicator and on Android. The bottom button is not covered by the keyboard.

---

## Quick Reference: Footer Style

Standard pattern for a fixed bottom button:

```ts
footer: {
  paddingHorizontal: 16,
  paddingTop: 8,
  paddingBottom: 8,
}
```

If the parent `SafeAreaView` has `edges={["bottom"]}`, bottom padding is added to the inset (more space without keyboard, less with keyboard). If the parent does NOT include `bottom` in its edges, `paddingBottom` is constant, which is recommended to avoid Bug 2.
