# 05 — Performance Guidelines

> الهدف: **60 FPS** — كل إطار في أقل من **16.66ms**

---

## القياس والاختبار

| ❌ لا تفعل | ✅ افعل |
|---|---|
| قياس في Debug Mode | **Release/Profile Mode** دائماً |
| اختبار على iPhone أو Simulator | **Android منخفض المواصفات** |
| تجاهل DevTools | **Flutter DevTools** (CPU + Raster Thread) |

```bash
flutter run --release --flavor prod -t lib/main_prod.dart
```

---

## 1. Widgets vs Helper Methods ⚠️ مهم جداً

```dart
// ❌ خطأ — _buildCard() تُعيد بناء الـ parent كله لما أي state يتغير
class MyScreen extends StatelessWidget {
  Widget _buildCard() => Card(child: Text('Hello'));

  @override
  Widget build(BuildContext context) {
    return Column(children: [_buildCard()]);
  }
}

// ✅ صح — MyCard يتبنى لوحده فقط لما هو يحتاج
class MyCard extends StatelessWidget {
  const MyCard({super.key});

  @override
  Widget build(BuildContext context) => Card(child: const Text('Hello'));
}

class MyScreen extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return const Column(children: [MyCard()]); // const ← لا يتبنى أبداً
  }
}
```

---

## 2. const Constructors — استخدمها في كل مكان

```dart
// ✅ كل ده يتحسن بـ const
const Text('Hello')
const SizedBox(height: 16)
const Icon(Icons.star)
const EdgeInsets.all(16)
const Padding(padding: EdgeInsets.all(8), child: Text('Hi'))
const MyWidget()
```

---

## 3. إدارة الحالة — احصر setState في أضيق نطاق

```dart
// ❌ خطأ — setState على الشاشة كاملة لعنصر صغير
setState(() => _showScrollButton = true);

// ✅ صح — widget مستقلة تتولى حالتها وحدها
class ScrollToTopButton extends StatefulWidget {
  const ScrollToTopButton({super.key, required this.controller});
  final ScrollController controller;

  @override
  State<ScrollToTopButton> createState() => _ScrollToTopButtonState();
}

class _ScrollToTopButtonState extends State<ScrollToTopButton> {
  bool _visible = false;

  @override
  void initState() {
    super.initState();
    widget.controller.addListener(() {
      final shouldShow = widget.controller.offset > 200;
      if (shouldShow != _visible) setState(() => _visible = shouldShow);
    });
  }

  @override
  Widget build(BuildContext context) =>
      AnimatedOpacity(opacity: _visible ? 1 : 0, duration: 300.ms, child: ...);
}
```

---

## 4. ListView.builder — مش SingleChildScrollView

```dart
// ❌ خطأ — يبني 50 عنصر دفعة واحدة في الذاكرة
SingleChildScrollView(
  child: Column(children: items.map((e) => ItemCard(e)).toList()),
)

// ✅ صح — يبني عنصر فقط لما يظهر على الشاشة
ListView.builder(
  itemCount: items.length,
  itemBuilder: (context, index) => ItemCard(item: items[index]),
)
```

---

## 5. RepaintBoundary — عزل الـ Widgets اللي بتتحدث كتير

```dart
// ✅ مناسب لـ: animations, charts, timers, real-time data
RepaintBoundary(
  child: AnimatedProgressBar(value: progress),
)

// ✅ في القوائم الثقيلة
ListView.builder(
  itemBuilder: (context, index) => RepaintBoundary(
    child: HeavyListItem(item: items[index]),
  ),
)

// ⚠️ لا تستخدمه بإفراط — بيزيد استهلاك الذاكرة والـ GPU uploads
// استخدمه بس لما DevTools تثبت إن في repaint problem
```

---

## 6. لا تعطل الـ UI Thread

```dart
// ❌ خطأ — يجمد الـ UI
final result = heavyJsonParsing(largeData);

// ✅ صح — Isolate منفصل
final result = await compute(heavyJsonParsing, largeData);

// ✅ أو الطريقة الأحدث
final result = await Isolate.run(() => heavyJsonParsing(largeData));
```

---

## 7. تحسين الصور

```dart
// ✅ تحميل بالحجم المناسب للـ Container
final pixelRatio = MediaQuery.of(context).devicePixelRatio;
final imageWidth = (containerWidth * pixelRatio).toInt();

CachedNetworkImage(
  imageUrl: '$apiUrl/image?width=$imageWidth',
  memCacheWidth: imageWidth,
)

// ✅ cacheWidth لتقليل استهلاك الذاكرة
Image.network(url, cacheWidth: 300, cacheHeight: 300)
```

---

## 8. FadeTransition بدل Opacity في الـ Animations

```dart
// ❌ Opacity يستخدم saveLayer() — مكلف
Opacity(opacity: animation.value, child: widget)

// ✅ FadeTransition أسرع بكتير
FadeTransition(opacity: animation, child: widget)
```

---

## 9. ShaderMask — اجمع العناصر

```dart
// ❌ ShaderMask على كل عنصر منفصل (Skeleton Loader مثلاً)
Column(children: items.map((e) => ShaderMask(child: e)).toList())

// ✅ ShaderMask واحد على الكل
ShaderMask(
  shaderCallback: (bounds) => shimmerGradient.createShader(bounds),
  child: Column(children: skeletonItems),
)
```

---

## 10. AnimatedBuilder — child يُبنى مرة واحدة

```dart
// ❌ HeavyStaticWidget يتبنى مع كل frame
AnimatedBuilder(
  animation: _controller,
  builder: (context, _) => Column(
    children: [HeavyStaticWidget(), AnimatedPart(...)],
  ),
)

// ✅ HeavyStaticWidget يُبنى مرة واحدة فقط
AnimatedBuilder(
  animation: _controller,
  child: const HeavyStaticWidget(),  // ← مرة واحدة
  builder: (context, child) => Column(
    children: [child!, AnimatedPart(value: _controller.value)],
  ),
)
```

---

## 11. Lazy Loading للـ Tabs

```dart
class LazyTab extends StatefulWidget {
  const LazyTab({super.key, required this.child});
  final Widget child;

  @override
  State<LazyTab> createState() => _LazyTabState();
}

class _LazyTabState extends State<LazyTab> with AutomaticKeepAliveClientMixin {
  bool _initialized = false;

  @override
  void didChangeDependencies() {
    super.didChangeDependencies();
    if (!_initialized) {
      _initialized = true;
      // Load data here
    }
  }

  @override
  bool get wantKeepAlive => true;

  @override
  Widget build(BuildContext context) {
    super.build(context);
    return widget.child;
  }
}
```

---

## 12. StringBuffer في الـ Loops

```dart
// ❌ ينشئ String object جديد في كل iteration
String result = '';
for (final item in items) { result += '${item.name}, '; }

// ✅ يجمع كل حاجة ويعمل toString() مرة واحدة
final buffer = StringBuffer();
for (final item in items) { buffer.write('${item.name}, '); }
final result = buffer.toString();
```

---

## Performance Checklist قبل كل PR

- [ ] كل الـ Widgets الثابتة عندها `const`
- [ ] مفيش helper methods — كلها Widgets منفصلة
- [ ] القوائم بتستخدم `ListView.builder`
- [ ] الـ setState محدودة في Stateful Widgets صغيرة
- [ ] العمليات الثقيلة بتستخدم `compute()` أو `Isolate.run()`
- [ ] الصور بتتحمل بالحجم المناسب
- [ ] الـ Animations بتستخدم `FadeTransition` مش `Opacity`
- [ ] تم الاختبار على Release Mode وجهاز Android فعلي
