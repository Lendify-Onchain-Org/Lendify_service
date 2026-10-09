# UI Context — Lendify-App

## Design Philosophy

Lendify-App targets learners and interns in emerging markets — many of whom are new to DeFi and crypto. The UI must be:

- **Simple over clever** — every screen is immediately understandable without a tutorial
- **Encouraging over technical** — plain language, no DeFi jargon without explanation
- **Mobile-first** — designed for Android and iOS, not web
- **Dark and focused** — deep navy backgrounds, blue-green accents, no distractions
- **Accessible** — high contrast ratios, large tap targets (min 44px), readable font sizes

---

## Theme

**Dark only. No light mode. No theme toggle.**

The visual language is a clean fintech dark — deep navy base, layered surfaces, blue-green brand accents, and warm green CTAs. Every color is a named token in `constants/colors.ts`. No hardcoded hex values anywhere in the codebase.

---

## Color Tokens (`constants/colors.ts`)

```typescript
export const colors = {
  // Backgrounds
  background:      '#080F1A',  // Page background — deepest layer
  surface:         '#0D1B2A',  // Card and panel background
  elevated:        '#132238',  // Elevated surface — modals, bottom sheets
  subtle:          '#1A2E45',  // Subtle fill — input backgrounds, inactive states

  // Borders
  border:          '#1E3A52',  // Default border
  borderSubtle:    '#2A4A66',  // Subtle border — dividers

  // Text
  textPrimary:     '#F0F4F8',  // Primary text — headings, values
  textSecondary:   '#A8BCCF',  // Secondary text — labels, subtitles
  textMuted:       '#5A7A94',  // Muted text — placeholders, captions
  textFaint:       '#3A5570',  // Faint text — disabled states

  // Brand
  brandBlue:       '#2563EB',  // Primary brand blue
  brandGreen:      '#22C55E',  // Primary brand green
  brandBlueDim:    'rgba(37, 99, 235, 0.15)',   // Blue tinted background
  brandGreenDim:   'rgba(34, 197, 94, 0.15)',   // Green tinted background

  // CTA
  cta:             '#22C55E',  // Primary action button fill
  ctaText:         '#080F1A',  // Text on CTA button (dark on green)

  // Semantic
  error:           '#EF4444',
  errorDim:        'rgba(239, 68, 68, 0.15)',
  warning:         '#F59E0B',
  warningDim:      'rgba(245, 158, 11, 0.15)',
  success:         '#22C55E',
  successDim:      'rgba(34, 197, 94, 0.15)',

  // Reputation tiers
  tier: {
    gold:    '#F59E0B',
    silver:  '#94A3B8',
    bronze:  '#D97706',
    starter: '#5A7A94',
  },
}
```

---

## Typography

No custom fonts in v1. System default — San Francisco on iOS, Roboto on Android.

| Role | Size | Weight | Token |
|---|---|---|---|
| Screen title | 24 | 700 (bold) | — |
| Section header | 18 | 600 (semibold) | — |
| Body | 16 | 400 (regular) | — |
| Secondary body | 14 | 400 (regular) | — |
| Caption / label | 12 | 400 (regular) | — |
| Large amount | 32 | 700 (bold) | — |
| Button label | 16 | 600 (semibold) | — |

NativeWind classes for typography:
```
text-2xl font-bold      → Screen title
text-lg font-semibold   → Section header
text-base               → Body
text-sm                 → Secondary body / labels
text-xs                 → Captions
text-4xl font-bold      → Large amount display
```

---

## Border Radius

| Context | NativeWind class | Value |
|---|---|---|
| Badges, chips, tags | `rounded-xl` | 12px |
| Cards, panels, inputs | `rounded-2xl` | 16px |
| Primary buttons | `rounded-2xl` | 16px |
| Bottom sheets (top) | `rounded-t-3xl` | 24px |
| Modals and overlays | `rounded-3xl` | 24px |
| Avatars | `rounded-full` | 50% |

---

## Icons (Lucide React Native)

**Lucide React Native is the only icon library in Lendify-App.** No other icon libraries. Stroke-based icons only — no filled variants.

### Import Pattern

```tsx
import { Wallet, CreditCard, Star, ChevronRight, AlertCircle, CheckCircle } from 'lucide-react-native'
```

### Size Scale

| Context | Size prop | When to use |
|---|---|---|
| Inline text icon | `size={16}` | Inside text, badges, small labels |
| Button icon | `size={20}` | Inside buttons, list row actions |
| Tab bar icon | `size={24}` | Bottom navigation tabs |
| Feature icon | `size={28}` | Section headers, prominent actions |
| Empty state icon | `size={32}` | Empty state illustrations |

### Color

Always pass `color` explicitly — never rely on default:

```tsx
<Wallet size={20} color={colors.textPrimary} />
<CheckCircle size={16} color={colors.success} />
<AlertCircle size={20} color={colors.error} />
```

### Common Icons Per Feature

| Feature | Icon |
|---|---|
| Wallet / Auth | `Wallet` |
| Loans | `CreditCard` |
| Reputation | `Star` |
| Vendors | `Store` |
| Installments | `Calendar` |
| Payment | `ArrowUpRight` |
| Notifications | `Bell` |
| Settings | `Settings` |
| Success | `CheckCircle` |
| Error / Warning | `AlertCircle` |
| Loading | Use `Loader.tsx` component |
| Navigate forward | `ChevronRight` |
| Navigate back | `ChevronLeft` |
| Sponsor / Invest | `TrendingUp` |
| Vouch | `UserCheck` |

---

## Component Patterns

### Primary Button

```tsx
import { colors } from '@/constants/colors'

<TouchableOpacity
  className="h-14 w-full rounded-2xl items-center justify-center"
  style={{ backgroundColor: colors.cta }}
  activeOpacity={0.8}
  onPress={handlePress}
>
  <Text className="text-base font-semibold" style={{ color: colors.ctaText }}>
    Apply for loan
  </Text>
</TouchableOpacity>
```

### Secondary Button (outline)

```tsx
<TouchableOpacity
  className="h-14 w-full rounded-2xl items-center justify-center border"
  style={{ borderColor: colors.border }}
  activeOpacity={0.8}
>
  <Text className="text-base font-semibold" style={{ color: colors.textPrimary }}>
    View history
  </Text>
</TouchableOpacity>
```

### Card

```tsx
<View
  className="rounded-2xl p-4"
  style={{
    backgroundColor: colors.surface,
    borderWidth: 1,
    borderColor: colors.border,
  }}
>
  {children}
</View>
```

### Reputation Score Card

```tsx
<View className="rounded-2xl p-5 gap-3" style={{ backgroundColor: colors.surface, borderWidth: 1, borderColor: colors.border }}>
  <Text className="text-sm" style={{ color: colors.textMuted }}>Your trust score</Text>
  <Text className="text-4xl font-bold" style={{ color: colors.textPrimary }}>{score}</Text>
  <View className="flex-row items-center gap-2">
    <View className="rounded-xl px-3 py-1" style={{ backgroundColor: tierColor + '20' }}>
      <Text className="text-sm font-semibold" style={{ color: tierColor }}>
        {tier}
      </Text>
    </View>
    <Text className="text-sm" style={{ color: colors.textMuted }}>
      Pay on time to increase your score
    </Text>
  </View>
</View>
```

### Installment Row

```tsx
<View className="flex-row items-center justify-between py-4" style={{ borderBottomWidth: 1, borderBottomColor: colors.border }}>
  <View className="flex-row items-center gap-3">
    <View
      className="h-8 w-8 rounded-full items-center justify-center"
      style={{ backgroundColor: isPaid ? colors.brandGreenDim : colors.subtle }}
    >
      <CheckCircle size={16} color={isPaid ? colors.brandGreen : colors.textMuted} />
    </View>
    <View>
      <Text className="text-sm font-medium" style={{ color: colors.textPrimary }}>
        {formatDate(dueDate)}
      </Text>
      <Text className="text-xs" style={{ color: colors.textMuted }}>
        {formatAmount(amount)}
      </Text>
    </View>
  </View>
  <Text
    className="text-sm font-medium"
    style={{ color: isPaid ? colors.success : isOverdue ? colors.error : colors.warning }}
  >
    {isPaid ? 'Paid' : isOverdue ? 'Overdue' : 'Due'}
  </Text>
</View>
```

### Empty State

```tsx
<View className="flex-1 items-center justify-center px-8 gap-4">
  <View
    className="h-16 w-16 rounded-2xl items-center justify-center"
    style={{ backgroundColor: colors.brandBlueDim }}
  >
    <CreditCard size={28} color={colors.brandBlue} />
  </View>
  <Text className="text-lg font-semibold text-center" style={{ color: colors.textPrimary }}>
    No loans yet
  </Text>
  <Text className="text-sm text-center" style={{ color: colors.textMuted }}>
    Apply for your first loan to get started
  </Text>
  <TouchableOpacity
    className="h-12 px-6 rounded-2xl items-center justify-center"
    style={{ backgroundColor: colors.cta }}
  >
    <Text className="text-sm font-semibold" style={{ color: colors.ctaText }}>
      Apply now
    </Text>
  </TouchableOpacity>
</View>
```

### Input Field

```tsx
<View className="gap-1">
  <Text className="text-sm" style={{ color: colors.textSecondary }}>
    {label}
  </Text>
  <TextInput
    className="h-12 rounded-xl px-4 text-base"
    style={{
      backgroundColor: colors.subtle,
      borderWidth: 1,
      borderColor: colors.border,
      color: colors.textPrimary,
    }}
    placeholderTextColor={colors.textMuted}
    placeholder={placeholder}
    value={value}
    onChangeText={onChange}
  />
</View>
```

### Notification Badge

```tsx
<View
  className="rounded-full px-2 py-0.5 min-w-[20px] items-center"
  style={{ backgroundColor: colors.error }}
>
  <Text className="text-xs font-bold" style={{ color: '#fff' }}>
    {count > 9 ? '9+' : count}
  </Text>
</View>
```

---

## Screen Layout Patterns

### Standard Scrollable Screen

```tsx
export default function Screen() {
  return (
    <SafeAreaView className="flex-1" style={{ backgroundColor: colors.background }}>
      <ScrollView className="flex-1" contentContainerStyle={{ padding: 16, paddingBottom: 32 }}>
        <Text className="text-2xl font-bold mt-2 mb-6" style={{ color: colors.textPrimary }}>
          Screen Title
        </Text>
        {/* content */}
      </ScrollView>
    </SafeAreaView>
  )
}
```

### Bottom Sheet

```tsx
<View
  className="rounded-t-3xl p-6"
  style={{ backgroundColor: colors.elevated }}
>
  {/* drag handle */}
  <View
    className="w-10 h-1 rounded-full self-center mb-6"
    style={{ backgroundColor: colors.border }}
  />
  {/* content */}
</View>
```

### Transaction Confirmation Screen

Always shown before any blockchain action:

```tsx
<View className="flex-1 px-4 pt-6 gap-6" style={{ backgroundColor: colors.background }}>
  <Text className="text-xl font-bold" style={{ color: colors.textPrimary }}>
    Review your loan
  </Text>

  {/* Summary card */}
  <View className="rounded-2xl p-4 gap-3" style={{ backgroundColor: colors.surface, borderWidth: 1, borderColor: colors.border }}>
    <SummaryRow label="Amount" value="$500" />
    <SummaryRow label="Installments" value="6 months" />
    <SummaryRow label="Interest rate" value="8% (Bronze tier)" />
    <SummaryRow label="Vendor" value="ALX Africa" />
  </View>

  {/* Warning */}
  <View className="rounded-xl p-3 flex-row gap-2" style={{ backgroundColor: colors.warningDim }}>
    <AlertCircle size={16} color={colors.warning} />
    <Text className="text-sm flex-1" style={{ color: colors.warning }}>
      This action will be recorded on the Stellar blockchain and cannot be reversed.
    </Text>
  </View>

  {/* Actions */}
  <View className="gap-3 mt-auto">
    <PrimaryButton label="Sign with wallet" onPress={handleSign} />
    <SecondaryButton label="Cancel" onPress={handleCancel} />
  </View>
</View>
```

---

## Learner-First UX Rules

These rules are non-negotiable for the learner audience:

1. **Tooltips on financial terms** — "Reputation score", "APY", "Credit limit", "Installment" must all have a tappable info icon with a plain-English explanation.

2. **No raw DeFi jargon** — Never show "XDR", "Soroban", "Horizon", "Ledger" to users. Show "Signing your loan agreement", "Sending payment", etc.

3. **Transaction confirmation always** — Never sign or submit a transaction without showing the user a clear summary of what they're signing.

4. **Friendly payment reminders** — "Your next payment of $50 is due in 3 days 📅" not "Payment overdue: action required."

5. **Wallet onboarding help** — The sign-in screen must have a "Don't have a Stellar wallet?" link that opens a step-by-step guide.

6. **Amounts in context** — Always show payment amounts with the installment number: "Payment 3 of 6 — $50".

7. **Score context** — Never show just the score number. Always show tier and what it means: "72/100 · Bronze · 8% interest rate".

8. **Error messages for humans** — Never show raw API error codes. Map all errors to user-friendly messages in the service layer.
