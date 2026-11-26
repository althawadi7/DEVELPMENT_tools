# Apple Liquid Glass Navbar — Flutter Guide

Exact implementation used for the floating Apple-style Liquid Glass navigation bar (real border refraction + diagonal white rim + sliding pill).

---

## 1. Add Package (`pubspec.yaml`)

```yaml
dependencies:
  liquid_glass_renderer: ^0.2.0-dev.4
```

Run:
```bash
flutter pub get
```

> **Note:** Real shader refraction requires **Impeller** (enabled by default on iOS and modern Android). On older devices/tests, pass `fake: true` to fall back cleanly.

---

## 2. Create `liquid_glass_surface.dart`

Save this self-contained file anywhere in `lib/widgets/liquid_glass_surface.dart`:

```dart
import 'dart:math' as math;

import 'package:flutter/material.dart';
import 'package:liquid_glass_renderer/liquid_glass_renderer.dart';

/// Descriptor for a single tab in [GlassNavRow].
class GlassNavTab {
  const GlassNavTab({
    required this.label,
    this.icon,
    this.activeIcon,
    this.iconBuilder,
  }) : assert(icon != null || iconBuilder != null);

  final String label;
  final IconData? icon;
  final IconData? activeIcon;
  final Widget Function(BuildContext context, bool isActive, Color color)?
      iconBuilder;
}

/// Apple-style Liquid Glass surface with border-only light bending & refraction.
class LiquidGlassSurface extends StatelessWidget {
  const LiquidGlassSurface({
    super.key,
    required this.child,
    this.height,
    this.borderRadius = 28,
    this.tint,
    this.blur,
    this.settings,
    this.shape,
    this.fake = false,
  });

  final Widget child;
  final double? height;
  final double borderRadius;
  final Color? tint;
  final double? blur;
  final LiquidGlassSettings? settings;
  final LiquidShape? shape;
  final bool fake;

  @override
  Widget build(BuildContext context) {
    final isDark = Theme.of(context).brightness == Brightness.dark;

    // `thickness: 14` keeps refraction focused strictly along the rounded borders,
    // while `refractiveIndex: 1.80` bends & stretches light passing behind the rim.
    final glass = settings ??
        LiquidGlassSettings(
          thickness: 14,
          blur: blur ?? 0.8,
          saturation: 1.75,
          refractiveIndex: 1.80,
          chromaticAberration: .10,
          lightAngle: -math.pi / 4,
          lightIntensity: isDark ? 1.35 : 1.45,
          ambientStrength: isDark ? .20 : .32,
          glassColor:
              tint ?? Colors.white.withValues(alpha: isDark ? .04 : .14),
        );

    final effectiveShape =
        shape ?? LiquidRoundedSuperellipse(borderRadius: borderRadius);
    final rimRadius = BorderRadius.circular(borderRadius);
    final outerBorder = RoundedSuperellipseBorder(borderRadius: rimRadius);

    return DecoratedBox(
      decoration: ShapeDecoration(
        shape: outerBorder,
        shadows: [
          BoxShadow(
            color: Colors.black.withValues(alpha: isDark ? 0.42 : 0.14),
            blurRadius: 24,
            offset: const Offset(0, 8),
          ),
        ],
      ),
      child: LiquidGlassLayer(
        settings: glass,
        fake: fake,
        child: LiquidGlass.grouped(
          shape: effectiveShape,
          child: SizedBox(
            height: height,
            width: double.infinity,
            child: Stack(
              fit: StackFit.passthrough,
              children: [
                child,
                // 1.3px diagonal white gradient rim along the outer border
                Positioned.fill(
                  child: IgnorePointer(
                    child: CustomPaint(
                      painter: _GlassRimPainter(
                        shape: outerBorder,
                        isDark: isDark,
                      ),
                    ),
                  ),
                ),
              ],
            ),
          ),
        ),
      ),
    );
  }
}

class _GlassRimPainter extends CustomPainter {
  const _GlassRimPainter({
    required this.shape,
    required this.isDark,
  });

  final ShapeBorder shape;
  final bool isDark;

  @override
  void paint(Canvas canvas, Size size) {
    final rect = Offset.zero & size;
    final path = shape.getOuterPath(rect.deflate(0.65));
    final paint = Paint()
      ..style = PaintingStyle.stroke
      ..strokeWidth = 1.3
      ..shader = LinearGradient(
        begin: Alignment.topLeft,
        end: Alignment.bottomRight,
        stops: const [0.0, 0.45, 1.0],
        colors: isDark
            ? [
                Colors.white.withValues(alpha: 0.62),
                Colors.white.withValues(alpha: 0.14),
                Colors.white.withValues(alpha: 0.36),
              ]
            : [
                Colors.white.withValues(alpha: 0.92),
                Colors.white.withValues(alpha: 0.30),
                Colors.white.withValues(alpha: 0.68),
              ],
      ).createShader(rect);

    canvas.drawPath(path, paint);
  }

  @override
  bool shouldRepaint(covariant _GlassRimPainter oldDelegate) =>
      oldDelegate.isDark != isDark || oldDelegate.shape != shape;
}

/// Circular Apple-style Liquid Glass icon button (for back/share/favorite buttons).
class LiquidGlassIconButton extends StatelessWidget {
  const LiquidGlassIconButton({
    super.key,
    required this.child,
    this.onTap,
    this.size = 42,
  });

  final Widget child;
  final VoidCallback? onTap;
  final double size;

  @override
  Widget build(BuildContext context) {
    final isDark = Theme.of(context).brightness == Brightness.dark;
    final glass = LiquidGlassSettings(
      thickness: 12,
      blur: 0.8,
      saturation: 1.75,
      refractiveIndex: 1.75,
      chromaticAberration: .10,
      lightAngle: -math.pi / 4,
      lightIntensity: isDark ? 1.35 : 1.45,
      ambientStrength: isDark ? .20 : .32,
      glassColor: Colors.white.withValues(alpha: isDark ? .06 : .18),
    );

    const circleShape = CircleBorder();

    final button = DecoratedBox(
      decoration: ShapeDecoration(
        shape: circleShape,
        shadows: [
          BoxShadow(
            color: Colors.black.withValues(alpha: isDark ? 0.36 : 0.14),
            blurRadius: 14,
            offset: const Offset(0, 4),
          ),
        ],
      ),
      child: LiquidGlassLayer(
        settings: glass,
        child: LiquidGlass.grouped(
          shape: const LiquidOval(),
          child: SizedBox(
            width: size,
            height: size,
            child: Stack(
              fit: StackFit.expand,
              children: [
                Center(child: child),
                IgnorePointer(
                  child: CustomPaint(
                    painter: _GlassRimPainter(
                      shape: circleShape,
                      isDark: isDark,
                    ),
                  ),
                ),
              ],
            ),
          ),
        ),
      ),
    );

    if (onTap == null) return button;
    return GestureDetector(
      behavior: HitTestBehavior.opaque,
      onTap: onTap,
      child: button,
    );
  }
}

/// Animated sliding-pill navigation row designed to sit inside [LiquidGlassSurface].
class GlassNavRow extends StatelessWidget {
  const GlassNavRow({
    super.key,
    required this.tabs,
    required this.selectedIndex,
    required this.onSelected,
    this.accentColor,
    this.duration = const Duration(milliseconds: 320),
  });

  final List<GlassNavTab> tabs;
  final int selectedIndex;
  final ValueChanged<int> onSelected;
  final Color? accentColor;
  final Duration duration;

  @override
  Widget build(BuildContext context) {
    final isDark = Theme.of(context).brightness == Brightness.dark;
    final accent = accentColor ?? Theme.of(context).colorScheme.primary;
    final activeColor =
        ThemeData.estimateBrightnessForColor(accent) == Brightness.dark
            ? Colors.white
            : Colors.black87;
    final inactiveColor = isDark
        ? Colors.white.withValues(alpha: 0.88)
        : const Color(0xFF141414).withValues(alpha: 0.85);

    return LayoutBuilder(
      builder: (context, constraints) {
        final count = tabs.length;
        if (count == 0) return const SizedBox.shrink();

        final slotWidth = constraints.maxWidth / count;
        final clampedIndex = selectedIndex.clamp(0, count - 1);
        final pillStart = clampedIndex * slotWidth;

        return Stack(
          alignment: Alignment.center,
          children: [
            AnimatedPositionedDirectional(
              duration: duration,
              curve: Curves.easeOutCubic,
              start: pillStart + 6,
              width: (slotWidth - 12).clamp(40.0, constraints.maxWidth),
              height: 48,
              child: DecoratedBox(
                decoration: BoxDecoration(
                  color: accent,
                  borderRadius: BorderRadius.circular(100),
                ),
              ),
            ),
            Row(
              mainAxisAlignment: MainAxisAlignment.center,
              crossAxisAlignment: CrossAxisAlignment.center,
              children: List.generate(count, (index) {
                final isActive = index == clampedIndex;
                final tab = tabs[index];
                final color = isActive ? activeColor : inactiveColor;

                return SizedBox(
                  width: slotWidth,
                  child: GestureDetector(
                    behavior: HitTestBehavior.opaque,
                    onTap: () => onSelected(index),
                    child: Center(
                      child: AnimatedScale(
                        scale: isActive ? 1.08 : 1.0,
                        duration: const Duration(milliseconds: 240),
                        curve: Curves.easeOutCubic,
                        child: tab.iconBuilder != null
                            ? tab.iconBuilder!(context, isActive, color)
                            : Icon(
                                isActive
                                    ? (tab.activeIcon ?? tab.icon)
                                    : tab.icon,
                                size: 27,
                                color: color,
                              ),
                      ),
                    ),
                  ),
                );
              }),
            ),
          ],
        );
      },
    );
  }
}
```

---

## 3. Usage in Your Main Screen

Place the navbar **inside a `Stack` over your page body** (not `Scaffold.bottomNavigationBar`!) so the page content scrolls underneath the glass and refracts through it:

```dart
class MainShell extends StatefulWidget {
  const MainShell({super.key});

  @override
  State<MainShell> createState() => _MainShellState();
}

class _MainShellState extends State<MainShell> {
  int _currentIndex = 0;

  final _pages = const [
    // Your tab pages (add bottom padding ~96 so content scrolls cleanly past the bar)
  ];

  @override
  Widget build(BuildContext context) {
    final bottomInset = MediaQuery.of(context).padding.bottom;

    return Scaffold(
      body: Stack(
        children: [
          Positioned.fill(
            child: IndexedStack(
              index: _currentIndex,
              children: _pages,
            ),
          ),
          Positioned(
            left: 16,
            right: 16,
            bottom: bottomInset > 0 ? bottomInset : 16,
            child: LiquidGlassSurface(
              height: 64,
              child: GlassNavRow(
                selectedIndex: _currentIndex,
                accentColor: const Color(0xFFA86A42), // Your brand color
                onSelected: (i) {
                  HapticFeedback.lightImpact();
                  setState(() => _currentIndex = i);
                },
                tabs: const [
                  GlassNavTab(
                    label: 'Home',
                    icon: Icons.home_outlined,
                    activeIcon: Icons.home_rounded,
                  ),
                  GlassNavTab(
                    label: 'Explore',
                    icon: Icons.dashboard_customize_outlined,
                    activeIcon: Icons.dashboard_customize_rounded,
                  ),
                  GlassNavTab(
                    label: 'Profile',
                    icon: Icons.person_outline,
                    activeIcon: Icons.person_rounded,
                  ),
                ],
              ),
            ),
          ),
        ],
      ),
    );
  }
}
```
