---
skill: android-engineering
section: design-ux
---

# Design & UX

## The Affordance Rule (most important)

**If it is tappable, it must look tappable. If it looks tappable, it must be tappable.**

This is the single most common Android UX failure. Static text that navigates somewhere,
cards with no visual tap cue, list items with no ripple — users will not know these are
interactive. The subscription screen example: a card that looked like a label but required
a tap to reveal detail. No ripple, no chevron, no underline = user cannot tell.

**Every interactive element must provide at least one of:**
- Ripple effect on touch (automatic with `Surface(onClick=...)`, `Button`, `TextButton`, etc.)
- Underline (links in text)
- Chevron / arrow icon (items that navigate deeper)
- Elevation change on press
- Color that is distinct from body text (primary color for primary actions)

**What goes wrong:**
```kotlin
// Wrong — no affordance cue, no ripple
Box {
    Text("View subscription details")
}

// Correct — ripple, semantic role
Surface(
    onClick = { onViewSubscription() },
    modifier = Modifier.semantics { role = Role.Button }
) {
    Row {
        Text("View subscription details")
        Icon(Icons.Default.ChevronRight, contentDescription = null)
    }
}
```

---

## Material Design 3 (current standard)

Material Design 3 Expressive (released Google I/O May 2025) is the current Android design
language. Use the `androidx.compose.material3` library.

**Core principles:**
- Color signals priority — primary color = primary action. Never two primary-colored
  buttons on the same screen. One CTA per screen.
- Shape communicates intent — rounded = approachable, sharp = structured. Be consistent
  within a screen. M3 Expressive uses spring-based shape morphing on interaction.
- Size signals hierarchy — larger = more important. Don't make secondary actions visually
  compete with primary actions.

**Button hierarchy (use the right type):**

| Type | When | Compose |
|---|---|---|
| Filled `Button` | Primary action (one per screen) | `Button(onClick=...)` |
| `FilledTonalButton` | Secondary, important | `FilledTonalButton(onClick=...)` |
| `OutlinedButton` | Secondary, neutral | `OutlinedButton(onClick=...)` |
| `TextButton` | Tertiary, low emphasis | `TextButton(onClick=...)` |
| `IconButton` | Toolbar / icon-only | `IconButton(onClick=...)` |

Never use a `TextButton` for a destructive action — use `FilledTonalButton` or `Button`
with error color so users cannot accidentally miss the consequence.

**Cards:**
- Use `Card(onClick=...)` for tappable cards — provides ripple automatically
- Use `ElevatedCard` for content that floats above the surface
- Never use a non-clickable `Card` layout wrapping invisible click handler

**Lists:**
- `ListItem` Composable handles icon, headline, supporting text, and trailing content correctly
- Do not roll your own list item from `Row` unless `ListItem` cannot accommodate the design

---

## Touch Targets

Minimum touch target: **48dp × 48dp**, even if the visual is smaller.

```kotlin
// Icon button that is visually 24dp but has 48dp touch target
IconButton(
    onClick = { onDelete() },
    modifier = Modifier.size(48.dp)  // touch target
) {
    Icon(
        Icons.Default.Delete,
        contentDescription = "Delete message",
        modifier = Modifier.size(24.dp)  // visual size
    )
}
```

48dp is the WCAG minimum and Android accessibility guideline minimum.
Never make interactive elements smaller than 48dp touch target.

---

## Navigation Patterns

**Bottom navigation:** 3–5 top-level destinations. Visible always. Use `NavigationBar`
with `NavigationBarItem`.

**Don't use a hamburger menu (NavigationDrawer) for primary navigation.** Drawers hide
destination options from users. Acceptable for settings or secondary content only.

**Back stack:** Compose Navigation manages back stack. Do not manage your own. Do not
override back behavior unless there is a deliberate UX reason (e.g., confirm before
discard on a form).

**Single Activity:** One `MainActivity`, all screens are Composables. No Fragment
navigation, no multiple Activities for normal flows.

**Typing in navigation arguments:** Pass IDs, not objects. Objects are not Parcelable
across navigation boundaries safely.

```kotlin
// Correct — pass ID
navController.navigate("chat/$jid")

// Wrong — passing a serialized object through nav args is fragile
navController.navigate("chat/${Json.encode(chatObject)}")
```

---

## Accessibility

**Content descriptions:**
```kotlin
// Image that conveys meaning — needs description
Image(
    painter = painterResource(R.drawable.contact_avatar),
    contentDescription = "Contact avatar for $contactName"  // describes meaning
)

// Purely decorative — explicitly null
Icon(
    imageVector = Icons.Default.ChevronRight,
    contentDescription = null  // decorative — screen reader skips it
)
```

**Interactive element labels — describe the action, not the visual:**
```kotlin
IconButton(
    onClick = { onScheduleMessage() },
    modifier = Modifier.semantics {
        contentDescription = "Schedule message"  // what it does, not "clock icon"
    }
) {
    Icon(Icons.Default.Schedule, contentDescription = null)  // null here — parent has it
}
```

**State descriptions for toggles:**
```kotlin
Switch(
    checked = isEnabled,
    onCheckedChange = { onToggle(it) },
    modifier = Modifier.semantics {
        stateDescription = if (isEnabled) "Auto-reply enabled" else "Auto-reply disabled"
    }
)
```

**Rules:**
- Do not rely on color alone to communicate state — colorblind users cannot distinguish
- Use built-in composables (`Button`, `Switch`, `Checkbox`) — they include correct semantics
- Test with TalkBack before shipping any screen
- Minimum text contrast ratio: 4.5:1 for normal text, 3:1 for large text

---

## Typography & Spacing

**Text sizes:**
- Use `MaterialTheme.typography.*` — never hardcode `sp` values
- `displayLarge` / `headlineLarge` for screen titles
- `bodyLarge` / `bodyMedium` for content
- `labelSmall` for captions, timestamps, secondary info

**Spacing:** Use multiples of 4dp. Standard gutters: 16dp. Standard item padding: 16dp
horizontal, 12dp vertical. Never use pixel values.

```kotlin
// Correct
Modifier.padding(horizontal = 16.dp, vertical = 12.dp)

// Wrong
Modifier.padding(15.dp)  // not a multiple of 4, not a standard value
```

---

## Empty States and Loading

Every screen that loads data must handle three states: loading, content, empty.

```kotlin
when {
    isLoading -> CircularProgressIndicator()
    chats.isEmpty() -> EmptyState(
        icon = Icons.Outlined.Chat,
        message = stringResource(R.string.no_chats_message)
    )
    else -> ChatList(chats)
}
```

Empty states must:
- Explain why it's empty (not just a blank screen)
- Offer the primary action that fills it, if applicable ("Pair your WhatsApp account")

---

## Error Display

**Transient errors** (network blip, retry-able): Snackbar — short, dismissible, with retry action if applicable.

**Persistent errors** (config problem, auth failure): Inline error in the screen with clear
next step — not a dialog that blocks the whole UI.

**Never use dialogs for errors that don't require immediate user decision.**

```kotlin
// Snackbar for transient error
LaunchedEffect(error) {
    error?.let {
        snackbarHostState.showSnackbar(
            message = it.message,
            actionLabel = "Retry",
            duration = SnackbarDuration.Long
        )
    }
}
```
